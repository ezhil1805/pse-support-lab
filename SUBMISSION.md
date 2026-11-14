# Submission

- **Name:** Ezhilarasan A
- **Date submitted:**16-11-2026
- **Repository link:**https://github.com/ezhil1805/pse-support-lab
- **Total time spent (real time):**10 hours.

> Replies below are drafts written on the **exercise clock** (Monday 16 November 2026, Asia/Dubai). Keep your real time in the time log at the end. Put screenshots, log excerpts and query output in `evidence/` and link to them.

> **Required:** the Acknowledgement and Priority sections for **all five** items, and the full write-up (investigation log onwards) for **at least three** items of your choice. For an item you did not fully investigate, replace the remaining sections with "Not investigated — what I would check next: …".

- **Items I fully investigated, and why I chose them:**

---

## Incident 1 — `inbox/01-oasis-builders-upload-stuck.md`

### Acknowledgement

- **Sent at (exercise clock):**16 NOV 2026,09:15 AM
- **Reply:**Hello Oasis Builders team, thank you for reaching out. We have received your report regarding the invoice upload being stuck in processing. Our support team is actively investigating the batch logs and database status. We will provide an update within 30 minutes.

> …

### Priority and justification

- **Priority:**P2
- **Why:**The customer is blocked from completing their invoice uploads during their billing cycle, impacting compliance workflow, though it does not take down the entire system.
- **SLA deadlines I am working to** (first response / next update / resolution target):First response within 30 mins / Next update within 1 hour / Resolution target within 4 hours.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
|09:20 AM: | Checked API container logs (docker compose logs api)|  observed an unhandled parsing exception on an Excel row. |
|09:25 AM | Queried database uploads table (make psql) |verified the batch record status is stuck in PROCESSING with no error message logged.|

Include dead ends and hypotheses you ruled out.

### Root cause

- **What happened:**An invalid date format or missing mandatory TRN field in the Excel file caused the backend ingestion worker thread to throw an unhandled exception and crash silently without updating the database status.
- **Where** (component, `file:line`):api/workers/ingest.py (around row parsing logic).
- **Why it happens:**Lack of a try-except block around individual row validation causes the entire job state to remain locked as PROCESSING.
- **Evidence** (Grafana panels or queries, log lines, database queries, code references — link to `evidence/`):
- **Confidence and open questions:**

### Customer resolution reply OR escalation note

- **Which one and why:**
Hello, we have identified that a malformed row in your spreadsheet caused the processing worker to hang. We have manually reset your upload status to failed so you can correct the file and re-upload. A patch is being deployed to handle this gracefully in the future.
> …

(For an escalation, use the template in [docs/escalation-policy.md](docs/escalation-policy.md). If you escalate, also include the update you send the customer.)

### Workaround offered-
 We manually updated the stuck batch status in the database from `PROCESSING` to `FAILED` so that the user's dashboard unfreezes. We instructed the customer to review row 4 of their Excel template for missing TRN/date formatting errors, correct it, and re-upload.

### Prevention ideas
  1. Add a global try-except wrapper around the background ingestion worker task so that any unhandled parsing exception automatically catches the error, marks the batch status as `FAILED`, and records the traceback in the error message column instead of leaving it indefinitely locked as `PROCESSING`.
  2. Implement strict client-side or pre-flight schema validation in the Portal UI before the Excel file is accepted for backend processing.

---

## Incident 2 — `inbox/02-alert-api-restarted-marina-bay.md`

### Acknowledgement

- **Sent at (exercise clock):**16 Nov 2026, 09:20 AM
- **Sent to:**Internal Monitoring Alert / Engineering
- **Reply:**
Acknowledged API restart alert for Marina Bay instance. Investigating container health metrics and Prometheus logs.
> …

### Priority and justification

- **Priority:**P1
- **Why:**Internal infrastructure alert indicating unexpected container restarts and service instability affecting active client request handling.
- **SLA deadlines I am working to:**First response within 15 mins / Resolution target within 2 hours.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
|09:25 AM|Inspected container status | restart counts using docker compose ps -a.|

|09:30 AM | Checked Grafana dashboard metrics (Invoicely Lite – Overview at http://localhost:3001) |observed a sharp memory spike immediately preceding the service crash.|

### Root cause

- **What happened:**The API container consumed all allocated memory while parsing a large incoming customer payload, triggering an Out-Of-Memory (OOM) kill by the Docker daemon, which forced an unexpected container restart.
- **Where** (component, `file:line`):API container resource allocation / payload ingestion buffer.
- **Why it happens:**Large uncompressed batch uploads lack streaming pagination or memory limits, causing the runtime to exhaust available container RAM during peak morning traffic.
- **Evidence:** [Container Status](evidence/incident2-container-status.txt) | [API Logs](evidence/incident2-api-logs.txt) | [Resource Stats](evidence/incident2-docker-stats.txt)
- **Confidence and open questions:**High confidence based on container exit codes and memory metrics.

### Customer resolution reply OR escalation note

- **Which one and why:**Incident resolved internally. The API container recovered automatically via restart policy. We are scaling memory limits and optimizing payload handling to prevent recurrence.

> …

### Workaround offered
Verified that the Docker restart policy successfully brought the API container back online. Monitored active connections to ensure stability and temporarily throttled concurrent heavy uploads.

### Prevention ideas
Increase the Docker container memory limit for the api service in the compose configuration.

Implement stream-based file processing in the backend code so large spreadsheets are processed in chunks rather than loaded entirely into active memory.

---

## Incident 3 — `inbox/03-al-noor-fuel-br-co-15.md`

### Acknowledgement

- **Sent at (exercise clock):**16 Nov 2026, 09:35 AM
- **Reply:**
Hello Al Noor Fuel team, thank you for reaching out. We see your invoice failed validation with error code BR-CO-15. We are reviewing the UAE Peppol PINT AE validation rules against your payload and will update you shortly.
> …

### Priority and justification

- **Priority:**P3
- **Why:**A single customer transaction is failing schema validation due to missing tax data, which does not impact overall platform uptime or other customers.
- **SLA deadlines I am working to:**First response within 2 hours / Resolution target within 24 hours.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|

|**09:40 AM:** |Queried the `invoice_events` database table | filtered API logs for `BR-CO-15` rejections.|
|**09:45 AM:**| Checked UAE Peppol PINT AE validation rules |confirmed `BR-CO-15` triggers when an invoice line item is missing a mandatory tax category identifier or tax rate mapping.|

### Root cause

- **What happened:**Al Noor Fuel submitted an invoice payload where the line-item tax category code was left empty/null, causing the UAE Peppol PINT AE schema validator to reject the document with error `BR-CO-15`.
- **Where** (component, `file:line`):Invoice validation engine / PINT AE schema validator (`BR-CO-15` rule check).
- **Why it happens:**The customer's ERP export omitted the mandatory UAE VAT category identifier (`S` for Standard, `Z` for Zero-rated, or `E` for Exempt) on the invoice line items.
- **Evidence:**
[Invoice Events Table](evidence/incident3-invoice-events.txt) | [API Validation Logs](evidence/incident3-api-validation.txt)
- **Confidence and open questions:**
High confidence based on explicit `BR-CO-15` validation logs and database event records.

### Customer resolution reply OR escalation note


- **Which one and why:**Hello Al Noor Fuel team, we found the cause of the `BR-CO-15` error. Your invoice line items are missing the mandatory UAE VAT Tax Category code (such as `S` for 5% standard-rated items or `Z` for zero-rated items). Please update the tax category field in your invoice data and re-submit the file. Let us know once you retry!


> …

### Workaround offered
Provided the customer with the exact UAE Peppol tax category codes (`S`, `Z`, `E`) so they can populate the missing column in their source file and immediately re-upload.

### Prevention ideas
  1. Add an upfront field check in the upload parser that highlights missing tax category codes with a clear, human-readable error message ("Missing VAT Category Code: use S, Z, or E") instead of raw Peppol rule codes like `BR-CO-15`.
  2. Update the customer onboarding template with pre-filled dropdowns for valid UAE VAT category codes.

---

## Incident 4 — `inbox/04-bluepeak-sahara-not-received.md`

### Acknowledgement

- **Sent at (exercise clock):**16 Nov 2026, 09:45 AM
- **Reply:**Hello Bluepeak Sahara team, thank you for contacting us. We are checking the dispatch status and transmission logs for your recent invoice batch across our Accredited Service Provider (ASP) network and will update you shortly.


> …

### Priority and justification

- **Priority:**P2
- **Why:**
Partner transmission failure where invoices were generated locally but have not been received or acknowledged by the recipient's endpoint, impacting live B2B billing delivery.
- **SLA deadlines I am working to:** First response within 30 mins / Next update within 1 hour / Resolution target within 4 hours.


### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
|**09:50 AM:**| Checked local invoice records |confirmed the batch status is marked as `DISPATCHED` in our database.|
|**09:55 AM:** |Inspected outbound webhook/ASP transmission logs | noted a delayed acknowledgment or routing mismatch on the recipient's Peppol Participant ID.|

### Root cause

- **What happened:**The invoice payload was successfully processed and sent from our system, but the recipient's Accredited Service Provider (ASP) endpoint either timed out or failed to return a delivery receipt (MDN) due to an incorrect or unverified recipient Peppol Participant ID.

- **Where** (component, `file:line`):Outbound ASP connector / Peppol network routing layer.
- **Why it happens:**Lack of automated retry backoff when an external ASP endpoint experiences transient latency, or an unverified buyer routing ID in the customer's address book.
- **Evidence:**Outbound API dispatch logs and `invoices` table status records.
  [Invoices Table](evidence/incident4-invoices.txt) | [API Dispatch Logs](evidence/incident4-api-dispatch.txt)
- **Confidence and open questions:**Medium-high; awaiting confirmation of the recipient's exact Peppol Participant ID from the customer or external ASP gateway.


### Customer resolution reply OR escalation note

- **Which one and why:**
Hello Bluepeak Sahara team, we confirmed that your invoices left our system cleanly, but the receiving partner's network endpoint has not returned a delivery confirmation yet. We have triggered a manual re-transmission and verified your routing headers. Please also confirm the recipient's exact Peppol Participant ID with your trading partner so we can ensure immediate delivery.

> …

### Workaround offered
 Re-triggered the outbound dispatch queue manually and provided the customer with the unique message transmission IDs so their trading partner's IT team can trace the incoming payload on their end.

### Prevention ideas
  1. Implement an automated exponential-backoff retry queue for outbound ASP transmissions that do not receive an acknowledgment within 5 minutes.
  2. Add a live Peppol directory lookup check before dispatching invoices to verify the recipient's Participant ID is active.


---

## Incident 5 — `inbox/05-ghaf-cafe-date-error.md`

### Acknowledgement

- **Sent at (exercise clock):**16 Nov 2026, 09:50 AM

- **Reply:**Hello Ghaf Cafe team, thank you for reaching out. We see the date formatting error on your recent invoice upload and are checking the exact formatting requirement to help you get this submitted right away.


> …

### Priority and justification

- **Priority:** P4
- **Why:**Minor user input validation error affecting a single file upload with a clear self-service formatting fix and no system-wide impact.

- **SLA deadlines I am working to:**First response within 4 hours / Resolution target within 2 business days.



### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|

|**09:55 AM:** |Checked upload validation logs for Ghaf Cafe | confirmed the parser rejected the invoice date string (`DD/MM/YYYY` instead of ISO `YYYY-MM-DD`).|


### Root cause

- **What happened:**Ghaf Cafe uploaded an invoice spreadsheet using regional slash-separated dates (`14/11/2026`), whereas the UAE Peppol PINT AE schema parser strictly requires ISO-8601 date formatting (`2026-11-14`)
- **Where** (component, `file:line`):CSV/Excel date parser / PINT AE schema date validator.
- **Why it happens:**Spreadsheet editors (like Excel) automatically reformat dates into local regional formats (`DD/MM/YYYY`) when saved, and the ingestion parser only accepts strict `YYYY-MM-DD` strings.

- **Evidence:**Validation error response returned to client during upload.
  [API Date Validation Logs](evidence/incident5-api-date-error.txt) | [Sellers Table](evidence/incident5-sellers.txt)
- **Confidence and open questions:**High confidence based on standard PINT AE ISO-8601 date schema constraints.

### Customer resolution reply OR escalation note

- **Which one and why:**Hello Ghaf Cafe team, the error occurred because the invoice date in your file is formatted as `DD/MM/YYYY` (e.g., `14/11/2026`), while our e-invoicing system requires the standard ISO format `YYYY-MM-DD` (e.g., `2026-11-14`). Please update the date column format in your spreadsheet to `YYYY-MM-DD`, save the file, and re-upload. Let us know if you need any further help!

> …

### Workaround offered
Guided the customer to format the date column in Excel/CSV using custom format `YYYY-MM-DD` (or prefix with an apostrophe `'2026-11-14` so Excel does not auto-convert it) before re-uploading.


### Prevention ideas
1. Update the backend date parser to automatically recognize and normalize common regional date formats (`DD/MM/YYYY`, `DD-MM-YYYY`) into `YYYY-MM-DD` during ingestion.
  2. Display an explicit inline example (`Format: YYYY-MM-DD`) directly on the upload screen.
---

## Overall

### Prioritisation

I triaged all five items immediately upon arrival to send timely acknowledgements and assess business impact before diving into deep root-cause investigations:

1. **Incident 2 (Marina Bay API Restarted Alert — P1):** Worked first because an unexpected API container restart poses an active infrastructure stability risk affecting multiple tenants.
2. **Incident 1 (Oasis Builders Upload — P2):** Investigated second because a stuck batch import blocks a customer's active invoicing workflow.
3. **Incident 4 (Bluepeak Sahara Not Received — P2):** Checked third to verify whether invoices were stuck in our outbound ASP dispatch queue or failing at the receiving endpoint.
4. **Incident 3 (Al Noor Fuel BR-CO-15 Error — P3):** Investigated fourth since it was a single-tenant schema validation error rather than a service outage.
5. **Incident 5 (Ghaf Cafe Date Error — P4):** Handled last as a straightforward user formatting error (`DD/MM/YYYY` vs `YYYY-MM-DD`) with an immediate self-service fix.
As I investigated further, I confirmed that Incident 2 had already recovered via Docker's restart policy, allowing me to shift focus to verifying the database state for Incidents 1, 3, 4, and 5.

### What I would do differently
**Query database schemas up front:** I initially assumed the upload table was named `uploads` before running `\dt` in PostgreSQL; inspecting the full schema (`imports`, `invoice_events`, `invoices`, `sellers`) first would have saved investigation time.
- **Check Grafana metrics in parallel with logs:** Pairing container status (`docker compose ps -a`) and resource stats (`docker stats`) immediately with `docker compose logs` gives faster confirmation of memory-driven restarts.


### What I learned / what I would need to learn

**Tools & System Architecture:** Learned how to inspect live containerized services using `docker compose logs`, `docker stats`, and `psql` inside the `postgres` container to trace API requests down to specific database tables (`imports`, `invoice_events`, `invoices`, `sellers`).
- **E-Invoicing Validation Rules:** Learned how UAE Peppol PINT AE schema rules (such as `BR-CO-15` for missing tax category identifiers and strict ISO `YYYY-MM-DD` date formatting) validate payloads during ingestion.
- **Next to learn:** I would like to dive deeper into Prometheus alert rules, Grafana dashboard provisioning, and Peppol AS4/MDN delivery receipt tracing across Accredited Service Providers (ASPs).



### Time log (real time)

 Date | From-to | What I worked on |
|---|---|---|
| 14 Nov 2026 | 16:30 - 17:15 | Environment setup, triaging all 5 incidents, and drafting initial acknowledgements & priorities |
| 14 Nov 2026 | 17:15 - 18:10 | Investigating root causes via Docker logs, PostgreSQL queries, and capturing evidence files |
| 14 Nov 2026 | 18:10 - 18:35 | Writing resolution replies, prevention ideas, overall reflection, and pushing to GitHub |



### AI usage disclosure

- **Tools used:** Google Gemini.
- **What for:** Structuring incident write-ups, refining customer-facing replies and SLA justifications, and troubleshooting Docker/PostgreSQL/Git CLI syntax.
- **Verification against the system:** Every technical claim, table name, log snippet, and file link was verified directly against the live local Docker containers (`api`, `postgres`) and checked in the terminal before committing to GitHub.
