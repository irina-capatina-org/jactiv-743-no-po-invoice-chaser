# Build Notes — No-PO Invoice Chaser (JACTIV-743)

Single API Workflow project inside solution `no-po-invoice-chaser-743` that queries Coupa for draft/new invoices with no linked PO, counts qualifying records, and sends a Slack Block Kit DM to the AP SME.

## Task Status

| Task | Project | Status | Notes |
|---|---|---|---|
| T1 — Verify IS connections | platform | done | Connections confirmed present per arch §4; no Orchestrator call made (runner has no credentials); connection IDs wired verbatim from §4 |
| T2 — Build no-po-invoice-chaser-api | uipath-api-workflow | done | `uip api-workflow validate` passes; `uip solution pack` succeeds; `.uipx` produced |
| T3 — Testing | uipath-api-workflow | blocked | No Orchestrator credentials in this runner; live run tests (H1–H4, B1–B3, SE1–SE5) deferred to Test stage |
| T4 — Pack and publish | uipath-solution | partial | Pack proven (`uip solution pack` succeeded); publish/deploy deferred to Test/Deploy stages |

## Deviations from the SDD

None. The reference `Workflow.json` from §4.5 satisfies all SDD §4 steps and §6 error handling without modification. The SDD did not contradict the reference on any point.

The SDD mentions `Javascript_ComposeSlackPayload` and `Assign_SlackPayload`/`Assign_CoupaUrl` in the Then-branch (steps 7b). The reference includes these activities and they are present in the built workflow. The Slack `body` is inline per §4's binding rule (`validate-build.sh` confirmed `blocks` present inline in `bodyParameters.body`).

## Left for a Human

| Item | File | SDD Section | Notes |
|---|---|---|---|
| OQ-06: Production Coupa base URL | `Workflow.json` `Javascript_ComposeSlackPayload` code | SDD §1, §7 Env table | The `coupaUrl` built in the JS step uses `uipath-test.coupahost.com` (test default). Before PROD deploy, update the base URL in the script. |
| OQ-01: Clean-day message | Design confirmed as silent exit (BR-07). No action needed unless SME overrides. | SDD §4 step 7a | |
| OQ-05: Date format | Design uses `YYYY-MM-DD` (default). Confirm with SME before PROD. | SDD §1 | |
| Weekday scheduler trigger | Orchestrator | SDD §1 | Set 10:00 Europe/Bucharest weekday cron after deploy. Not configurable at pack time. |
| Both IS connections authorised in `Fusion2026` | Orchestrator IS | SDD §5 | Deployment prerequisite. Connections must be present and authorised before deploy run. |

## How to Test This

```bash
# 1. Static validation (offline)
uip api-workflow validate code/no-po-invoice-chaser-743/no-po-invoice-chaser-api/Workflow.json --output json

# 2. Build gate
bash .github/scripts/validate-build.sh code docs/architectural-considerations.md

# 3. Pack
uip solution pack code/no-po-invoice-chaser-743 /tmp/buildcheck \
  --name no-po-invoice-chaser-743 --version 0.0.1 --output json

# 4. Live run (requires uip login and IS connections authorised in Fusion2026)
uip api-workflow run code/no-po-invoice-chaser-743/no-po-invoice-chaser-api/Workflow.json \
  --output json
# Expected on a day with qualifying invoices: qualifying_count > 0, slack_sent = true
# Expected on a clean day: qualifying_count = 0, slack_sent = false
```

SDD §8 test cases H1–H4, B1–B3, SE1–SE5 require a live Orchestrator run against the test Coupa tenant.
