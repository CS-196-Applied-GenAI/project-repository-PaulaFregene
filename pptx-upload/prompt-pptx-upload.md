## Prompt 0 — Operating rules (read first)

```text
You are implementing the backend changes described in:
- `documentations/pptx-upload/spec-pptx-upload.md`
- `documentations/pptx-upload/plan-pptx-upload.md`

Repo constraints:
- Backend is FastAPI in `backend/` (Python).
- Tests use `pytest`.
- Production uses `USE_S3=true` and S3 prefixes `pdfs/`, `incoming/`, `jobs/`.

Global implementation rules:
- Work test-first: add/adjust tests before/with implementation.
- No orphaned code: every new module must be imported/used by a route or another used module in the same prompt.
- Keep changes small and incremental. Avoid large refactors.
- Don’t weaken security: admin-only endpoints must be backend-enforced (not UI-only).
- After edits, run backend tests (`pytest -q`) and fix failures.

Deliverable for each prompt:
- Implement exactly what’s requested.
- Add/adjust tests.
- Ensure code is wired into the running app (`backend/main.py` routes/dependencies) and tests pass.
```

---

## Prompt 1 — PR 1: Auth scaffolding (no endpoint changes yet)

```text
Goal: Add Clerk-auth scaffolding utilities without changing any routes yet.

Implement:
1) Create `backend/auth_clerk.py` with:
   - `extract_bearer_token(request: fastapi.Request) -> str | None`
     - Reads `Authorization` header.
     - Accepts `Bearer <token>`; returns token string or None.
     - Must be robust to missing/empty/malformed header.
   - `is_admin_claims(claims: dict) -> bool`
     - Returns True iff claims contain `publicMetadata.role` exactly equal to `"admin"` (lowercase, case-sensitive).
     - Claims structure: `{"publicMetadata": {"role": "admin"}}` (treat missing keys as non-admin).

Testing:
- Add `backend/tests/test_auth_clerk.py`:
  - Unit tests for `extract_bearer_token` using a minimal Request object (or build a FastAPI app route that passes Request into the function).
  - Unit tests for `is_admin_claims` covering:
    - missing keys
    - wrong role
    - correct role
    - role with different casing (should be False)

Constraints:
- Do not add token verification yet.
- Do not modify `backend/main.py` routes yet.

Finish:
- Run `pytest -q` and ensure all tests pass.
```

---

## Prompt 2 — PR 2: Token verification + `admin_required` dependency (no routes wired yet)

```text
Goal: Implement Clerk token verification and an `admin_required` FastAPI dependency, but do not apply it to any routes yet.

Implement:
1) In `backend/auth_clerk.py`, add:
   - `verify_clerk_token(token: str) -> dict`
     - Verifies JWT signature using Clerk JWKS.
     - Validate standard fields minimally (exp, iat); reject expired tokens.
     - Configuration:
       - JWKS URL from env `CLERK_JWKS_URL` (required in production).
       - Accept optional env `CLERK_ISSUER` to validate `iss` if provided.
     - Return decoded claims dict on success; raise a custom exception on failure.

2) Add `admin_required` dependency in `backend/auth_clerk.py`:
   - Reads bearer token from request
   - Verifies token
   - Checks `is_admin_claims`
   - Raises HTTPException:
     - 401 if missing/invalid token
     - 403 if valid token but not admin
   - Returns claims dict for downstream usage

Testing:
- Add tests that mock JWKS fetch:
  - Use a local JWKS fixture and monkeypatch the JWKS retrieval inside `verify_clerk_token`.
  - Write tests for:
    - Missing token -> 401 via `admin_required`
    - Invalid token -> 401
    - Valid token non-admin -> 403
    - Valid token admin -> returns claims

Notes:
- Choose a standard Python JWT library if not present; add to `backend/requirements.txt` if needed.
- Keep tests deterministic; do not call real Clerk network endpoints.

Finish:
- Run `pytest -q` and ensure all tests pass.
```

---

## Prompt 3 — PR 3: Protect one endpoint end-to-end (`GET /api/admin/export-data`)

```text
Goal: Wire the new admin dependency into exactly one backend endpoint end-to-end.

Implement:
1) Update `backend/main.py`:
   - Add dependency injection for `admin_required` on `GET /api/admin/export-data`.
   - Ensure this endpoint returns 401/403 appropriately.

Testing:
- Add/extend tests using FastAPI TestClient:
  - Request without Authorization -> 401
  - Request with valid non-admin token -> 403
  - Request with valid admin token -> 200
  - Mock `verify_clerk_token` to return claims rather than verifying real JWTs in integration tests.

Constraints:
- Do not protect any other endpoints yet.
- Keep diffs small.

Finish:
- Run `pytest -q` and ensure all tests pass.
```

---

## Prompt 4 — PR 4: Protect remaining admin endpoints (no behavior changes)

```text
Goal: Apply backend admin enforcement to all admin-only endpoints, without changing their existing behavior.

Implement:
1) In `backend/main.py`, apply `admin_required` to:
   - `POST /api/upload`
   - `POST /api/ingest`
   - `POST /api/delete-pdf`
   - `POST /api/set-display-name`
   - `POST /api/reorder-files`
   - (keep `GET /api/admin/export-data` protected)

2) Remove or leave `_require_admin` but ensure it is no longer relied upon for security.

Testing:
- Add at least one integration test per route category:
  - One POST endpoint protected (401/403/200)
  - Ensure existing tests still pass

Finish:
- Run `pytest -q` and ensure all tests pass.
```

---

## Prompt 5 — PR 5: Add S3 job store library + unit tests

```text
Goal: Add a backend library to manage job JSON objects in S3 under `jobs/<job_id>.json`.

Implement:
1) Create `backend/job_store.py`:
   - A `Job` typed structure (dataclass or Pydantic model) matching the spec’s JSON fields:
     - job_id, created_at, updated_at, requested_by, input, output, state, attempt, max_attempts, progress, error
   - Functions:
     - `now_iso() -> str`
     - `create_job(s3_client, bucket: str, job: Job) -> Job` (sets job_id if missing, timestamps, writes JSON)
     - `get_job(s3_client, bucket: str, job_id: str) -> Job`
     - `put_job(s3_client, bucket: str, job: Job) -> None` (updates updated_at, writes JSON)
     - `list_recent_jobs(s3_client, bucket: str, limit: int = 5) -> list[Job]`

2) Ensure keys use prefix `jobs/`.

Testing:
- Add `backend/tests/test_job_store.py` using botocore Stubber or moto to avoid real AWS:
  - Create/get/update roundtrip
  - List recent returns expected order (use different updated_at values)
  - Missing job raises a clean exception (to be mapped to 404 later)

Constraints:
- Do not add any endpoints yet.

Finish:
- Run `pytest -q` and ensure all tests pass.
```

---

## Prompt 6 — PR 6: Add job status endpoints (admin-only)

```text
Goal: Add admin-only APIs to retrieve job status and list recent jobs.

Implement in `backend/main.py`:
1) `GET /api/upload-status/{job_id}`
   - Protected by `admin_required`
   - Reads from S3 via `job_store.get_job(...)`
   - Returns the job JSON (dict) directly
   - Returns 404 if job not found

2) `GET /api/upload-jobs/recent?limit=5`
   - Protected by `admin_required`
   - Uses `job_store.list_recent_jobs(...)`
   - Enforces max reasonable limit (e.g., <= 20) even if caller requests higher

Testing:
- Add integration tests:
  - 401/403 enforcement (mock `verify_clerk_token`)
  - 404 path for unknown job_id
  - recent endpoint returns expected number and order

Finish:
- Run `pytest -q` and ensure all tests pass.
```

---

## Prompt 7 — PR 7: Update `/api/upload` to accept multipart `file` and create jobs (PDF path only)

```text
Goal: Make `/api/upload` create a job and store PDFs in S3 `pdfs/`, but only fully support PDF uploads for now.

Implement:
1) Update `POST /api/upload` in `backend/main.py`:
   - Protected by `admin_required`
   - Accept multipart field name `file` (keep backwards compatibility: if existing clients send `pdf`, accept it too)
   - Validate extension:
     - If `.pdf`: proceed
     - If `.ppt`/`.pptx`: return 501 Not Implemented (for now) with clear JSON message
     - Else: 400
   - Create `job_id`, write `jobs/<job_id>.json` with state `queued`
   - Upload the PDF bytes to S3 key `pdfs/<output_pdf_filename>`
     - output_pdf_filename is the uploaded base name + `.pdf` (for PDF, same name)
   - Update job status to `done` with updated timestamps and output.s3_key
   - Return `{status:"success", job_id, output_pdf_filename}`

Testing:
- Integration tests for PDF path:
  - Upload returns job_id
  - Job object exists and ends in state `done`
  - S3 put called with key prefix `pdfs/`
- Test `.pptx` returns 501 with message
- Use S3 stub/moto; do not hit real AWS.

Finish:
- Run `pytest -q` and ensure all tests pass.
```

---

## Prompt 8 — PR 8: Single-job lock enforcement on upload

```text
Goal: Prevent concurrent indexing/jobs by rejecting uploads while indexing or another active job exists.

Implement:
1) In backend web service, add an `is_busy_for_upload()` helper that returns True if:
   - `rag_engine.IS_INDEXING` is True
   - OR there exists a job in S3 with state in {queued, converting, uploading_pdf, ingesting} that is recent (e.g., updated within last 24h)
     - Keep scanning bounded: only look at a limited number of recent jobs (e.g., 20) to avoid expensive S3 list.

2) In `POST /api/upload`, if busy:
   - Return 409 with JSON:
     - status="error", code="INDEXING_IN_PROGRESS", message="Indexing in progress—try again later."

Testing:
- Unit test `is_busy_for_upload` using stubbed job_store list results.
- Integration test upload returns 409 when:
  - `IS_INDEXING` true, and when
  - Active job exists in S3

Finish:
- Run `pytest -q` and ensure all tests pass.
```

---

## Prompt 9 — PR 9: Add incremental ingest endpoint (`POST /api/ingest-file`) with delete-then-index ordering

```text
Goal: Implement `POST /api/ingest-file` that replaces embeddings for exactly one PDF, admin-only, with correct ordering and locking.

Implement:
1) Add a request model (Pydantic) for `{ "filename": "<name>.pdf" }`.
2) Add `POST /api/ingest-file` in `backend/main.py`:
   - Protected by `admin_required`
   - Enforce lock:
     - If `rag_engine.IS_INDEXING` true, return 409 (same code/message as upload)
     - Set `rag_engine.IS_INDEXING=True` for duration in try/finally
   - Ensure local file exists:
     - If S3 enabled, download `pdfs/<filename>` into `./data/uploaded_pdfs/<filename>` if missing.
   - Call `rag_engine.delete_pdf_from_database(filename)` BEFORE indexing.
   - Index only that file:
     - Use a temp directory containing only that PDF and load via `SimpleDirectoryReader` with `PDFReader`.
     - Insert into existing Chroma-backed index without rebuilding everything.
   - Always set `IS_INDEXING=False` in finally.
   - Return JSON with status and document_count/chunk_count as appropriate.

Testing:
- Unit test ordering:
  - Mock `delete_pdf_from_database` and the indexing function and assert delete called before index.
- Integration test:
  - When `IS_INDEXING` true, endpoint returns 409 and does not index.

Finish:
- Run `pytest -q` and ensure all tests pass.
```

---

## Prompt 10 — PR 10: Integration test for ingest-file “no duplicates”

```text
Goal: Add a robust test that proves ingesting the same file twice does not create duplicate embeddings.

Implement:
1) Add a small PDF fixture under `backend/tests/fixtures/` (keep file small).
2) In an integration test:
   - Ensure Chroma test environment uses an isolated temporary directory (don’t touch real `./data/chroma_db`).
   - Call ingest-file twice for the same filename (using local file path / mocked S3 download).
   - Assert the collection count for that file is the same after the second run (no increase).

Notes:
- If the current rag_engine architecture makes this hard, refactor minimally:
  - Allow overriding CHROMA_PATH and PDF_UPLOAD_DIR during tests via env vars.
  - Keep production defaults unchanged.

Finish:
- Run `pytest -q` and ensure all tests pass.
```

---

## Prompt 11 — PR 11: Worker service skeleton + secret auth

```text
Goal: Add a new worker service codebase with a secret-protected start endpoint. No conversion yet.

Implement:
1) Create `worker/` package with:
   - `worker/app.py` (FastAPI app)
   - `worker/main.py` or entrypoint
2) Implement `POST /worker/start-job`:
   - Requires header `X-Worker-Secret` matching env `WORKER_SHARED_SECRET`
   - Accepts JSON `{ "job_id": "..." }`
   - For now, just:
     - Loads the job from S3
     - Writes job state to `converting` with progress message "Worker accepted job"
     - Returns 202

Testing:
- Add `worker/tests/test_worker_auth.py`:
  - missing/invalid secret -> 401
  - valid secret -> 202
- Mock S3 calls in worker tests; do not hit AWS.

Finish:
- Ensure worker tests pass (`pytest -q` for worker suite).
```

---

## Prompt 12 — PR 12: Worker Docker image adds LibreOffice + fonts (smoke test)

```text
Goal: Add Docker/build artifacts for the worker that install LibreOffice headless and fonts, and a smoke test.

Implement:
1) Add `worker/Dockerfile` that:
   - Installs libreoffice (headless)
   - Installs font packages (Noto + Liberation minimum)
   - Installs Python deps for worker
2) Add a small smoke test script or test that can run in CI:
   - Confirms `soffice --version` works (skip if not running in container)

Notes:
- Keep the web service Dockerfile unchanged.
- Don’t add conversion logic yet.
```

---

## Prompt 13 — PR 13: Worker conversion + S3 upload (conversion-only pipeline)

```text
Goal: Implement PPT/PPTX conversion in the worker with 3 retries and correct job state transitions, but do not call ingest yet.

Implement in worker:
1) Conversion utility:
   - `convert_ppt_to_pdf(input_path, output_dir, export_hidden=True) -> output_pdf_path`
   - Build LibreOffice command using impress export options to include hidden slides:
     - `ExportHiddenSlides=true`
   - Validate output PDF exists and is non-empty.
   - Add a timeout to prevent hangs.

2) Job runner for start-job:
   - Read job JSON from `jobs/<job_id>.json`
   - For `.ppt`/`.pptx` jobs:
     - Update state: `converting` (attempt 1..3) with progress
     - Download `incoming/<job_id>/...` from S3
     - Convert to PDF
     - Update state: `uploading_pdf`
     - Upload PDF to `pdfs/<pdf_filename>` with content-type `application/pdf`
     - Delete incoming object
     - Update state: `done`
   - On errors:
     - Retry up to 3 times with exponential backoff
     - After final attempt: set state `failed` and populate `error.message` and `error.retryable`

Testing:
- Unit tests for command construction (export hidden slides option present).
- Unit test for retry behavior + state transitions sequence.
- Mock S3 and subprocess execution; do not require LibreOffice in unit tests.

Finish:
- Run worker tests and ensure they pass.
```

---

## Prompt 14 — PR 14: Internal ingest endpoint + worker call (full pipeline)

```text
Goal: Wire worker → web ingest for converted PDFs using a secret-protected internal endpoint, and ensure maintenance lock works via `IS_INDEXING`.

Implement on web service:
1) Add `POST /internal/ingest-file`:
   - Protected by header `X-Worker-Secret` matching env `WORKER_SHARED_SECRET`
   - Accepts `{ "filename": "<name>.pdf" }`
   - Calls the same implementation as `/api/ingest-file` (reuse code), but does NOT require Clerk.

2) Add tests:
   - Missing/invalid secret -> 401
   - Valid secret -> 200 and triggers ingest logic (mock heavy bits)

Implement on worker:
1) After uploading the PDF, update job state `ingesting`, then call web `/internal/ingest-file` with the secret header.
2) Only mark job `done` after ingest succeeds; otherwise mark failed with error.

Finish:
- Run backend + worker tests.
- Ensure `/api/status` still reflects indexing via `IS_INDEXING` during ingest.
```

---

## Prompt 15 — PR 15: Cleanup routine + 7-day retention

```text
Goal: Implement deletion of old `jobs/` status objects and orphaned `incoming/` objects older than 7 days.

Implement in worker:
1) Add `DELETE /worker/cleanup` (or `POST`) protected by `X-Worker-Secret`.
2) Cleanup behavior:
   - List `jobs/` objects; parse each job JSON updated_at; delete if older than 7 days.
   - List `incoming/` prefix; delete objects older than 7 days (by LastModified).
   - Return summary counts.

Testing:
- Unit tests for cutoff logic (date parsing) with mocked S3 list results.
- Auth tests for cleanup endpoint.

Finish:
- Ensure tests pass.
```

---

## Prompt 16 — Final wiring pass: enable PPT/PPTX path in `/api/upload`

```text
Goal: Flip the final switch: `/api/upload` should fully accept `.ppt`/`.pptx` and trigger the worker.

Implement on web service:
1) Update `POST /api/upload`:
   - For `.ppt`/`.pptx`:
     - Create job in `jobs/` state `queued`
     - Upload original to `incoming/<job_id>/<originalFilename>`
     - Trigger worker by HTTP:
       - `POST <WORKER_BASE_URL>/worker/start-job` with header `X-Worker-Secret`
       - Body `{job_id}`
     - Return `{status:"success", job_id, output_pdf_filename}`
   - For `.pdf`, keep existing job model behavior (upload to `pdfs/` and mark done), OR optionally also trigger ingest-file automatically if desired.

2) Add integration tests (mock worker HTTP call):
   - `.pptx` upload stores incoming object, creates queued job, calls worker start endpoint, returns job_id.
   - Ensure lock returns 409 when busy.

Finish:
- Run backend tests.
- Confirm no dead code remains (all modules used).
```

