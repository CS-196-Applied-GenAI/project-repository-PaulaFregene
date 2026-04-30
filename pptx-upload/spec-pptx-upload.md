# Spec: PPT/PPTX Upload → Unhide Slides → Convert to PDF → Auto-Ingest (Railway + S3)

## Goals

- Add support for uploading **PowerPoint** files (`.pptx` and `.ppt`) in addition to PDFs.
- Preserve current production behavior where **S3 is the source of truth** (`USE_S3=true`), and the app lists/serves course files from S3-backed PDFs.
- For PPT/PPTX uploads, generate a **PDF** that includes **all slides**, including slides marked **Hidden** in PowerPoint.
- Store **only the converted PDF** long-term (do **not** keep PPT/PPTX permanently).
- Keep the website responsive and safe during indexing by:
  - Running conversion in a **separate Railway worker service**.
  - Using the existing **maintenance lock** mechanism (students blocked during ingest).
- Make admin-only endpoints **secure** by enforcing **Clerk token verification + admin role checks** on the backend.

## Non-goals

- Converting PPT/PPTX in the browser.
- Maintaining a full version history for uploads (we overwrite by filename).
- Storing PPT/PPTX artifacts long-term.

## Current System (as-is)

- Backend: FastAPI (`backend/main.py`)
  - `POST /api/upload`: saves PDF to `./data/uploaded_pdfs/` and uploads to S3 under `pdfs/<filename>.pdf` (when `USE_S3=true`).
  - `POST /api/ingest`: triggers `rag_engine.ingest_pdfs()` in background; sets `rag_engine.IS_INDEXING` during indexing.
  - `GET /api/status`: returns `{"is_indexing": rag_engine.IS_INDEXING}`.
  - `GET /api/files`: lists S3 PDFs (or local PDFs), plus display names and presigned URLs.
- Frontend:
  - `/chat` page (`frontend/app/chat/ChatPage.tsx`) has an **admin-only** upload control and **auto-calls ingest** after upload.
  - `/admin` course materials page (`frontend/app/admin/UploadPDF.tsx`) uploads but **does not auto-ingest**.
  - A **maintenance popup** blocks the chat UI when `is_indexing=true` (students can’t query during ingestion).

## Desired Behavior (to-be)

### File types

- Accept **PDF**, **PPTX**, and **PPT** uploads in the admin upload UI.
- For PPT/PPTX:
  - Convert to PDF server-side using **LibreOffice headless**.
  - Ensure **hidden slides are included** in the produced PDF.
  - PDF output filename is **base name + `.pdf`**:
    - `Lecture 3.pptx` → `Lecture 3.pdf`
    - `Lecture 3.ppt` → `Lecture 3.pdf`
  - If `Lecture 3.pdf` already exists, **overwrite** it.
  - **Replace embeddings** for that PDF (delete old chunks for `file_name == "Lecture 3.pdf"` then re-index that file).

### Auto-ingest UI uploads

- Uploading via UI should auto-trigger **incremental ingest of the new/updated PDF** (not “ingest all”).
- Bulk uploads performed directly in S3 remain supported via the existing manual “Ingest All PDFs”.

### Indexing lock / student experience

- Students should see the existing **maintenance popup** during ingestion (no detailed job status).
- Admin should be able to use admin pages while ingestion runs, but admin actions that mutate content must be disabled:
  - During ingestion (indexing in progress): **disable Upload and Delete** in admin UIs and show a clear message.
  - Admin should not use the AI bot during indexing (already enforced by the maintenance popup on `/chat`).

### Job-based processing

- UI uploads return quickly with a **job ID**, then the UI polls for status.
- Worker executes conversion and ingest pipeline.
- Job status is stored in S3 (`jobs/<jobId>.json`) and retained for **7 days**.
- Admin UI shows:
  - Inline status next to the upload control.
  - A “Recent jobs” panel showing the **last 5** jobs.
- Admin UI polls job status **every 5 seconds** and stops after done/failed.

### Trigger model

- Use **backend → worker HTTP trigger** (no S3 polling) because uploads are infrequent.
- Worker “start job” endpoint is protected with a **shared secret** header.
- Enforce **single active job** at a time; reject new uploads while a job is running (“Indexing in progress—try again later”).

## Architecture

### Services (Railway)

- **Web service**: existing FastAPI backend.
- **Worker service**: new Railway service built from the repo (new Dockerfile or target), running:
  - A small FastAPI app (or a minimal HTTP server) with an authenticated endpoint to start a job.
  - LibreOffice headless installed.
  - Font packages installed (see Fonts section).

### Storage (S3)

- PDFs (canonical, visible to users):
  - `pdfs/<filename>.pdf`
- Temporary incoming originals (deleted after conversion):
  - `incoming/<jobId>/<originalFilename>.pptx` (or `.ppt`)
- Job status objects:
  - `jobs/<jobId>.json`

## Security: Backend Clerk verification (Admin enforcement)

### Admin definition

- A user is an admin if their Clerk `publicMetadata.role` is **exactly** `"admin"` (lowercase, case-sensitive).

### Required changes

- All admin-only backend endpoints must require:
  - A valid Clerk session token (sent from the frontend as `Authorization: Bearer <token>`).
  - Token verification on the backend using Clerk’s backend verification.
  - Role check: `publicMetadata.role === "admin"`.

### Endpoints to protect

Protect all endpoints that mutate or expose admin-only data:

- `POST /api/upload` (new behavior: accept PDF/PPT/PPTX via unified endpoint)
- `POST /api/ingest` (manual ingest-all)
- `POST /api/delete-pdf`
- `POST /api/set-display-name`
- `POST /api/reorder-files`
- `GET /api/admin/export-data`
- New job endpoints (see below)

### Notes

- UI-only gating is not sufficient; backend must enforce admin.
- Keep CORS settings compatible with the frontend.

## API Design

### 1) Upload (unified): `POST /api/upload`

**Purpose**

- Accept PDF/PPT/PPTX uploads.
- Enforce single active job (if indexing or job running, reject).

**Request**

- `multipart/form-data`
- Field name: **`file`** (standardize across UIs)
- Accepted extensions:
  - `.pdf`, `.pptx`, `.ppt`

**Response (success)**

- HTTP 200
- JSON:
  - `status: "success"`
  - `job_id: string`
  - `output_pdf_filename: string` (e.g., `Lecture 3.pdf`)

**Response (busy)**

- HTTP 409
- JSON:
  - `status: "error"`
  - `code: "INDEXING_IN_PROGRESS"`
  - `message: "Indexing in progress—try again later."`

**Response (validation error)**

- HTTP 400
- JSON:
  - `status: "error"`
  - `message: "Only PDF/PPT/PPTX allowed"`

**Server-side behavior**

1. Verify admin via Clerk token.
2. If `rag_engine.IS_INDEXING` is true or there is an active job running, reject with 409.
3. Create `job_id` (UUID).
4. Write initial job status to `jobs/<job_id>.json`:
   - state `queued`
5. For uploads:
   - If PDF:
     - Upload directly to `pdfs/<output_pdf_filename>`
   - If PPT/PPTX:
     - Upload original to `incoming/<job_id>/<originalFilename>`
6. Trigger worker via HTTP (see “Worker trigger”).
7. Return `{job_id, output_pdf_filename}`.

### 2) Job status: `GET /api/upload-status/{job_id}`

**Purpose**

- Admin UI polls this endpoint to show progress/errors.

**Auth**

- Admin-only via Clerk.

**Response**

- HTTP 200 JSON: contents of the job status object.
- HTTP 404 if job not found.

### 3) Recent jobs: `GET /api/upload-jobs/recent?limit=5`

**Purpose**

- Admin UI displays the last 5 jobs.

**Auth**

- Admin-only via Clerk.

**Behavior**

- List objects under `jobs/` prefix, sort by `updated_at` descending, return up to `limit` (default 5).

### 4) Manual ingest-all (unchanged): `POST /api/ingest`

- Remains available and admin-protected.
- Triggers full sync+ingest of all PDFs.

## Job Status Object (S3 JSON)

Path: `jobs/<jobId>.json`

Example schema:

```json
{
  "job_id": "uuid",
  "created_at": "2026-04-29T03:00:00Z",
  "updated_at": "2026-04-29T03:01:10Z",
  "requested_by": {
    "clerk_user_id": "user_...",
    "email": "optional"
  },
  "input": {
    "original_filename": "Lecture 3.pptx",
    "content_type": "application/vnd.openxmlformats-officedocument.presentationml.presentation",
    "s3_key": "incoming/<jobId>/Lecture 3.pptx"
  },
  "output": {
    "pdf_filename": "Lecture 3.pdf",
    "s3_key": "pdfs/Lecture 3.pdf"
  },
  "state": "queued | converting | uploading_pdf | ingesting | done | failed",
  "attempt": 1,
  "max_attempts": 3,
  "progress": {
    "message": "Converting with LibreOffice…"
  },
  "error": {
    "message": "string",
    "detail": "string",
    "retryable": true
  }
}
```

Retention: delete after **7 days** (see Cleanup).

## Worker Service

### Trigger endpoint

- `POST /worker/start-job`
- Header: `X-Worker-Secret: <WORKER_SHARED_SECRET>`
- Body:
  - `job_id`

Worker validates secret and starts processing.

### Worker processing steps (happy path)

1. Read `jobs/<jobId>.json` from S3.
2. Transition state to `converting` (if PPT/PPTX).
3. Download incoming PPT/PPTX from `incoming/<jobId>/...` to local temp.
4. Convert to PDF using LibreOffice headless.
5. Ensure hidden slides are included in PDF:
   - Use LibreOffice export options for Impress PDF export with `ExportHiddenSlides=true`.
   - Validate output by ensuring PDF page count is at least slide count (best-effort sanity check).
6. Upload PDF to `pdfs/<pdf_filename>` with `ContentType: application/pdf`.
7. Delete the incoming original (`incoming/<jobId>/...`) from S3.
8. Transition state to `ingesting`.
9. Trigger incremental ingest for the output PDF:
   - Call the backend (internal) incremental ingest endpoint (see below), or embed ingest logic in worker (prefer calling backend to centralize RAG code).
10. Transition state to `done`.

### Retry behavior

- For conversion failures, retry up to **3 total attempts** (attempts 1..3), with exponential backoff (e.g., 1s, 2s, 4s).
- Mark job `failed` after final attempt.
- If conversion fails, keep the incoming object until failure is recorded, then delete (to preserve privacy); job error logs remain in job JSON.

### Cleanup

- A daily worker cron (or lightweight scheduled task) deletes:
  - `jobs/*.json` older than 7 days
  - any orphaned `incoming/<jobId>/...` older than 7 days

## Incremental Ingest / Replace Embeddings

### Requirements

- When a PDF with filename `X.pdf` is newly uploaded or overwritten, ingestion must:
  - Delete existing Chroma embeddings where metadata `file_name == "X.pdf"`.
  - Index the new PDF’s contents (no duplicates).

### Proposed implementation (backend)

Add a new admin-only endpoint:

- `POST /api/ingest-file`
  - JSON body: `{ "filename": "Lecture 3.pdf" }`

Backend behavior:

1. Verify admin via Clerk.
2. Ensure `rag_engine.IS_INDEXING` is false (single job lock) then set it true for the duration.
3. If S3 enabled, ensure the PDF exists locally in `./data/uploaded_pdfs/`:
   - Download `pdfs/<filename>` if needed.
4. Delete embeddings for that file:
   - Use existing `delete_pdf_from_database(filename)` (already deletes by `file_name` metadata).
5. Load and index only that PDF:
   - Use a `SimpleDirectoryReader` pointed at a temp dir containing only that PDF OR a reader that loads that single file.
   - Insert into the existing Chroma-backed index (not rebuild everything).
6. Clear `IS_INDEXING` in `finally`.

Notes:

- If LlamaIndex “insert into existing index” APIs are unstable for the current pinned dependencies, fall back to rebuilding the index ONLY as a last resort. The target behavior is incremental ingest.

## Frontend UI Changes

### Unify uploaders (Combined)

Implement combined uploader in BOTH places:

1. `/chat` admin sidebar uploader (`frontend/app/chat/ChatPage.tsx`)
2. `/admin` course materials page (`frontend/app/admin/UploadPDF.tsx`)

Both should:

- Use `accept=".pdf,.ppt,.pptx"`
- Send multipart field name `file`
- On success:
  - Start polling job status every 5 seconds via `/api/upload-status/<jobId>`
  - When job transitions to `ingesting`, the global maintenance popup will appear for students (and for anyone on `/chat`).

### Disable actions during indexing

Use `GET /api/status` to disable:

- Upload buttons/inputs
- Delete buttons

Show message:

- “Indexing in progress—changes temporarily locked.”

### Job progress UI

1. Inline status near uploader: show `state` and `progress.message`.
2. “Recent jobs” panel showing last 5 jobs:
   - Filename, started time, state, result, and “View error” on failures.

### Student UI

- Students see only the existing maintenance popup message (no job details).

## Fonts (Bundled)

To improve PPT → PDF fidelity, install common font packages in the worker image, e.g.:

- Noto fonts (broad Unicode coverage)
- Liberation fonts (metrics-compatible with common Office fonts)
- Optional Microsoft core fonts if licensing permits in your deployment context

Document the chosen font packages in the worker Dockerfile.

## Observability / Logging

- Worker logs should include:
  - job_id, input filename, output pdf filename, state transitions, attempt count, error messages.
- Backend logs should include:
  - job_id creation, worker trigger result, ingest start/finish for `ingest-file`.

## Backwards Compatibility & Migration

- Existing PDFs in `pdfs/` remain unchanged.
- Manual bulk ingest remains available.
- UI behavior differences today:
  - `/chat` uploader already auto-ingests; `/admin` does not. After changes, both will use the job model and consistent behavior.

## Acceptance Criteria

1. Admin can upload `.pptx`, `.ppt`, or `.pdf` from `/chat` or `/admin`.
2. PPT/PPTX upload results in a PDF named `<base>.pdf` stored at `pdfs/<base>.pdf`.
3. Hidden slides are included in the PDF export.
4. The app’s file list shows the converted PDF and it is viewable via the existing PDF viewer.
5. Upload triggers incremental ingest of that PDF:
   - No duplicate embeddings remain for that file.
6. During ingestion:
   - Students cannot use the bot (maintenance popup).
   - Admin can access admin pages, but upload/delete are disabled with a message.
7. Job status is visible to admins only, includes retries (3 attempts), and is retained 7 days.
8. Backend admin endpoints require Clerk token verification and `publicMetadata.role === "admin"`.
9. Worker trigger endpoint is protected by `WORKER_SHARED_SECRET`.
10. System rejects new uploads while a job is active/indexing.

