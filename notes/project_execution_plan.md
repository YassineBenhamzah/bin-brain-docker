# Project Execution Plan: A Bin with a Brain

---

## Section 1 — Development Phases

---

### Phase 1: Docker Foundation & Service Skeletons

- Create `docker-compose.yml` defining four services: `mysql`, `laravel`, `ocr`, and `nextjs`, plus a named volume shared between `laravel` and `ocr`
- Scaffold a minimal Laravel app with `.env` pointing to the MySQL container; confirm `php artisan migrate` runs cleanly inside the container
- Scaffold a minimal Express app in the OCR service directory with a single `GET /health` endpoint that returns `{ status: "ok" }`
- Scaffold a minimal Next.js app with a single index page; confirm it resolves via the browser from the host

**Deliverable:** All four containers start without errors via `docker compose up`. The Next.js index page loads. The OCR service responds to `GET /health`. Laravel connects to MySQL and migrations run.

---

### Phase 2: File Upload Pipeline (Laravel → Volume)

- Create the `documents` MySQL table with columns: `id`, `original_filename`, `stored_filename`, `file_path`, `status` (`pending`/`processing`/`done`/`failed`), `uploaded_at`
- Implement `POST /api/documents` in Laravel: validate the uploaded file (PDF only, max size), store it to the shared volume path using a UUID-based filename, insert a `documents` row with `status = pending`
- Configure MySQL-backed queues in Laravel (`QUEUE_CONNECTION=database`); create the `jobs` table via `php artisan queue:table && php artisan migrate`
- Dispatch an `ProcessDocumentJob` from the upload controller immediately after saving the file; the job body can be empty for now

**Deliverable:** Upload a PDF via `curl` or Postman to `POST /api/documents`. The file appears in the shared volume. A row exists in `documents` with `status = pending`. A row exists in the `jobs` table.

---

### Phase 3: OCR Service — Core Extraction

- Implement `POST /process` on the Express OCR service: accept `{ filePath: string }` in the request body, read the file from the shared volume path, run Tesseract.js on it, and return `{ text: string, confidence: number }`
- Add per-page extraction for multi-page PDFs using `pdf-to-png-converter` or `pdfjs-dist` to rasterize pages before passing them to Tesseract; average confidence across pages
- Add structured error responses: if the file is not found, return `{ error: "file_not_found" }` with HTTP 404; if Tesseract fails, return `{ error: "ocr_failed" }` with HTTP 500

**Deliverable:** `POST /ocr/process` with a valid file path returns extracted text and a confidence score. A missing path returns a 404 JSON error. Testable with `curl` from inside the Laravel container using the shared volume path.

---

### Phase 4: Laravel Worker — Job Execution & Result Storage

- Implement `ProcessDocumentJob`: set document `status = processing`, call the OCR service at its internal Docker hostname (`http://ocr:3000/process`) via Laravel's HTTP client, handle the response
- Add `ocr_text` (`LONGTEXT`) and `confidence_score` (`FLOAT`) columns to the `documents` table; on success, write results and set `status = done`; on failure (non-200 or exception), set `status = failed` and store the error message in a `failure_reason` column
- Add `php artisan queue:work` to the Laravel container's startup command (or a separate worker container); configure `--tries=3` and `--timeout=120`

**Deliverable:** Upload a PDF. Within seconds (or after manually running `queue:work`), the document row in MySQL is updated with OCR text, confidence, and `status = done`. Inspect directly in MySQL.

---

### Phase 5: Next.js Frontend — Upload & Status UI

- Build a document upload form in Next.js that posts to `POST /api/documents` via the Laravel API; display upload confirmation and the new document's ID
- Build a document list page that polls `GET /api/documents` every 3 seconds and renders each document's `original_filename`, `status`, and `confidence_score`
- Build a document detail page at `/documents/[id]` that fetches `GET /api/documents/{id}` and renders the full extracted OCR text
- Add a search input on the list page that hits `GET /api/documents?q=term`; implement the search in Laravel using a `LIKE` query on `ocr_text`

**Deliverable:** End-to-end flow fully visible in the browser: upload a PDF, watch the status change from `pending` → `processing` → `done`, click through to see the extracted text, and search for a term that appears in the document.

---

### Phase 6: Hardening & Portfolio Polish

- Replace polling with a `status` endpoint that returns only `{ id, status, updated_at }` to minimize payload; add `updated_at` to the `documents` table
- Add input validation and meaningful error states in the Next.js UI (file type rejection client-side, API error messages rendered inline)
- Add a failed-document UI state with a retry button that re-dispatches the job via `POST /api/documents/{id}/retry`
- Write a `README.md` at the repo root documenting local setup in under 5 commands: `git clone`, `cp .env.example .env`, `docker compose up --build`, `php artisan migrate` (via exec), upload a test PDF

**Deliverable:** A complete, demo-able application. Any developer can clone and run it with the README instructions. Upload, processing, search, and error handling all behave correctly.

---

## Section 2 — First 7 Days Execution Plan

---

**Day 1:**
- Write the `docker-compose.yml` with all four services and the named volume; confirm all containers start
- Verify MySQL is reachable from the Laravel container (`php artisan migrate` succeeds)
- Verify the OCR container's `GET /health` is reachable from the Laravel container via `curl http://ocr:3000/health`

**Day 2:**
- Create the `documents` migration and run it
- Implement `POST /api/documents` in Laravel: file validation, UUID rename, volume storage, DB insert
- Test with Postman or curl: confirm the file exists on disk and the DB row is created

**Day 3:**
- Create the `jobs` table via artisan and configure `QUEUE_CONNECTION=database`
- Create `ProcessDocumentJob` as a stub (no HTTP call yet); dispatch it from the upload controller
- Confirm a row appears in the `jobs` table after upload via direct MySQL inspection

**Day 4:**
- Implement `POST /process` in the Express OCR service with Tesseract.js
- Test the OCR endpoint directly from inside the Laravel container: `curl -X POST http://ocr:3000/process -d '{"filePath":"/storage/test.pdf"}'`
- Resolve any volume path mismatch between what Laravel saves and what OCR expects

**Day 5:**
- Implement the full `ProcessDocumentJob` body: set status to `processing`, call OCR via HTTP client, write results, set status to `done` or `failed`
- Run `php artisan queue:work` manually inside the Laravel container and upload a PDF; inspect the `documents` row in MySQL for populated `ocr_text` and `confidence_score`

**Day 6:**
- Build the Next.js upload form and wire it to `POST /api/documents`
- Build the document list page with polling on `GET /api/documents`; display `status` per document
- Confirm the browser reflects the status transition from `pending` to `done` without a page refresh

**Day 7:**
- Build the document detail page to display OCR text
- Add the search input wired to `GET /api/documents?q=term`
- Do a full end-to-end manual test: upload → processing → done → view text → search for a word in the document

---

## Section 3 — Critical Early Decisions

---

### 1. Shared Volume Path Contract

**Decision:** The absolute path at which Laravel writes uploaded files must be identical to the path the OCR service reads from, with no transformation in between.

**Why costly to change later:** The file path is stored in the `documents` table and embedded in dispatched job payloads. Changing it after jobs are in the queue or after rows exist in the DB requires a migration, a data fix, and redeployment. A mismatch discovered late means jobs silently fail with "file not found" and you have no easy way to know which stored paths are wrong.

**Recommended approach:** Define a single environment variable (e.g. `STORAGE_BASE_PATH=/var/documents`) set identically in both the Laravel and OCR containers via the `docker-compose.yml` environment block. Store only the relative filename (UUID + extension) in the DB, not the full path. Reconstruct the full path at runtime in both services by prepending the env variable.

---

### 2. Job Payload Shape

**Decision:** What `ProcessDocumentJob` puts into the MySQL `jobs` table payload must be stable across schema changes.

**Why costly to change later:** Laravel serializes Eloquent models in job payloads by default. If the `documents` table schema changes (columns added/renamed), old serialized model payloads in the `jobs` table can fail to deserialize, causing queue workers to crash silently or log confusing errors. Unprocessed jobs already queued become unrestorable.

**Recommended approach:** Serialize only the `document_id` (integer) in the job payload. Inside `handle()`, fetch the fresh model from the DB using that ID. This makes the payload schema-agnostic and immune to model changes.

---

### 3. OCR Service Internal API Contract

**Decision:** The request/response shape of `POST /ocr/process` must be defined before Laravel calls it.

**Why costly to change later:** The Laravel job hardcodes the OCR endpoint URL and parses its response. If the OCR response shape changes (e.g. `confidence` renamed to `score`, or text moved into a nested key), the worker silently stores `null` instead of the real value — no exception is thrown. Fixing it requires updating the worker, redeploying, and re-processing affected documents.

**Recommended approach:** Lock the contract on Day 4. Request: `{ "filePath": "<absolute path>" }`. Response: `{ "text": "<extracted text>", "confidence": <0.0–100.0> }`. Error: `{ "error": "<error_code>" }`. Do not change these field names after the Laravel worker is implemented.

---

### 4. `status` Column Enum Values

**Decision:** The set of valid `status` string values must be finalized before the frontend renders them.

**Why costly to change later:** If you later add a status (e.g. `queued` vs `pending`) or rename one, you need a DB migration, a worker update, and a frontend update simultaneously. If any of the three is out of sync, the frontend renders a blank or unknown state for documents stuck in an old status, with no error.

**Recommended approach:** Fix exactly four values from the start: `pending`, `processing`, `done`, `failed`. Use a MySQL `ENUM` column to enforce this at the DB level. Never add an in-between status without updating all three services at once.

---

## Section 4 — Silent Failure Risks

---

### 1. File Saved, Job Never Dispatched

**Failure scenario:** The upload controller saves the file and inserts the DB row, but an unhandled exception between the insert and the `dispatch()` call causes the job to never be dispatched. The document sits at `status = pending` forever.

**Why dangerous:** The user sees "Upload successful." The frontend shows `pending` indefinitely. No exception is thrown to the user. No row appears in the `failed_jobs` table because the job was never queued — it simply doesn't exist.

**Detection:** Add a scheduled artisan command (daily) that queries for documents with `status = pending` and `uploaded_at` older than 10 minutes and logs them or re-dispatches their jobs.

**Prevention:** Wrap the file write, DB insert, and job dispatch in a single database transaction. If dispatch fails, the transaction rolls back and the upload returns a 500 to the client — a visible failure instead of a silent one.

---

### 2. OCR Returns Empty Text with Zero Confidence

**Failure scenario:** Tesseract processes the file successfully (exit code 0, HTTP 200 from OCR service) but the PDF contains only scanned images with no recoverable text, or the image quality is too low. The response is `{ "text": "", "confidence": 0 }`.

**Why dangerous:** The worker treats this as a success: `status = done`, `ocr_text = ""`, `confidence_score = 0`. The document appears complete in the UI. The user searches for content and finds nothing. There is no error, no indication the document was unreadable.

**Detection:** In the worker, after receiving a 200 response, check if `text` is empty or `confidence` is below a threshold (e.g. 10). Log a warning with the document ID. Optionally set a `low_confidence` boolean flag in the DB.

**Prevention:** Treat `confidence < 10` or `text length === 0` as a soft failure: store the result but set `status = done_low_confidence` (or use the `low_confidence` flag) so the frontend can display a warning banner on the document detail page.

---

### 3. Queue Worker Not Running — Jobs Accumulate Silently

**Failure scenario:** The Laravel container starts but `queue:work` is not running (e.g. the supervisor process crashed, or the worker container was never started). Documents are uploaded, jobs pile up in the `jobs` table, and nothing is processed.

**Why dangerous:** Uploads appear successful. The frontend shows `pending` for every document. The developer assumes processing is slow. There is no log output because nothing is running to log. The `jobs` table grows but nothing reads it.

**Detection:** Add `GET /api/queue/health` in Laravel that queries `SELECT COUNT(*) FROM jobs WHERE created_at < NOW() - INTERVAL 5 MINUTE` and returns a warning if unprocessed old jobs exist. Poll this from the Next.js admin view.

**Prevention:** Run `queue:work` as a supervised process inside the Laravel container using a `supervisord.conf` so it restarts automatically on crash. Log worker startup to stdout so Docker logs make it visible.

---

### 4. Volume Mount Path Mismatch — File Written to Wrong Location

**Failure scenario:** The Laravel container volume is mounted at `/var/documents` but the OCR container volume is mounted at `/data/documents`. Laravel saves to `/var/documents/uuid.pdf` and stores that path. The OCR service tries to read `/var/documents/uuid.pdf` from its own filesystem — which is a different physical path — and the file does not exist.

**Why dangerous:** The OCR service returns `{ "error": "file_not_found" }`. If the worker treats any non-200 as a generic failure, it sets `status = failed` with no further detail. The developer sees failed documents but does not know why — there is no path mismatch logged anywhere.

**Detection:** In the OCR service, when returning a file-not-found error, include the attempted file path in the response body: `{ "error": "file_not_found", "attempted_path": "/var/documents/uuid.pdf" }`. Log this in the Laravel worker alongside the document ID.

**Prevention:** Declare the volume mount point identically in both service entries in `docker-compose.yml`. Use the same `STORAGE_BASE_PATH` environment variable in both containers. Verify on Day 4 by curling the OCR service from inside the Laravel container with an actual stored file path before wiring the worker.

---

### 5. Confidence Score Stored as Null Due to JSON Key Mismatch

**Failure scenario:** The OCR service returns `{ "text": "...", "confidence": 87.3 }` but the Laravel worker reads `$response->json('confidence_score')` instead of `$response->json('confidence')`. The `confidence_score` column is written as `null`. No exception is thrown because `json()` returns `null` for missing keys.

**Why dangerous:** The document status is `done`, the text is correct, but `confidence_score` is `null` in every row. The frontend either renders "null%" or nothing. The developer may assume confidence is not being returned by Tesseract rather than suspecting a key name mismatch.

**Detection:** After writing the result, assert that `confidence_score IS NOT NULL` in the worker before marking the job complete. If null, log the raw response body from the OCR service alongside the document ID.

**Prevention:** Define the JSON key names as constants in the Laravel worker (e.g. `const OCR_TEXT_KEY = 'text'; const OCR_CONFIDENCE_KEY = 'confidence';`) rather than inline strings. Match these explicitly against the OCR service contract defined in Section 3.

---

## Section 5 — Service Integration Order

---

### Integration Step 1: Laravel API ↔ MySQL (Queue)

**What is being connected:** Laravel's queue driver to the MySQL `jobs` table.

**Minimum viable integration test:** Upload a document via `POST /api/documents`. Then run `SELECT COUNT(*) FROM jobs;` directly in the MySQL container. The count must be 1. Run `php artisan queue:work --once` inside the Laravel container and confirm the count drops to 0 (job consumed, even if it does nothing yet).

**What breaks if skipped:** If the queue connection is misconfigured, `dispatch()` throws a `PDOException` or silently discards the job depending on the `QUEUE_CONNECTION` setting. You will not know jobs are lost until Phase 4 when you expect processing to happen and nothing does.

---

### Integration Step 2: Laravel Worker ↔ OCR Service (HTTP)

**What is being connected:** The `ProcessDocumentJob` inside the Laravel container calling `POST http://ocr:3000/process` over the Docker internal network.

**Minimum viable integration test:** Before implementing the full job, run this from inside the Laravel container: `curl -X POST http://ocr:3000/process -H "Content-Type: application/json" -d '{"filePath":"/var/documents/test.pdf"}'`. Place a real PDF at that path first. Confirm you receive `{ "text": "...", "confidence": ... }` back. Only then wire the HTTP call into the job.

**What breaks if skipped:** The OCR hostname `ocr` only resolves inside the Docker network. If you develop and test the worker on the host or with a hardcoded IP, the call works in testing but fails silently in production (inside Docker) because the hostname or port is wrong. You will see `status = failed` with a connection-refused or DNS-resolution error that only appears in Docker logs, not in the DB.

---

### Integration Step 3: Next.js Frontend ↔ Laravel API

**What is being connected:** The Next.js app making HTTP requests to the Laravel API from the browser (host machine) using the host-mapped port (e.g. `http://localhost:8000/api`).

**Minimum viable integration test:** Before building any UI, make a raw `fetch('http://localhost:8000/api/documents')` from the browser console and confirm you receive a JSON array (even if empty). Confirm CORS headers are present in the response. Then build UI on top of confirmed working endpoints only.

**What breaks if skipped:** CORS misconfiguration in Laravel means every API call from Next.js silently fails in the browser with a CORS error — not a 4xx response, but a network-level block. The Next.js `fetch` returns a generic network error with no response body. This is indistinguishable from the API being down, and developers frequently spend hours debugging application logic before checking CORS headers.

---

### Integration Step 4: End-to-End Smoke Test (All Three Services)

**What is being connected:** The full pipeline: browser → Laravel → MySQL queue → Laravel worker → OCR service → MySQL result → browser.

**Minimum viable integration test:** Upload a single-page, text-based PDF through the Next.js UI. Without touching any terminal, watch the document list page. Within 60 seconds, the status must transition from `pending` to `done` and the document detail page must display non-empty OCR text. This test must pass before any polish work begins.

**What breaks if skipped:** Each service in isolation can appear to work correctly while the full pipeline is broken at a seam. Volume paths, queue consumption timing, OCR response parsing, and DB writes can all have subtle bugs that only appear when all services are running together. Deferring this test until Phase 5 means discovering integration bugs after substantial UI work has been built on top of a broken foundation.
