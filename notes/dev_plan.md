# A Bin with a Brain — Dev Plan

## Phase 1: Docker & Service Skeletons
- [ ] `docker-compose.yml` — 4 services: `mysql`, `laravel`, `ocr`, `nextjs` + named shared volume
- [ ] Laravel `.env` wired to MySQL container; `php artisan migrate` runs clean
- [ ] OCR Express app — `GET /health` returns `{ status: "ok" }`
- [ ] Next.js index page loads in browser from host

**✅ Done when:** all 4 containers start, health check passes, Next.js renders

---

## Phase 2: File Upload Pipeline
- [ ] `documents` table — `id`, `original_filename`, `stored_filename`, `file_path`, `status`, `uploaded_at`
- [ ] `POST /api/documents` — validate PDF, save to shared volume with UUID filename, insert DB row (`status = pending`)
- [ ] `jobs` table — `php artisan queue:table && migrate`, set `QUEUE_CONNECTION=database`
- [ ] Dispatch `ProcessDocumentJob` after upload (stub body is fine)

**✅ Done when:** upload via curl → file on volume + row in `documents` + row in `jobs`

---

## Phase 3: OCR Service — Extraction
- [ ] `POST /process` — accepts `{ filePath }`, runs Tesseract.js, returns `{ text, confidence }`
- [ ] Multi-page PDF support — rasterize pages, average confidence
- [ ] Error responses — `404` file not found, `500` OCR failed (include `attempted_path` in body)

**✅ Done when:** curl from inside Laravel container returns text + confidence for a real PDF

---

## Phase 4: Laravel Worker — Job Execution
- [ ] `ProcessDocumentJob` — set `status = processing`, call `http://ocr:3000/process`, handle response
- [ ] Add `ocr_text` (LONGTEXT), `confidence_score` (FLOAT), `failure_reason`, `updated_at` columns
- [ ] On success → write text/confidence, `status = done`; on failure → `status = failed` + reason
- [ ] Run `queue:work` via `supervisord` inside container (`--tries=3 --timeout=120`)

**✅ Done when:** upload PDF → worker auto-picks job → DB row shows `done` + OCR text

---

## Phase 5: Next.js Frontend
- [ ] Upload form → `POST /api/documents` → show confirmation + document ID
- [ ] Document list page → polls `GET /api/documents` every 3s → shows filename, status, confidence
- [ ] Document detail page `/documents/[id]` → shows full OCR text
- [ ] Search input → `GET /api/documents?q=term` → Laravel `LIKE` on `ocr_text`
- [ ] Failed document state → retry button → `POST /api/documents/{id}/retry` re-dispatches job

**✅ Done when:** full browser flow: upload → watch status update → read text → search

---

## Phase 6: Hardening
- [ ] CORS configured in Laravel for `http://localhost:3000`
- [ ] `GET /api/queue/health` → warn if jobs older than 5 min are unprocessed
- [ ] Validate `confidence < 10` or empty text → set `low_confidence` flag, show warning in UI
- [ ] Client-side file-type + size validation in Next.js upload form
- [ ] `README.md` — clone → `.env` → `docker compose up --build` → migrate → upload test PDF

**✅ Done when:** demo-ready; any dev can run it from the README in 5 commands

---

## API Reference

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/documents` | Upload PDF |
| GET | `/api/documents` | List all (supports `?q=`) |
| GET | `/api/documents/{id}` | Get single doc + OCR text |
| POST | `/api/documents/{id}/retry` | Re-queue failed doc |
| GET | `/api/queue/health` | Queue backlog check |

---

## Document Status Values
`pending` → `processing` → `done` / `failed`

---

## Key Constraints (do not forget)
- Store only the **relative filename** in DB; prepend `STORAGE_BASE_PATH` env var at runtime in both Laravel and OCR containers
- Job payload = **`document_id` only** (no serialized Eloquent model)
- OCR response keys are **`text`** and **`confidence`** — never rename them
- Next.js never calls the OCR service directly — all traffic via Laravel API
