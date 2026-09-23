# PDD - No-PO Invoice Chaser

## Document History

| Date | Version | Author | Role | Comments |
|---|---|---|---|---|
| 2026-09-23 | 0.1 | uipath-analyst | Analyst | Initial PDD created from docs/jactiv-785-request-details.docx. |

| Date | Version | Author | Role | Comments |
|------|---------|--------|------|----------|
| 2026-09-23 | 0.1 | uipath-analyst | Analyst | Initial analysis from request-work/request-details.md (UiPath Cartographer, v1.0, 16 Sep 2026) |

## 1. Document Control

| Field | Value |
|-------|-------|
| Document | PDD - No-PO Invoice Chaser |
| Story key | JACTIV-785 |
| Epic key | [SME REVIEW] |
| Source file | request-work/request-details.md |
| Branch | analysis-jactiv-785 |
| Author | uipath-analyst |
| Status | Draft - pending SME approval |
| Version | 0.1 |

## 2. Introduction

**Process name:** No-PO Invoice Chaser
**Process Full Name:** `NoPoInvoiceChaser`

**Business objective:** Automate the daily Accounts Payable review that identifies invoices from the past seven days with no properly linked purchase order, and notify the SME via Slack so missing-PO exceptions are caught consistently rather than only when a supplier chases payment.

**Owning department:** Accounts Payable

| Role | Name | Contact |
|------|------|---------|
| SME / Process Owner | Irina Capatina | irina.capatina@uipath.com · Slack WLX9BD8FN |
| BA | uipath-analyst | — |
| Developer | [SME REVIEW] | — |

## 3. Process Overview

| Field | Value |
|-------|-------|
| Process full name | NoPoInvoiceChaser |
| Function and department | Accounts Payable – invoice compliance |
| Short description | Weekday automated run that queries Coupa for invoices with no linked PO and notifies the AP SME on Slack |
| Required roles | Automation (unattended); SME receives notification only |
| Trigger and schedule | Weekday schedule at 10:00 Romania time (Europe/Bucharest) |
| Volume (items per day / peak) | ~1 run per day; sample count 194 qualifying invoices [SME REVIEW for peak] |
| Average handling time | Manual: ~daily ad-hoc; Automated target: minutes per run |
| FTE effort | [SME REVIEW] |
| Estimated exception rate | Low – data is structured; credit-note exclusion is the main filter [SME REVIEW] |
| Input data | Coupa invoice list: invoice date, status, PO linkage, invoice type |
| Output data | Slack Block Kit direct message to SME with count and filtered Coupa URL; nothing on clean day |

## 4. To-Be Process (High Level)

The automation replaces the entire manual AP review. A weekday scheduler fires at 10:00 Romania time; the robot queries Coupa for invoices dated within the past seven days, retains only those with status draft or new, excludes credit notes, and counts those with no properly linked purchase order. If the count is greater than zero it sends one Slack Block Kit direct message to the SME containing the count, a policy reminder, and a link to the filtered Coupa list. If the count is zero it sends nothing.

**Manual steps that disappear:**
- AP member manually filters the Coupa invoice list
- AP member copies invoice fields and groups entries by requester
- AP member composes and sends the Slack message

**What stays human:** The SME receives the notification and takes action to ensure POs are raised and linked. Requester follow-up is out of scope.

## 5. Detailed Process Steps

| Step | Action | Application | Expected Result | Remarks |
|------|--------|-------------|-----------------|---------|
| 1.1 | Weekday scheduler fires at 10:00 Romania time (Europe/Bucharest) | Scheduler | Run is triggered | Runs Monday–Friday only; no manual trigger defined [SME REVIEW] |
| 1.2 | Calculate date window: window_end = today, window_start = today − 7 days | Automation | window_start and window_end variables set | Used in Coupa query and in the Slack message footer |
| 2.1 | Authenticate to Coupa using stored credentials | Coupa | Authenticated session / API token | Credential source [SME REVIEW]; see BR-10 |
| 2.2 | Query Coupa invoice list filtered to invoice_date >= window_start AND invoice_date <= window_end AND status IN (draft, new) | Coupa | Collection of candidate invoice records | Apply date and status filter in the query; see BR-04 |
| 2.3 | For each invoice record: check invoice type — **if** type = credit note → exclude record (log exclusion) | Coupa data | Credit notes removed from collection | BR-03; loop over returned records |
| 2.4 | For each remaining record: check PO linkage on invoice LINES — **if** no properly linked PO exists (PO present only in description field counts as missing) → mark as qualifying | Coupa data | Qualifying invoice set | BR-01, BR-02; PO linkage is on lines not header |
| 2.5 | Count qualifying invoices → invoice_count | Automation | invoice_count integer | BR-05 |
| 3.1 | **Decision:** invoice_count = 0 → go to step 3.2; invoice_count > 0 → go to step 3.3 | Automation | Branch selected | BR-07 |
| 3.2 | invoice_count = 0: end run without sending any message | Automation | Run completes silently | BR-07; clean day = no output |
| 3.3 | Build filtered Coupa URL: https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D={{window_start}}&q%5Binvoice_date_lteq%5D={{window_end}}&q%5Bstatus_eq%5D=draft | Automation | coupa_url string | URL template from source; status=draft only in URL per source example [SME REVIEW if new status also needed in URL] |
| 3.4 | Compose Slack Block Kit JSON payload with placeholders: invoice_count, coupa_url, window_start, window_end, run_date | Automation | Formatted Block Kit JSON body | Title + 3 body lines + primary button + footer; exact JSON layout to be defined in SDD from source template |
| 4.1 | Send Block Kit payload as Slack direct message to SME Slack member ID WLX9BD8FN | Slack | Message delivered to SME | BR-06; DM by member ID not email |
| 4.2 | Confirm HTTP 200 response from Slack API | Slack | Delivery confirmed | On non-200, treat as system error S3 |
| 4.3 | End run | Automation | Run completed successfully | |

## 6. Applications and Systems

| Application | Interface type | Access method | Login method | Credential handling | Comments |
|-------------|---------------|---------------|--------------|--------------------|---------| 
| Coupa | API | REST API (coupahost.com) | API key / OAuth [SME REVIEW] | UiPath Orchestrator credential asset [SME REVIEW] | Read-only; PO linkage on invoice lines not header |
| Slack | API | Slack HTTP Request activity (Incoming Webhook or Bot token) | Bot OAuth token [SME REVIEW] | UiPath Orchestrator credential asset [SME REVIEW] | DM to member ID WLX9BD8FN; Block Kit payload |
| UiPath Orchestrator | Scheduler / credential store | Orchestrator API | Robot service account | Managed by Orchestrator | Triggers weekday run and holds credentials |

## 7. Business Rules

| ID | Rule | Source | Applies at step |
|----|------|--------|-----------------|
| BR-01 | Apply the no-PO-no-pay policy: invoices without a properly linked PO qualify for notification. | BR-001 | 2.4 |
| BR-02 | A PO number typed into the invoice description but not linked on the invoice lines does not satisfy the PO requirement and is treated as missing. | BR-002 | 2.4 |
| BR-03 | Exclude credit notes from the qualifying population before counting. | BR-003 | 2.3 |
| BR-04 | Include only invoices with status draft or new and an invoice date within the past seven days (window_start to window_end inclusive). | BR-004 | 2.2 |
| BR-05 | Count the qualifying invoices; the message reports the count only — individual invoice details are not listed. | BR-005 | 2.5 |
| BR-06 | Send the count to SME Irina Capatina (Slack member ID WLX9BD8FN) by Slack direct message, including the policy note, action request, and filtered Coupa link. | BR-006 | 4.1 |
| BR-07 | Send nothing when a successful query returns zero qualifying invoices (clean day). | BR-007 | 3.1, 3.2 |
| BR-08 | No retry, fallback or recovery behaviour is required; a run that cannot complete is reported as a failed run. | BR-008 | 9 (all error steps) |
| BR-09 | A failed run produces no notification; no second message path exists. | BR-009 | 9 (all error steps) |
| BR-10 | The automation does not create or modify purchase orders, approve invoices, change Coupa records, or track requester completion. | BR-010 | All steps |

## 8. Business Exceptions

| ID | Name | Trigger step | Trigger condition | Action |
|----|------|-------------|-------------------|--------|
| B1 | Credit note excluded | 2.3 | Invoice type = credit note | Exclude record; log exclusion reason; continue processing |
| B2 | Description-only PO | 2.4 | PO reference present only in invoice description field, not linked on lines | Treat as missing PO; include in qualifying count if other rules pass |
| B3 | No qualifying invoices | 3.1 | invoice_count = 0 after all filters | End run silently; send no Slack message (BR-07) |

## 9. System Errors

| ID | Name | Trigger condition | Severity | Retry policy | Action |
|----|------|-------------------|----------|--------------|--------|
| S1 | Coupa unavailable | Coupa API returns 5xx or connection timeout at step 2.1–2.2 | High | [DEFAULT] No retry per BR-08 | Log error; mark run as failed; no notification sent |
| S2 | Coupa authentication failure | Coupa returns 401/403 at step 2.1 | High | [DEFAULT] No retry | Log error; alert via Orchestrator alert; no notification sent |
| S3 | Slack delivery failure | Slack API returns non-200 at step 4.2 | Medium | [DEFAULT] No retry per BR-08 | Log error; mark run as failed |
| S4 | Credential retrieval failure | Orchestrator asset not found or inaccessible | High | [DEFAULT] No retry | Log error; mark run as failed |
| S5 | Unhandled exception | Unexpected runtime error at any step | High | [DEFAULT] No retry | Log full exception; mark run as failed; no notification sent |

## 10. Assumptions, Dependencies and Open Questions

1. **OQ-01 - Coupa interface type.** Is the Coupa integration via REST API or UI automation? [SME REVIEW]
2. **OQ-02 - Coupa credential asset name.** What Orchestrator asset holds the Coupa API key/token? [SME REVIEW]
3. **OQ-03 - Slack delivery mechanism.** Is the Slack notification sent via Incoming Webhook or Bot token (chat.postMessage)? [SME REVIEW]
4. **OQ-04 - Coupa URL host for production.** Source shows uipath-test.coupahost.com; confirm production host before go-live. [SME REVIEW]
5. **OQ-05 - Clean-day message conflict.** Section 7 (exceptions) says send a congratulations message on a clean day; BR-07 and scope table say send nothing. BR-07 is treated as the authoritative rule here. [SME REVIEW]
6. **OQ-06 - Filtered URL covers both statuses.** The source URL example uses status_eq=draft only; confirm whether status=new should also appear in the Coupa link. [SME REVIEW]
7. **OQ-07 - Romania timezone DST handling.** Run time is 10:00 Europe/Bucharest; confirm Orchestrator trigger is configured with this IANA zone, not a fixed UTC offset. [DEFAULT UTC+3 assumed until confirmed]
8. **OQ-08 - Block Kit JSON payload.** Exact Block Kit JSON is described in prose; source states it is recorded verbatim in architectural considerations (Section 4) but that attachment is not present in the source file. Architect should confirm payload before build. [SME REVIEW]
9. **Coupa read-only access.** Automation has read-only access to Coupa per scope; no write permissions are provisioned.
10. **No AI classification required.** Rules are deterministic; no ML model or human-in-loop step is needed per source.

## 11. Success Criteria

1. A weekday run executes at 10:00 Europe/Bucharest and produces a logged result (message sent or clean-day exit).
2. Only invoices with status draft or new and invoice date within the past seven days are evaluated.
3. Credit notes are excluded from the qualifying count.
4. A PO reference present only in the description field is treated as a missing PO.
5. One Slack Block Kit direct message is sent to member ID WLX9BD8FN when invoice_count > 0, containing the correct count and a working filtered Coupa URL.
6. No Slack message is sent when invoice_count = 0.
7. The automation makes no changes to any Coupa record and creates no purchase orders.
8. A run that cannot complete is recorded as a failed run with no Slack output.
