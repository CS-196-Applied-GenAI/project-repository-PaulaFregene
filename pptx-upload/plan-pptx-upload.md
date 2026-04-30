# Backend build plan: PPT/PPTX upload + worker conversion + incremental ingest + Clerk admin enforcement

This plan implements the backend described in `documentations/pptx-upload/spec-pptx-upload.md`, with a strong focus on safety, security, and incremental delivery.

## Guiding principles

- **Ship in thin vertical slices**: each chunk should be deployable and provide user-visible value or measurable safety.
- **Prefer deterministic, testable code paths**: isolate S3 I/O, Clerk verification, conversion invocation, and ingest logic behind small modules with unit tests.
- **Fail safely**: never leave the system in an ambiguous state; job status must end as `done` or `failed` with actionable error text.
- **No new external paid infra**: use **S3** for job state and temp artifacts; use a **Railway worker** as a second service.
- **Single active job**: enforce one job at a time to avoid index corruption/races.

## Target backend architecture (final state)

### Web service (FastAPI, existing)

- **Admin auth**: verify Clerk token on backend; enforce `publicMetadata.role === "admin"` for admin endpoints.
- **Unified upload endpoint**: `POST /api/upload` accepting `.pdf`, `.ppt`, `.pptx`, returning `job_id`.
- **Job APIs**:
  - `GET /api/upload-status/{job_id}` (admin-only)
  - `GET /api/upload-jobs/recent?limit=5` (admin-only)
- **Incremental ingest**: `POST /api/ingest-file` (admin-only) to replace embeddings for exactly one PDF.
- **Existing**: `/api/ingest` remains for bulk ingest; `/api/status` remains for maintenance popup.

### Worker service (new Railway service)

- Minimal HTTP app with:
  - `POST /worker/start-job` protected by `X-Worker-Secret: WORKER_SHARED_SECRET`
  - Optional daily cleanup routine (can be triggered by Railway cron or a scheduled loop)
- Responsibilities:
  - Convert PPT/PPTX to PDF using **LibreOffice headless**, with hidden slides exported.
  - Upload resulting PDF to `pdfs/<name>.pdf`.
  - Delete `incoming/<job_id>/...`.
  - Call web service `POST /api/ingest-file` to index the produced PDF (replace embeddings).
  - Update S3 job status object throughout the pipeline.

### S3 key layout (final)

- Canonical PDFs: `pdfs/<filename>.pdf`
- Temp incoming originals: `incoming/<job_id>/<originalFilename>.(ppt|pptx)`
- Job status JSON: `jobs/<job_id>.json` (retain 7 days)

## Detailed build blueprint (step-by-step)

### 0) Preflight: baseline test harness and configuration sanity

- Ensure backend test suite can run locally (`pytest`).
- Add minimal integration-test helpers for:
  - S3 client mocking (e.g., `moto` or botocore stubber)
  - HTTP client for FastAPI (TestClient)
- Define environment variables (documented in repo + Railway):
  - Web: `CLERK_JWKS_URL` or equivalent Clerk verification config, `CLERK_SECRET_KEY` (depending on chosen method), `USE_S3=true`, AWS creds, bucket, region
  - Worker: AWS creds, bucket, region, `WORKER_SHARED_SECRET`, web service base URL (internal)

### 1) Security foundation: backend Clerk token verification + admin enforcement

- Implement backend auth module:
  - `verify_clerk_token(Authorization: Bearer ...) -> user_claims`
  - `require_admin(request) -> claims` raising 401/403
  - Enforce `publicMetadata.role == "admin"` (lowercase, case-sensitive)
- Wire `require_admin` into all admin endpoints:
  - upload, ingest, delete, reorder, set-display-name, export-data, job status endpoints, ingest-file
- Add tests:
  - Missing token → 401
  - Invalid token → 401
  - Valid token but non-admin → 403
  - Valid admin token → 200

### 2) S3 job store module (pure backend library)

- Implement a small library for job state in S3:
  - `create_job(job: Job) -> job_id`
  - `get_job(job_id) -> Job`
  - `put_job(job) -> None` (atomic-ish overwrite)
  - `list_recent_jobs(limit=5) -> list[Job]`
  - helpers: `now_iso()`, `job_state_transition(...)`
- JSON schema matches spec (state, attempt/max_attempts, progress, error, updated_at).
- Add tests using stubbed S3:
  - Create/get/update
  - List recent ordering
  - Schema validation / required fields

### 3) “Single active job” lock

- Implement lock check in web service:
  - Reject upload if `rag_engine.IS_INDEXING` true
  - Additionally reject if there exists any job in `jobs/` with state in `{queued, converting, uploading_pdf, ingesting}` updated recently (or store a dedicated `jobs/active.json` lock object)
- Tests:
  - When lock set → upload returns 409 with `INDEXING_IN_PROGRESS`

### 4) Unified upload endpoint (web service): accept PDF/PPT/PPTX and create job

- Update `POST /api/upload` to:
  - Accept multipart field `file`
  - Validate extension: `.pdf`, `.ppt`, `.pptx`
  - Determine output PDF filename: base name + `.pdf`
  - Create `jobs/<job_id>.json` with state `queued` and `output.pdf_filename`
  - Store artifact:
    - If PDF: upload to `pdfs/<output_pdf_filename>` directly
    - If PPT/PPTX: upload to `incoming/<job_id>/<originalFilename>`
  - Trigger worker via HTTP (next step)
  - Respond with `{job_id, output_pdf_filename}`
- Add endpoint tests:
  - PDF upload returns job_id and uploads to correct S3 key
  - PPTX upload returns job_id and uploads to incoming key
  - Invalid file extension returns 400

### 5) Job status APIs (web service)

- Implement:
  - `GET /api/upload-status/{job_id}` reads `jobs/<job_id>.json`
  - `GET /api/upload-jobs/recent?limit=5` lists recent jobs
- Add tests:
  - Admin-only enforcement works
  - 404 for unknown job_id

### 6) Incremental ingest endpoint (web service): replace embeddings for a single PDF

- Implement `POST /api/ingest-file`:
  - Admin-only
  - Enforce `IS_INDEXING` lock; set `IS_INDEXING=True` for duration
  - Ensure local copy exists (download from `pdfs/<filename>` if needed)
  - Call `delete_pdf_from_database(filename)` to remove embeddings for that file
  - Index only that file and insert into Chroma without duplicates
  - Always clear `IS_INDEXING` in `finally`
- Tests:
  - Calls delete-by-file first, then indexes
  - Subsequent ingest-file does not increase chunk count for same content (no duplicates) (use a small fixture PDF)
  - Lock prevents concurrent ingest

### 7) Worker service: skeleton + shared secret auth

- Create a new `worker/` package (or `backend_worker/`) with:
  - Minimal FastAPI app exposing `POST /worker/start-job`
  - Reject if `X-Worker-Secret` mismatched
- Basic test:
  - Missing/invalid secret → 401
  - Valid secret → 202/200 and starts processing (can be mocked)

### 8) Worker: convert PPT/PPTX to PDF with LibreOffice

- Add Dockerfile/build changes for worker image:
  - Install LibreOffice headless
  - Install font packages (Noto + Liberation at minimum)
- Implement conversion function:
  - Download incoming object from S3 to temp dir
  - Run LibreOffice convert-to pdf with export options:
    - Ensure hidden slides are exported (`ExportHiddenSlides=true`)
  - Validate output exists and is non-empty
- Retry policy:
  - Up to 3 attempts with exponential backoff
- Tests:
  - Unit test command construction
  - Integration test in container (optional) with a small PPTX fixture (only if feasible in CI)

### 9) Worker: end-to-end job runner

- Implement job runner that:
  - Reads `jobs/<job_id>.json`
  - Writes state transitions: `converting` → `uploading_pdf` → `ingesting` → `done` (or `failed`)
  - Uploads PDF to `pdfs/<pdf_filename>` with correct content-type
  - Deletes incoming object
  - Calls web service `POST /api/ingest-file` (admin-auth not needed if this endpoint is internal; see “Worker-to-web auth” below)
  - On failures: populate `error` in job JSON and set `failed`
- Tests:
  - State transitions happen in order
  - Incoming is deleted on success
  - Failed conversion sets failed state and error message

### 10) Worker ↔ web authentication (internal call)

Two acceptable approaches (pick one in implementation):

- **Option A (recommended)**: create a dedicated internal endpoint `POST /internal/ingest-file` protected by `WORKER_SHARED_SECRET`, not Clerk.
  - Worker calls this internal endpoint.
  - Web service keeps public admin endpoints Clerk-protected.
- **Option B**: worker holds an admin Clerk token (not ideal operationally).

Add tests for the internal secret-protected endpoint.

### 11) Cleanup: delete old jobs and orphaned incoming objects

- Implement cleanup routine in worker:
  - Delete `jobs/*.json` older than 7 days
  - Delete `incoming/<jobId>/...` older than 7 days
- Can be:
  - A periodic loop in worker (daily), or
  - A separate Railway cron trigger hitting `/worker/cleanup` protected by secret
- Tests:
  - Date parsing and cutoff logic

### 12) Hardening and operational safeguards

- Add size limits:
  - Max upload bytes for PPT/PPTX (configurable env)
- Timeouts:
  - Worker conversion timeout (kill LO process if it hangs)
- Logging:
  - Include `job_id` in all logs
- Error messages:
  - User-safe error text in job status (no secrets)

## Chunking: small iterative increments

Below is the same plan decomposed into “chunks that build on each other”, then broken down again into “small steps” suitable for safe implementation with testing.

### Round 1: Iterative chunks (deployable milestones)

#### Chunk 1: Secure admin endpoints with Clerk verification
- Deliverable: backend truly enforces admin role, not UI-only.
- Tests: auth unit/integration tests.

#### Chunk 2: S3 job store + job status APIs
- Deliverable: create/read/list job objects via admin-only endpoints (no worker yet).
- Tests: S3 stubs, endpoint tests.

#### Chunk 3: Unified upload endpoint (PDF only) using job model
- Deliverable: PDF upload returns job_id; job status becomes `done` immediately; no PPTX yet.
- Tests: upload path + job updates + lock behavior.

#### Chunk 4: Incremental ingest endpoint (replace embeddings for one PDF)
- Deliverable: `POST /api/ingest-file` works and avoids duplicates.
- Tests: fixture PDF, chunk-count stability.

#### Chunk 5: Worker skeleton + secret-protected trigger
- Deliverable: worker accepts start-job calls and updates job state (mock conversion).
- Tests: secret auth tests.

#### Chunk 6: PPT/PPTX conversion in worker + upload PDF to S3
- Deliverable: PPTX job converts and writes `pdfs/<name>.pdf`, updates job state; no ingest call yet.
- Tests: conversion command tests + job state.

#### Chunk 7: Worker triggers ingest-file and maintenance lock works
- Deliverable: full pipeline: upload → worker conversion → ingest-file → done; students blocked during ingest.
- Tests: end-to-end in staging; unit tests for worker-to-web call.

#### Chunk 8: Cleanup + hardening
- Deliverable: 7-day retention enforcement, timeouts, limits, better logs.
- Tests: cleanup logic tests.

### Round 2: Break each chunk into small steps (right-sized tasks)

#### Chunk 1 steps (Clerk admin enforcement)
1. Add `backend/auth_clerk.py` (or similar) with token extraction + verification stub and role check.
2. Add tests for role check function using mock claims.
3. Implement actual token verification (JWKS fetch + signature verify) and add tests with a mocked JWKS response.
4. Replace `_require_admin` usage in `backend/main.py` with `require_admin_clerk(request)`.
5. Add endpoint tests for one representative endpoint (e.g., `/api/admin/export-data`), then replicate for others.

#### Chunk 2 steps (S3 job store + APIs)
1. Add `backend/job_store.py` with typed `Job` model and `get/put/create` functions.
2. Add S3-stubbed unit tests for job store.
3. Add `GET /api/upload-status/{job_id}` endpoint (admin-only) with tests.
4. Add `GET /api/upload-jobs/recent` endpoint (admin-only) with tests.

#### Chunk 3 steps (Unified upload endpoint with jobs, PDF-only first)
1. Update `POST /api/upload` to accept multipart `file` and validate `.pdf` only (temporary).
2. Create job in S3 with state `queued` and then immediately set `done` after upload succeeds (PDF path).
3. Enforce single-job lock on upload (simple: reject if `IS_INDEXING` true).
4. Add tests for PDF upload path + job status written.
5. Expand validation to accept `.ppt/.pptx` but still reject with “not implemented” until worker lands (feature flag).

#### Chunk 4 steps (Incremental ingest-file)
1. Add endpoint `POST /api/ingest-file` with request model validation.
2. Add helper to ensure local PDF exists (download from S3 if missing).
3. Call `delete_pdf_from_database(filename)` and add a unit test around that call order (mock).
4. Implement “index single file” logic using a temp dir containing only that file.
5. Add integration test with a small PDF fixture verifying no duplicates on repeated ingest-file.

#### Chunk 5 steps (Worker skeleton)
1. Create `worker` app package + minimal server.
2. Add secret check middleware/dependency.
3. Add `POST /worker/start-job` that reads job_id and writes job state `converting` (no-op).
4. Add unit tests for secret enforcement and job state write.

#### Chunk 6 steps (Conversion + PDF upload)
1. Update worker Dockerfile to install LibreOffice + fonts; verify container starts.
2. Implement `convert_ppt_to_pdf(input_path, out_dir, export_hidden=True)`.
3. Implement S3 download of incoming ppt/pptx and S3 upload of produced pdf.
4. Update job states: `converting` → `uploading_pdf` → `done` for conversion-only path.
5. Add retry loop (3 attempts) with status updates `attempt`.
6. Add tests for:
   - Command assembly
   - Retry logic state transitions

#### Chunk 7 steps (Worker triggers ingest-file)
1. Add internal secret-protected endpoint on web service: `POST /internal/ingest-file`.
2. Worker calls `/internal/ingest-file` after uploading PDF and sets job state to `ingesting`.
3. Ensure web sets `IS_INDEXING` for duration so student maintenance popup works.
4. Add tests for internal endpoint secret enforcement.
5. Add staging end-to-end test procedure (manual but scripted) for: upload pptx → status transitions → pdf appears → citations work.

#### Chunk 8 steps (Cleanup + hardening)
1. Add cleanup routine that deletes job objects older than 7 days.
2. Add orphan cleanup for `incoming/` older than 7 days.
3. Add worker conversion timeout + process kill.
4. Add upload size limits (env-configured) for ppt/pptx.
5. Add structured logging with `job_id`.

### Round 3: One more decomposition (micro-steps suitable for safe PRs)

These are “PR-sized” steps that are small enough to implement + test safely.

#### PR 1: Auth scaffolding (no endpoint changes yet)
- Add `backend/auth_clerk.py` with:
  - `extract_bearer_token(request)`
  - `is_admin_claims(claims)` (checks `publicMetadata.role == "admin"`)
- Unit tests for token extraction + role check.

#### PR 2: Clerk token verification + guard dependency
- Implement `verify_token(token) -> claims` with mocked JWKS in tests.
- Add `admin_required` FastAPI dependency that:
  - verifies token
  - checks admin role
- Add tests for dependency behavior (401/403).

#### PR 3: Protect one endpoint end-to-end
- Apply `admin_required` to `GET /api/admin/export-data`.
- Add integration tests confirming protection.

#### PR 4: Protect remaining admin endpoints
- Apply `admin_required` to upload/ingest/delete/reorder/set-display-name.
- Add at least one test per route category (GET vs POST).

#### PR 5: Add job store library + unit tests
- `backend/job_store.py` with S3 stubs.
- Tests for create/get/update/list_recent.

#### PR 6: Add job status endpoints (admin-only)
- `GET /api/upload-status/{job_id}`
- `GET /api/upload-jobs/recent`
- Endpoint tests.

#### PR 7: Convert `/api/upload` to accept multipart `file` and create jobs (PDF path)
- Implement PDF upload via jobs:
  - upload to `pdfs/`
  - job state `queued` → `done`
- Tests for upload and job object written.

#### PR 8: Add single-job lock enforcement
- Implement “active job” detection (either dedicated lock object or scan recent jobs).
- Upload returns 409 when locked.
- Tests for lock logic.

#### PR 9: Add incremental ingest-file endpoint (admin-only)
- `POST /api/ingest-file` (and optional `/internal/ingest-file` stubbed)
- Unit tests for “delete then index” call order using mocks.

#### PR 10: Integration test for ingest-file “no duplicates”
- Add a small PDF fixture.
- Run ingest-file twice and assert Chroma count unchanged for that file.

#### PR 11: Worker service skeleton + secret auth
- New worker app with `POST /worker/start-job`.
- Worker writes job state updates (no conversion).
- Tests for secret enforcement.

#### PR 12: Worker Docker image adds LibreOffice + fonts
- Update worker Dockerfile.
- Smoke test: container can run `soffice --version`.

#### PR 13: Worker conversion + S3 upload (conversion-only pipeline)
- Implement download incoming → convert → upload pdfs/ → job done.
- Retry 3 times.
- Tests for retries and job state.

#### PR 14: Internal ingest endpoint + worker call
- Implement `/internal/ingest-file` protected by `WORKER_SHARED_SECRET`.
- Worker calls it and updates job state `ingesting`.
- Tests for internal secret auth.

#### PR 15: Cleanup routine + retention
- Implement 7-day cleanup for `jobs/` and `incoming/`.
- Unit tests for age cutoff.

## Testing strategy (minimum viable)

### Unit tests

- Auth:
  - role check (`publicMetadata.role`)
  - token extraction
- Job store:
  - create/get/update/list_recent
- Worker logic:
  - retry behavior
  - state transition order
  - secret validation

### Integration tests

- Protected endpoints require token (401/403 paths).
- `/api/upload` creates job and writes correct S3 keys (stubbed S3).
- `/api/ingest-file` called with fixture PDF does not create duplicates across two runs.

### Manual staging test checklist (repeatable)

1. Upload a small PDF → job done → file appears in `/api/files`.
2. Upload a small PPTX → job transitions `converting → uploading_pdf → ingesting → done`.
3. Confirm `Lecture.pptx` becomes `Lecture.pdf` and overwrites correctly.
4. During ingesting, students see maintenance popup and cannot query.
5. After done, query returns citations from the new PDF.

## Definition of “right-sized”

- A step/PR should:
  - Touch a small number of modules
  - Add or update tests
  - Be deployable without requiring all future steps
  - Avoid mixing infrastructure (worker/Docker) and application logic changes when possible

