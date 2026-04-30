# Ops setup guide: Railway + AWS + Clerk + Docker (PPT/PPTX upload pipeline)

This is the “outside the codebase” runbook that pairs with:

- `documentations/pptx-upload/spec-pptx-upload.md`
- `documentations/pptx-upload/plan-pptx-upload.md`
- `prompt-ppt-upload.md`

It explains **what to configure and when**, so each PR/prompt can be deployed and tested without missing keys or broken wiring.

## High-level rollout order (recommended)

1. **Clerk token verification config** on Railway (web service) so admin endpoints can be secured safely.
2. **S3 prefixes readiness** (`pdfs/`, `jobs/`, `incoming/`) and IAM permissions.
3. **Worker service** creation on Railway (but can be idle until later PRs).
4. **Worker secret + internal endpoint secret** wiring.
5. **LibreOffice + fonts** installed in worker Docker image.
6. **Internal URLs** (worker base URL and web base URL) set correctly.
7. **Cleanup job** (optional) after pipeline works.

## AWS (S3 + IAM) prerequisites

### 1) S3 bucket structure

You already have an S3 bucket used in production (`USE_S3=true`). Ensure these logical prefixes exist (S3 doesn’t require folders; this is about conventions):

- `pdfs/` — canonical PDFs used by the app
- `incoming/` — **temporary** PPT/PPTX uploads per job
- `jobs/` — job status JSON objects

No manual creation is required; the app will create objects under these keys.

### 2) IAM policy requirements (web + worker)

Both the **web** and **worker** services need S3 access. Recommended is **two IAM users/roles** (one for web, one for worker), but you can reuse the same keys initially.

Minimum permissions (bucket-scoped) for **web service**:

- Read/list:
  - `s3:ListBucket` (with prefix conditions if desired)
  - `s3:GetObject` for `pdfs/*`, `jobs/*`, `incoming/*`
- Write:
  - `s3:PutObject` for `pdfs/*`, `jobs/*`, `incoming/*`
- Delete:
  - `s3:DeleteObject` for `jobs/*`, `incoming/*` (optional for web; worker will delete incoming)

Minimum permissions for **worker service**:

- `s3:ListBucket`
- `s3:GetObject` for `incoming/*` and `jobs/*` and (optionally) `pdfs/*`
- `s3:PutObject` for `pdfs/*` and `jobs/*`
- `s3:DeleteObject` for `incoming/*` and `jobs/*` (cleanup)

### 3) AWS env vars (already present)

You already use:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_S3_BUCKET`
- `AWS_REGION`
- `USE_S3=true`

These must be set on **both** Railway services once the worker exists.

## Clerk prerequisites (admin enforcement)

### 1) Admin role data convention

Your admin convention:

- `publicMetadata.role` must be exactly `"admin"` (lowercase)

Ensure your admin accounts are set accordingly in the Clerk dashboard.

### 2) Token verification (what you’ll need)

The implementation plan uses a JWKS-based verification. You will need at least:

- `CLERK_JWKS_URL`
  - Typically: `https://<your-clerk-domain>/.well-known/jwks.json` (exact value depends on Clerk setup)
- Optional: `CLERK_ISSUER`
  - Used to validate the JWT `iss` claim if you choose to enforce it

Where to find these:

- Clerk dashboard → your application → **JWT / Sessions / API Keys** areas (naming varies).
- If you already use Clerk in Next.js, you likely have a frontend issuer/domain; your backend can verify via JWKS.

### 3) Frontend must send Authorization header

Once backend enforcement is enabled, the frontend must include:

- `Authorization: Bearer <Clerk session token>`

This is a code change, but operationally you should ensure:

- Clerk is correctly configured in production
- Admin users can sign in successfully

## Railway: services and environment variables

### Service A — Web backend (existing)

Current (from your screenshot) you have AWS keys, `USE_S3`, `OPENAI_API_KEY`, etc.

You will add (when the corresponding PR is deployed):

- **Clerk verification**
  - `CLERK_JWKS_URL` (required once PR 2+ is deployed)
  - `CLERK_ISSUER` (optional, only if code validates issuer)

- **Worker integration**
  - `WORKER_SHARED_SECRET` (required once internal endpoint + worker trigger are deployed)
  - `WORKER_BASE_URL` (required once you enable backend → worker HTTP trigger)
    - Example: `https://<your-worker-service>.up.railway.app` (Railway gives a URL per service)

### Service B — Worker (new Railway service)

Create a new Railway service in the same project:

- Source: same GitHub repo
- Build: from `worker/Dockerfile` (once PR 12 lands) or equivalent configuration
- Start command: whichever entrypoint the worker code uses (e.g., `uvicorn worker.app:app --host 0.0.0.0 --port $PORT`)

Worker env vars:

- AWS:
  - `AWS_ACCESS_KEY_ID`
  - `AWS_SECRET_ACCESS_KEY`
  - `AWS_S3_BUCKET`
  - `AWS_REGION`
  - `USE_S3=true`
- Secrets:
  - `WORKER_SHARED_SECRET` (must match web service)
- Web callback base URL:
  - `WEB_BASE_URL`
    - Example: `https://<your-backend-service>.up.railway.app`

### Recommended Railway networking notes

- If Railway provides internal/private networking between services, prefer that for `WORKER_BASE_URL` and `WEB_BASE_URL`.
- If not, HTTPS public URLs still work; protect worker endpoints with `WORKER_SHARED_SECRET`.

## Docker prerequisites (worker)

When you reach PR 12+, worker images need:

- LibreOffice headless
- Font packages (Noto + Liberation)

Operational checks after deployment:

- Worker logs show `soffice --version` works (or equivalent startup log).
- Memory/CPU: LibreOffice can be heavy; ensure worker has sufficient resources. If conversion fails due to OOM, scale worker instance size.

## Step-by-step external setup mapping (PR/prompt aligned)

This section maps to the PR sequence in `prompt-ppt-upload.md`.

### Prompt 1 (Auth scaffolding)

No external setup needed.

### Prompt 2 (Token verification + dependency)

**Do before deploying this code to production**:

- Add `CLERK_JWKS_URL` to the **web backend** Railway service variables.
- (Optional) Add `CLERK_ISSUER` if code enforces issuer.

**Smoke check after deploy**:

- Backend starts successfully (no missing env var crash).
- You can still hit public endpoints like `/health`.

### Prompt 3–4 (Protect endpoints)

**Do before deploying**:

- Ensure frontend is ready (or immediately after deploy) to send the Authorization header for admin endpoints.
  - If the frontend is not updated yet, you may temporarily lock yourself out of admin endpoints.

**Operational advice**:

- Deploy these changes during a maintenance window, and have an admin signed in to verify quickly.

### Prompt 5–6 (Job store + job APIs)

**Do before running in production**:

- Confirm AWS credentials are correct and have `PutObject/GetObject/ListBucket` for the `jobs/` prefix.

**Smoke check**:

- As admin, call:
  - `GET /api/upload-jobs/recent?limit=5` (should return `[]` initially or list)

### Prompt 7 (Upload creates job — PDF path only)

**Do before deploying**:

- Confirm AWS permissions include `PutObject` to `pdfs/*` and `jobs/*`.

**Smoke check**:

- Upload a PDF in admin UI, then:
  - verify `jobs/<job_id>.json` exists in S3
  - verify `pdfs/<filename>.pdf` exists in S3

### Prompt 8 (Single-job lock)

No new external keys.

**Operational check**:

- While indexing or while an active job exists, upload should return 409.

### Prompt 9–10 (Incremental ingest-file + no duplicates)

**Do before deploying**:

- Nothing new in AWS/Railway, but ensure:
  - Backend has enough CPU/memory for indexing.
  - `OPENAI_API_KEY` is present (required by `rag_engine.py`).

**Operational check**:

- After a PDF upload, call `POST /api/ingest-file` (or let UI do it later) and confirm:
  - students see maintenance popup during ingest
  - after completion, queries cite the new PDF

### Prompt 11 (Worker skeleton)

**Do on Railway**:

- Create the **worker service** (even if it’s a minimal app).
- Set worker env vars (AWS + `WORKER_SHARED_SECRET` + `WEB_BASE_URL`).

**Do on the web backend service**:

- Set `WORKER_SHARED_SECRET` (same as worker).

**Smoke check**:

- Call worker endpoint with correct secret (from a secure client) and confirm 202.

### Prompt 12 (Worker Docker adds LibreOffice + fonts)

**Do on Railway**:

- Ensure worker builds using the new Dockerfile.
- Increase worker resources if needed (LibreOffice often needs more RAM).

**Smoke check**:

- Worker logs show LibreOffice is available.

### Prompt 13 (Worker conversion + S3 upload)

**Do before testing**:

- Ensure worker IAM can:
  - read `incoming/*`
  - write `pdfs/*`
  - read/write `jobs/*`
  - delete `incoming/*`

**Smoke check**:

- Manually create a job + incoming object (or via the web upload once enabled) and confirm:
  - PDF appears in `pdfs/`
  - incoming is deleted
  - job transitions to `done` or `failed` with retries

### Prompt 14 (Internal ingest endpoint + worker call)

**Do on Railway**:

- On web backend service: ensure `WORKER_SHARED_SECRET` is set.
- On worker service: ensure `WORKER_SHARED_SECRET` and `WEB_BASE_URL` are set correctly.

**Networking check**:

- Worker must be able to call `WEB_BASE_URL/internal/ingest-file`.
- If your backend is behind auth/network policies, adjust accordingly.

**Smoke check**:

- Run a PPTX job end-to-end and confirm `state=ingesting` appears and then `done`.

### Prompt 15 (Cleanup)

**Do on Railway**:

- Decide how cleanup will run:
  - Option A: manual trigger (call `/worker/cleanup` occasionally)
  - Option B: Railway cron job hitting `/worker/cleanup`

**If using cron**:

- Configure Railway scheduled task to call:
  - `POST https://<worker>/worker/cleanup` with `X-Worker-Secret`

### Prompt 16 (Enable PPT/PPTX upload path and worker trigger)

**Do on web backend service**:

- Set `WORKER_BASE_URL` to worker public URL (or internal URL if available).
- Ensure `WORKER_SHARED_SECRET` is set and matches worker.

**Do on worker service**:

- Ensure worker is deployed and healthy.

**Full smoke test**:

1. Upload PPTX in admin UI.
2. Confirm S3:
   - `incoming/<job_id>/...` exists briefly then is deleted
   - `pdfs/<base>.pdf` exists
   - `jobs/<job_id>.json` shows full state transitions
3. Confirm app:
   - maintenance popup during ingestion
   - after done, query uses new PDF

## Troubleshooting checklist

### Backend returns 401/403 unexpectedly

- Confirm frontend is sending `Authorization: Bearer <token>`.
- Confirm JWT verification env vars are correct:
  - `CLERK_JWKS_URL` points to the right app/environment
- Confirm admin user has `publicMetadata.role: "admin"` (lowercase).

### Worker start-job returns 401

- `WORKER_SHARED_SECRET` mismatch between caller and worker.
- Caller forgot `X-Worker-Secret` header.

### Job stuck in converting / no PDF produced

- Check worker logs for LibreOffice invocation errors.
- Increase worker RAM/CPU (OOM is common).
- Ensure fonts installed (missing fonts can cause LO errors or poor output).

### Job status not updating / missing jobs

- IAM policy missing `PutObject` to `jobs/*`.
- S3 bucket name/region mismatch.

### Ingest creates duplicates

- Ensure ingest-file deletes embeddings for that `file_name` before indexing.
- Confirm `filename` used matches the metadata `file_name` stored in Chroma.

### Maintenance popup never appears

- Ensure `rag_engine.IS_INDEXING` is set `True` during ingest-file and cleared in `finally`.
- Confirm `/api/status` is reachable and returns `is_indexing: true` while indexing.

## Secret generation guidance

### WORKER_SHARED_SECRET

- Generate a long random string (32+ bytes).
- Store it only in Railway variables for web + worker services.
- Never commit it to git.

## Suggested “staging” approach without extra infra

If you don’t want a separate staging environment:

- Use a temporary S3 bucket/prefix and temporary Railway variables during rollout.
- Or deploy PRs one by one and validate using admin-only endpoints before enabling PPTX upload in Prompt 16.

