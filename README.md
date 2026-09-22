# Sheets → API → CRM: Automated Lead Sync to HubSpot

### Short Description
An n8n automation that polls a Google Sheet on a schedule, identifies new/unprocessed leads, checks HubSpot for existing contacts, creates or updates the contact record, and marks the source row as `Processed` — closing the loop between a simple lead-capture sheet and a production CRM with zero manual data entry.

---

## README

### Title
**Sheets → API → CRM (Automated Lead Sync Pipeline)**

### Short Description — Why is it needed
Most lead-gen funnels start messy — a Google Form, a landing page, or a manual intake sheet — long before there's budget or bandwidth to wire up a native CRM integration. Sales and RevOps teams end up copy-pasting leads into HubSpot by hand, which is slow, error-prone, and guarantees duplicate or stale contact records.

This workflow removes that manual step entirely. It treats the Google Sheet as a lightweight lead inbox and HubSpot as the system of record, syncing the two automatically and idempotently — so every lead lands in the CRM exactly once, correctly deduplicated, with no engineering lift required to stand it up.

### Problem & Goal
**Problem:** Leads captured in a spreadsheet don't automatically become CRM contacts. Manual entry introduces delay (leads go cold), human error (typos, missed rows), and duplication (no reliable check against existing contacts).

**Goal:**
- Automatically pick up new leads from a shared Google Sheet on a recurring schedule.
- Prevent duplicate CRM records by checking HubSpot for an existing contact before creating a new one.
- Guarantee each row is processed exactly once, even across multiple scheduled runs.
- Keep the sheet as an accurate, auditable log of sync status (`Processed` vs. pending).

### Architecture
```
Schedule Trigger
      │
      ▼
Get row(s) in Sheet  (Google Sheets — "CRM" tab)
      │
      ▼
Filter  (Status ≠ "Processed")
      │
      ▼
Loop Over Items  (Split In Batches — one lead at a time)
      │
      ▼
Search Contacts  (HubSpot — lookup by email)
      │
      ▼
If  (total contacts found == 0 ?)
      │
      ├── True (new lead)  ──┐
      │                      ▼
      └── False (existing) ──► Create or Update Contact (HubSpot Upsert)
                                       │
                                       ▼
                              Mark as Processed (Google Sheets update)
                                       │
                                       ▼
                              back to Loop Over Items (next batch)
```
The loop keeps running batch-by-batch until every unprocessed row in the current run has been synced and flagged, then the workflow idles until the next scheduled trigger.

### Tools & Integrations Used
| Tool | Role |
|---|---|
| **n8n** | Orchestration engine running the end-to-end automation |
| **Schedule Trigger** | Kicks off the sync on a recurring interval — no manual intervention |
| **Google Sheets API** | Source of truth for inbound leads; also the audit log for processed status |
| **HubSpot API** | Destination CRM — contact search (dedupe check) + create/update (upsert) |
| **Filter Node** | Cheap pre-filter so only unprocessed rows enter the expensive API loop |
| **Split In Batches (Loop)** | Processes leads one at a time to keep API calls sequential and traceable |
| **If Node** | Branches on HubSpot search result count to decide create vs. update path |

### Product Decisions
- **Sheet as inbox, HubSpot as source of truth.** The sheet is intentionally kept simple (Date, Name, email, Number, Status) — it's a capture surface, not a data model. All contact logic lives in HubSpot.
- **Status column instead of row deletion.** Marking rows `Processed` (rather than deleting them) preserves a full audit trail of every lead that ever entered the funnel, which matters for reporting and debugging.
- **Search-before-create ("upsert" pattern).** Rather than blindly creating a HubSpot contact per row, the workflow always searches by email first. This was a deliberate call to prioritize CRM data hygiene over pipeline simplicity — duplicate contacts are far more expensive to clean up later than one extra API call up front.
- **Both branches of the `If` converge on the same "Create or Update" node.** HubSpot's upsert-by-email behavior is used deliberately here, so a single node safely handles both "no match" and "match found" outcomes without branching logic duplication.
- **Batch-of-one looping.** Processing leads individually (vs. bulk) trades a bit of speed for much simpler error isolation — one bad row won't block the rest of the batch, and the sheet's `Status` update happens per-lead, so partial runs are always in a consistent state.

### How Workflow Response Is Evaluated
- **Correctness:** A lead should always resolve to exactly one HubSpot contact — verified by checking the `total` count returned from the HubSpot search before deciding create vs. update.
- **Idempotency:** Re-running the workflow should never re-process an already-`Processed` row — enforced by the `Filter` node reading current sheet state on every run.
- **Completeness:** Every unprocessed row present at trigger time should exit the loop marked `Processed`; any row left "stuck" mid-status signals a failure worth investigating (e.g., HubSpot auth issue, malformed email).
- **Latency:** Time between a lead landing in the sheet and appearing in HubSpot is bounded by the Schedule Trigger interval — the tighter the interval, the fresher the CRM data.

### Limitations & Next Steps
- **No error handling / retry logic yet.** A failed HubSpot call (rate limit, auth expiry) currently stalls that row without marking it processed — needs an error-branch with logging/alerting.
- **No field-level validation.** Malformed emails or empty required fields are passed through as-is; a validation step before the HubSpot search would harden this.
- **Single-sheet, single-pipeline scope.** Currently hardcoded to one spreadsheet/tab and one HubSpot pipeline — multi-source lead intake would need a source-tagging layer.
- **No notification loop.** Sales reps aren't proactively notified when a new contact is created — a Slack/email ping on successful sync is a natural next iteration.
- **Status is binary.** Only `Processed` / not-processed exists today; adding `Failed` / `Retrying` states would make the sheet a true operational dashboard rather than just a log.

---

## License
This project is licensed under the **MIT License** — see the [LICENSE](./LICENSE) file for details.

---

## Workflow JSON
📄 [`Project_8_Sheets_API_CRM.json`](./Project_8_Sheets_API_CRM.json) — attached in this repository.

## Workflow Screenshot
📸 `screenshot.png` — attached in this repository. *(Add your exported n8n canvas screenshot here.)*
