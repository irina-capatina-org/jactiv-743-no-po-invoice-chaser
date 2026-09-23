# PDD - No-PO Invoice Chaser

## Document History

| Date | Version | Author | Role | Comments |
|---|---|---|---|---|
| 2026-09-23 | 0.1 | uipath-analyst | Analyst | Initial PDD created from docs/jactiv-743-request-details.docx. |

| Date | Version | Author | Role | Comments |
|---|---|---|---|---|
| 2026-09-23 | 0.1 | uipath-analyst | Analyst | Initial analysis from request-work/request-details.md (No-PO invoice finder v1.0, UiPath Cartographer, 16 Sep 2026) |

## 1. Document Control

| Field | Value |
|---|---|
| Document | PDD - No-PO Invoice Chaser |
| Story key | JACTIV-743 |
| Epic key | [SME REVIEW] |
| Source file | request-work/request-details.md |
| Branch | analysis-jactiv-743 |
| Author | uipath-analyst |
| Status | Draft - pending SME approval |
| Version | 0.1 |

## 2. Introduction

**Process name:** No-PO Invoice Chaser
**Process Full Name:** `NoPoInvoiceChaser`

**Business objective:** Automatically identify Coupa invoices from the past seven days with no properly linked PO and notify the AP SME via Slack each weekday, eliminating manual filtering and ensuring consistent application of the no-PO-no-pay policy.

**Owning department:** Accounts Payable (Finance)

| Role | Name | Contact |
|---|---|---|
| SME / Process Owner | Irina Capatina | irina.capatina@uipath.com · Slack WLX9BD8FN |
| BA | uipath-analyst | — |
| Developer | [SME REVIEW] | — |

## 3. Process Overview

| Attribute | Value |
|---|---|
| Process full name | NoPoInvoiceChaser |
| Function and department | Accounts Payable, Finance |
| Short description | Weekday automated run queries Coupa for draft/new invoices in the past seven days with no properly linked PO, counts qualifying records, and sends one Slack Block Kit DM to the AP SME; sends nothing on a clean day |
| Required roles | Automation (unattended); AP SME receives notification only |
| Trigger and schedule | Weekday scheduler at 10:00 Romania time (EET/EEST) |
| Volume (items per day / peak) | ~194 qualifying invoices observed in live sample; normal daily volume [SME REVIEW] |
| Average handling time | Manual: daily ad-hoc, timing inconsistent; Automated: minutes per run [SME REVIEW] |
| FTE effort | [SME REVIEW] |
| Estimated exception rate | Low — data is structured and rules are deterministic |
| Input data | Coupa invoice list: invoice_date, status, invoice_type, PO linkage on invoice lines |
| Output data | Slack Block Kit DM to SME with invoice count and filtered Coupa URL; no output on clean day |

## 4. Scope

**In scope:**
- Weekday execution at 10:00 Romania time
- Coupa invoice query for invoices dated within the past seven days
- Filtering to status draft or new only
- Exclusion of credit notes
- Detection of missing or invalid PO linkage (including description-only PO numbers)
- Count of qualifying invoices
- One Slack Block Kit DM to SME with count, policy note, action request, and filtered Coupa link
- No message sent when qualifying count is zero

**Out of scope:**
- Purchase-order creation or modification
- Invoice approval or payment release
- Any Coupa record modification
- Supplier communication
- Requester direct notification or follow-up tracking
- Listing individual invoice numbers in the Slack message
- Audit-retention design
- Retry, fallback and error-recovery behaviour (per BR-08)

## 5. To-Be Process (High Level)

Fully unattended linear sequence: **read → filter → count → notify**. No human review in the normal path.

- Scheduler triggers at 10:00 Romania time on weekdays.
- Robot queries Coupa for invoices dated in the past seven days with status draft or new, excluding credit notes.
- Each invoice line is checked for a properly linked PO object; description-field PO text does not qualify.
- Qualifying invoices are counted.
- If count > 0: one Slack Block Kit DM is sent to the SME with the count, policy reminder, call to action, and filtered Coupa link.
- If count = 0: no message is sent.

Manual steps eliminated: manual Coupa filtering, field copying, grouping by requester, and manual Slack message composition.

## 6. Detailed Process Steps

| Step | Action | Application | Expected Result | Remarks |
|---|---|---|---|---|
| 1.0 | Scheduler fires weekday trigger at 10:00 Romania time | UiPath Orchestrator | Process instance starts | EET/EEST per Romanian DST; BR-04 |
| 1.1 | Calculate date window: window_end = today; window_start = today − 7 days | Automation | window_start and window_end available | Used in Coupa query params and Slack footer |
| 2.0 | Query Coupa for invoices where invoice_date BETWEEN window_start AND window_end AND status IN (draft, new) | Coupa | Paginated candidate invoice records returned | BR-04; API vs UI access [SME REVIEW] |
| 2.1 | For each invoice: if invoice_type = credit note → exclude; log exclusion | Automation | Credit notes removed from candidate list | BR-03 |
| 2.2 | For each remaining invoice: if no linked PO object exists on any invoice line → mark as qualifying (description-only PO text does not count) | Automation | Qualifying list built | BR-01, BR-02 |
| 2.3 | Count qualifying invoices → invoice_count | Automation | Integer invoice_count available | BR-05 |
| 3.0 | Decision: invoice_count = 0? | Automation | Branch to 3.1 (yes) or 4.0 (no) | BR-07 |
| 3.1 | Count = 0 — end run; no Slack message sent | Automation | Process completes cleanly | BR-07; see OQ-01 re: source exception-table conflict |
| 4.0 | Build coupa_url: base URL + invoice_date_gteq=window_start, invoice_date_lteq=window_end, status_eq=draft | Automation | coupa_url string ready | Production base URL [SME REVIEW]; test URL: uipath-test.coupahost.com |
| 4.1 | Compose Slack Block Kit JSON: title, three body lines, primary button (url=coupa_url), footer with window dates and run_date | Automation | Block Kit JSON payload ready with all placeholders substituted | Exact JSON in source section 4.3; count appears in title and body line 1 |
| 5.0 | POST Block Kit payload to Slack chat.postMessage; channel = WLX9BD8FN | Slack | Slack API confirms delivery | BR-06; member ID not email; one message only |
| 6.0 | Log success; mark run complete in Orchestrator | Orchestrator | Run marked success | Normal end |

## 7. Applications and Systems

| Application | Interface type | Access method | Login method | Credential handling | Comments |
|---|---|---|---|---|---|
| Coupa | API | Coupa REST API (read-only) | OAuth 2.0 / API key [SME REVIEW] | Orchestrator Asset [DEFAULT] | Production base URL [SME REVIEW]; test: uipath-test.coupahost.com |
| Slack | API | Slack Web API — HTTPS/REST (chat.postMessage) | Bot token | Orchestrator Asset [DEFAULT] | Recipient: member ID WLX9BD8FN; Block Kit payload |
| UiPath Orchestrator | Scheduler / runtime | Orchestrator web | Service account [DEFAULT] | Orchestrator-managed [DEFAULT] | Triggers weekday 10:00 Romania time job |

## 8. Business Rules

| ID | Rule | Source | Applies at step |
|---|---|---|---|
| BR-01 | Invoices without a properly linked purchase order are flagged under the no-PO-no-pay policy. | BR-001 | 2.2 |
| BR-02 | A PO number typed into the invoice description but not linked as a PO object does not satisfy the PO requirement. | BR-002 | 2.2 |
| BR-03 | Exclude credit notes from the qualifying population. | BR-003 | 2.1 |
| BR-04 | Include only invoices with status draft or new and invoice date within the past seven days. | BR-004 | 2.0 |
| BR-05 | Count qualifying invoices; the message reports that count. Individual invoices are not listed. | BR-005 | 2.3, 4.1 |
| BR-06 | Send the count to SME Irina Capatina (Slack member ID WLX9BD8FN) via Slack DM, with policy note, action request, and filtered Coupa link. | BR-006 | 5.0 |
| BR-07 | Send nothing when a successful query returns zero qualifying invoices. | BR-007 | 3.0–3.1 |
| BR-08 | No retry, fallback or recovery behaviour required; a run that cannot complete is reported as a failed run. | BR-008 | 6.0 |
| BR-09 | A failed run produces no notification; no second message path exists. | BR-009 | 6.0 |
| BR-10 | The automation does not create or modify POs, approve invoices, change Coupa records, or track requester completion. | BR-010 | All |

## 9. Business Exceptions

| ID | Name | Trigger step | Trigger condition | Action |
|---|---|---|---|---|
| B1 | Credit note excluded | 2.1 | invoice_type = credit note | Exclude from candidate list; log exclusion; continue |
| B2 | Description-only PO | 2.2 | PO number in description only, no linked PO object on any line | Treat as missing PO; include in qualifying list if other rules pass (BR-02) |
| B3 | Clean day | 3.0 | invoice_count = 0 after all filters | End run without sending any Slack message (BR-07); see OQ-01 |

## 10. System Errors

| ID | Name | Trigger condition | Severity | Retry policy | Action |
|---|---|---|---|---|---|
| S1 | Coupa API unavailable | HTTP 5xx or timeout at step 2.0 | High | None (BR-08) [DEFAULT] | Log error; mark run failed; no Slack notification (BR-09) |
| S2 | Coupa auth failure | HTTP 401/403 at step 2.0 | High | None [DEFAULT] | Log credential error; mark run failed; Orchestrator alert [DEFAULT] |
| S3 | Slack API unavailable | HTTP 5xx or timeout at step 5.0 | High | None (BR-08) [DEFAULT] | Log error; mark run failed; no retry (BR-09) |
| S4 | Slack auth failure | HTTP 401/403 at step 5.0 | High | None [DEFAULT] | Log credential error; mark run failed |
| S5 | Network timeout | Generic timeout on any HTTP call | Medium | None (BR-08) [DEFAULT] | Log and mark run failed |
| S6 | Unhandled exception | Unexpected runtime error at any step | High | None [DEFAULT] | Global handler logs stack trace; mark run failed |

## 11. Data Definitions

| Field | Type | Source | Target | Validation | Required |
|---|---|---|---|---|---|
| invoice_date | Date | Coupa invoice header | Filter | Within window_start..window_end inclusive | Yes |
| status | String (enum) | Coupa invoice header | Filter | Must be "draft" or "new" | Yes |
| invoice_type | String | Coupa invoice header | Filter | If = credit note → exclude (BR-03) | Yes |
| po_linkage | Object/reference | Coupa invoice line(s) | Filter | Properly linked PO object on at least one line; description text does not qualify (BR-02) | Yes |
| invoice_count | Integer | Computed | Slack message, Coupa URL | >= 0; 0 → clean-day path | Yes |
| window_start | Date | Computed (run date − 7 days) | Coupa query, Slack footer | Coupa-accepted date format [SME REVIEW] | Yes |
| window_end | Date | Computed (run date) | Coupa query, Slack footer | Coupa-accepted date format [SME REVIEW] | Yes |
| run_date | Date | System clock at trigger | Slack footer | ISO date | Yes |
| coupa_url | String (URL) | Computed | Slack Block Kit button | Parameterised with window_start, window_end, status_eq=draft; production base URL [SME REVIEW] | Yes |
| slack_member_id | String | Constant | Slack channel param | WLX9BD8FN | Yes |
| slack_payload | JSON | Computed (Block Kit) | Slack API body | Valid Block Kit JSON; placeholders substituted before send | Yes |
| requester | String | Coupa invoice header | Not used in to-be | N/A — traceability to as-is grouping step only | No |

**Coupa filtered URL pattern (source example):**
`https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D=<window_start>&q%5Binvoice_date_lteq%5D=<window_end>&q%5Bstatus_eq%5D=draft`

**Slack Block Kit message structure (source section 4.3):**

| Element | Content |
|---|---|
| Title | :receipt: {{invoice_count}} invoices need a purchase order |
| Body line 1 | :warning: {{invoice_count}} invoices from the last seven days have no purchase order linked. |
| Body line 2 | :no_entry: An invoice without a linked PO cannot be matched or paid under our no-PO-no-pay policy, and payment to the supplier stalls until it is fixed. |
| Body line 3 | :point_right: Please make sure a purchase order exists for these invoices and is correctly linked to each one. |
| Button | "Open the list in Coupa" — primary style — url = coupa_url |
| Footer | :calendar: Invoices dated {{window_start}} to {{window_end}} · :robot_face: No-PO Invoice Chaser · checked {{run_date}} |
| Recipient | DM to Slack member ID WLX9BD8FN (Irina Capatina) |

## 12. Assumptions, Dependencies and Open Questions

1. **OQ-01 - Clean-day message conflict.** `[SME REVIEW]` Section 7.1 exception table says send a congratulatory Slack message on a clean day; BR-007 and section 4.3 say send nothing — confirm which is required.
2. **OQ-02 - Coupa access method.** `[SME REVIEW]` Confirm REST API vs UI automation, production base URL, and auth credential type.
3. **OQ-03 - Status filter URL param.** `[SME REVIEW]` Source Coupa URL example uses status_eq=draft only; confirm whether the API supports filtering both draft and new in one call.
4. **OQ-04 - Future-state diagram shows retry logic.** `[SME REVIEW]` image2.png shows 3 retries for Coupa query and Slack send with an error-message path, contradicting BR-08/BR-09 — confirm authoritative behaviour.
5. **OQ-05 - Date format in message footer.** `[SME REVIEW]` Source next-steps item 3 flags date format as unconfirmed; confirm format for window_start/window_end in Slack footer.
6. **OQ-06 - Production Coupa base URL.** `[SME REVIEW]` Only test URL (uipath-test.coupahost.com) appears in source; supply production URL before build.
7. **Slack delivery method.** `[DEFAULT]` chat.postMessage via HTTPS REST with bot token stored as Orchestrator Asset.
8. **Coupa credentials storage.** `[DEFAULT]` API key or OAuth token stored as Orchestrator Asset; not hardcoded.
9. **Timezone handling.** `[DEFAULT]` Orchestrator job timezone set to Europe/Bucharest for the 10:00 Romania time trigger.
10. **No appendix attachment present.** Source Appendix A references embedded diagrams only; no separate file was provided. image2.png (future-state map) is readable; image1.png (as-is map) is readable.

## 13. Success Criteria

1. The process executes automatically on weekdays at 10:00 Romania time without manual intervention.
2. Only invoices with status draft or new and invoice date within the past seven days are evaluated.
3. Credit notes are excluded from the qualifying count.
4. An invoice with a PO number in the description field only is counted as missing a linked PO.
5. The Slack message reports the exact count of qualifying invoices and carries a working link to the filtered Coupa invoice list.
6. The Slack DM is delivered to Slack member ID WLX9BD8FN and to no other recipient.
7. When the qualifying count is zero, no Slack message is sent.
8. The automation does not create, modify or delete any record in Coupa.
