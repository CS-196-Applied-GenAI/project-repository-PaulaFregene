# Backend Plan: Clickable Citations → Jump to PDF Page

This plan implements the backend portion of the feature described in `spec.md`: return **structured citations** (file + 1-based page) while keeping **legacy citation strings** for a safe frontend transition.

## Backend deliverable (definition of done)
`POST /api/query` returns:
- `content`: assistant answer text (unchanged)
- `citations`: **array of objects** `[{ file: string, page: number }]`
- `citations_text`: **array of strings** `["file.pdf (Page N)", ...]` (legacy)
- `role`, `mode`: unchanged

No additional filesystem reads, S3 calls, or expensive operations are added to the `/api/query` hot path.

---

## Step-by-step blueprint (what to change and where)

### 1) Decide the backend contract precisely
- **New type**: `Citation`
  - `file: str` (the PDF filename used in `/api/files`, e.g. `cs211-03-lecture.pdf`)
  - `page: int` (1-based, use exactly the same page number currently shown in citation strings)
- **Response contract**: `citations` and `citations_text` are both present.
- **Backward compatibility**: keep existing `citations` string list semantics temporarily by moving it to `citations_text`, while `citations` becomes structured.

### 2) Add Pydantic models for clarity (optional but recommended)
Current `backend/models.py` only defines request models. Add response models to reduce accidental shape regressions.

Suggested models:
- `class Citation(BaseModel): file: str; page: int`
- `class QueryResponse(BaseModel): content: str; citations: list[Citation]; citations_text: list[str]; role: str; mode: str`

### 3) Implement structured citation extraction in `backend/rag_engine.py`
Today `query_rag()` returns `{\"answer\": answer_text, \"citations\": citations}` where `citations` is a list of strings.

Change the citation building block to produce two parallel outputs:
- `citations_text: list[str]` (existing)
- `citations: list[dict]` with `file` + `page` (new)

Key requirements:
- Continue stopping after **top 3 unique** sources.
- Preserve current uniqueness criteria (currently `file_name|page_label`).
- Page handling:
  - Prefer `page_label` if present.
  - Else if `page_num` is present, convert to 1-based via `int(page_num) + 1`.
  - If neither exists or it’s not parseable: skip that citation (don’t emit invalid objects).
- If the answer includes `"I cannot find this information"` or `"[NO_CITATIONS]"`:
  - return **empty** `citations` and `citations_text`
  - strip `[NO_CITATIONS]` from answer text (existing behavior)

### 4) Update `backend/main.py` to pass through the new fields
Update `/api/query` to:
- read `result[\"citations\"]` (new structured citations)
- read `result[\"citations_text\"]` (legacy text)
- return both fields in the JSON response

Also update the early failure case (missing `SYSTEM_PROMPT`) to include:
- `citations: []`
- `citations_text: []`

### 5) Validate that the `file` field is stable
Structured `file` should exactly match what `/api/files` returns (since frontend uses that list to decide availability and to map display names).

Given current RAG metadata uses `metadata.get(\"file_name\")`, ensure it matches the uploaded filename (it should, as long as ingestion preserves `file_name`).

### 6) Add automated tests (backend-only)
Two layers:

#### Unit tests for citation parsing/extraction
Refactor the “citation building” logic in `rag_engine.py` into a helper function that can be unit-tested without calling the LLM:
- Input: list of “source node metadata dicts” (or small adapter objects)
- Output: `(citations, citations_text)`

Unit test cases:
- **Happy path**: `file_name` + `page_label="5"` → emits `{file, page: 5}` and `"file (Page 5)"`
- **page_num fallback**: `page_num=0` → emits page 1
- **dedupe**: repeated `file|page` only included once
- **limit**: more than 3 unique citations → only first 3 returned
- **unknown page**: missing page info → skipped (no `page="?"` objects)

#### Integration tests for `/api/query` response shape (minimal)
Mock `rag_engine.query_rag` to return deterministic outputs and assert:
- response includes both fields
- types: `citations` is list of objects with `file`/`page`, `citations_text` is list of strings

This avoids hitting OpenAI/Chroma in CI.

### 7) Smoke test manually (developer workflow)
Local run:
- Start backend
- Call `/api/query` with a known question that yields citations
- Verify JSON includes both `citations` and `citations_text`
- Confirm `page` values match what users previously saw in the UI string citations

---

## Implementation chunks (iterative, building on each other)

Each chunk is intentionally small, reversible, and testable.

### Chunk 1: Introduce new citation type + keep legacy unchanged (scaffolding)
Outcome: backend code compiles, no behavior change yet.
- Add Pydantic `Citation` model (and optional `QueryResponse`).
- Add a minimal helper signature placeholder (no functional changes).

**Validation**
- Run backend import / startup (no runtime behavior change).

### Chunk 2: Generate structured citations in `rag_engine.py` (core logic)
Outcome: `query_rag()` returns both `citations` (objects) and `citations_text` (strings), but `/api/query` may still be returning the old shape until Chunk 3.

**Validation**
- Unit tests for extraction helper (fast, deterministic).

### Chunk 3: Update `/api/query` response to return both fields (API contract)
Outcome: clients can start using `citations` objects while still having `citations_text`.

**Validation**
- Integration test that mocks `query_rag` and asserts response shape.

### Chunk 4: Harden edge cases + logging (small polish)
Outcome: no invalid citations emitted; consistent empty lists on fallback paths.
- Ensure all fallback returns include both fields.
- Ensure unparseable pages are skipped, not emitted as `?`.

**Validation**
- Add/extend unit tests for edge cases.

---

## Second round: break each chunk into small, safe steps

### Chunk 1 steps (scaffolding)
1. Add `Citation` model to `backend/models.py` (no runtime use yet).
2. (Optional) Add `QueryResponse` model (not enforced yet).
3. Run a quick backend startup to ensure imports work.

### Chunk 2 steps (core logic)
1. Extract citation-building loop in `backend/rag_engine.py` into a helper function, e.g.:
   - `build_citations_from_source_nodes(source_nodes) -> tuple[list[dict], list[str]]`
2. Implement parsing rules:
   - prefer `page_label`, else `page_num + 1`, else skip
3. Preserve dedupe + limit-to-3 behavior.
4. Update `query_rag()` to call helper and return:
   - `{\"answer\": answer_text, \"citations\": citations, \"citations_text\": citations_text}`
5. Add unit tests for the helper (no external services).

### Chunk 3 steps (API contract)
1. Update `/api/query` in `backend/main.py`:
   - `citations_struct = result.get(\"citations\", [])`
   - `citations_text = result.get(\"citations_text\", [])`
2. Update the missing-`SYSTEM_PROMPT` early return to include both citation fields.
3. Update the exception/failure path to include both citation fields (empty lists).
4. Add integration test:
   - monkeypatch/mock `query_rag` to return both fields
   - assert `/api/query` response includes both fields

### Chunk 4 steps (hardening)
1. Add tests for:
   - `page_label=\"?\"` or non-numeric labels → skip structured object but may keep text if desired (recommend skipping both for consistency).
2. Decide & enforce consistency rule:
   - If structured citation is skipped due to non-numeric page, also skip its text entry (avoid UI showing a citation the app can’t navigate to).
3. Ensure `[NO_CITATIONS]` path returns:
   - `citations=[]`, `citations_text=[]`

---

## Third round: make steps “right sized” for safe implementation

The smallest practical, safe sequence (each step can be a single PR/commit):

1. **Add models only**
   - Add `Citation` (and optionally `QueryResponse`) in `backend/models.py`.
   - No endpoint changes.

2. **Add extraction helper + unit tests (no endpoint changes)**
   - Create helper in `rag_engine.py`.
   - Add `pytest` unit tests for helper.
   - Keep `query_rag()` returning old `citations` strings for now.

3. **Switch `query_rag()` to return both `citations` (objects) + `citations_text` (strings)**
   - Update return dict.
   - Update unit tests (if needed) to cover returned structure.

4. **Switch `/api/query` to return both fields**
   - Update `main.py` response.
   - Add integration test with mocked `query_rag`.

5. **Edge-case hardening**
   - Ensure non-numeric pages never produce structured citations.
   - Ensure both fields are present and empty on fallback/error paths.

This progression minimizes risk: you get tests in place *before* the public API shape changes, and every step is independently verifiable.
