# Prompts: Implement Backend Structured Citations (TDD, incremental)

Use these prompts *in order* with a code-generation LLM. Each prompt is designed to be small, test-driven, and fully integrated with the previous work (no orphan code).

## Shared context for every prompt
- Repo is `my-tutor-bot`.
- Backend is FastAPI in `backend/main.py`.
- RAG logic is in `backend/rag_engine.py` (`query_rag` currently returns `{ "answer": ..., "citations": [string, ...] }`).
- Goal per `spec.md` / `plan.md`: `POST /api/query` must return both:
  - `citations`: structured `[{ file: string, page: number }]`
  - `citations_text`: legacy `["file.pdf (Page N)", ...]`
- Performance constraint: **no additional filesystem reads / S3 calls** in `/api/query` hot path.
- Testing constraint: tests must **not** call OpenAI/Chroma; use unit tests + mocked integration tests.

---

## Prompt 1 — Add Pydantic models (no runtime behavior change)

```text
You are working in the repo `my-tutor-bot` (FastAPI backend).

Task: Add response models for the new citation contract WITHOUT changing runtime behavior yet.

Requirements:
1) Edit `backend/models.py`:
   - Add a Pydantic model `Citation` with fields:
     - `file: str`
     - `page: int`  (1-based)
   - Add a Pydantic model `QueryResponse` with fields:
     - `content: str`
     - `citations: list[Citation]`
     - `citations_text: list[str]`
     - `role: str`
     - `mode: str`
2) Do NOT change any endpoints yet (`backend/main.py` stays behavior-identical).
3) Keep imports clean and compatible with the project’s Python version (see backend requirements).

After changes:
- Ensure backend still starts (no failing imports).
- Do not add placeholder comments; keep it minimal.
```

---

## Prompt 2 — Add a pure helper for citation extraction + unit tests (no endpoint changes)

```text
You are working in the repo `my-tutor-bot`.

Task: Refactor the citation-building logic in `backend/rag_engine.py` into a pure helper function and add unit tests for it, WITHOUT changing the public API behavior yet.

Current behavior (important):
- `query_rag()` currently creates `citations` as strings like: "file.pdf (Page 5)"
- It dedupes by `file_name|page_label`
- It returns up to 3 unique citations
- If page is not available it uses "?" in the string today

New goal (for this step only):
- Create a helper that can generate BOTH:
  - structured citations: `[{ "file": "...", "page": 5 }, ...]`
  - citation strings: `["... (Page 5)", ...]`

BUT: do not wire it into `query_rag()` yet; keep `query_rag()` returning exactly what it does today so nothing breaks mid-step.

Implementation requirements:
1) In `backend/rag_engine.py`, add a function with a signature like:
   `def build_structured_citations_from_source_nodes(source_nodes, limit: int = 3): ...`
   - The function must be pure (no network, no filesystem).
   - It should accept an iterable of objects where each has `.node.metadata` OR accept metadata dicts; pick one approach and test accordingly.
2) Parsing rules inside helper:
   - `file = metadata.get("file_name", "Unknown File")`
   - Determine page:
     - if `page_label` is present and is numeric => page=int(page_label)
     - else if `page_num` is present => page=int(page_num)+1
     - else => skip this source entirely (do NOT emit structured citations with invalid page)
   - Dedupe: unique key = `file|page`
   - Stop after `limit` unique citations.
   - Create `citations_text` strings exactly `"${file} (Page ${page})"` corresponding to structured entries.
3) Add unit tests using `pytest` in a new test file (choose an appropriate path under `backend/` or a `tests/` folder if one exists).
   - Tests must not import OpenAI clients or run indexing.
   - Create minimal fake objects/fixtures to simulate `source_nodes` with metadata.
   - Required test cases:
     - Happy path `page_label="5"`
     - Fallback from `page_num=0` => page 1
     - Dedupe repeats
     - Limit to 3
     - Missing page info => skipped
     - Non-numeric `page_label="?"` => skipped

After changes:
- Ensure `pytest` runs and passes locally.
- Do not change `query_rag()` return shape yet.
```

---

## Prompt 3 — Wire helper into `query_rag()` and return BOTH fields (backend internal change)

```text
You are working in the repo `my-tutor-bot`.

Task: Update `backend/rag_engine.py` so `query_rag()` returns both `citations` (structured) and `citations_text` (legacy strings), while preserving existing fallback behavior.

Requirements:
1) In `query_rag()`:
   - Replace the existing citation-building loop with a call to the helper you created.
   - If the answer contains "I cannot find this information" OR "[NO_CITATIONS]":
     - return `citations=[]` and `citations_text=[]`
     - strip `[NO_CITATIONS]` from the answer text (existing behavior)
2) Return dict from `query_rag()` must become:
   `{ "answer": answer_text, "citations": structured_list, "citations_text": citations_text_list }`
3) Ensure the helper produces at most 3 citations and skips invalid pages.
4) Update/extend unit tests if needed to reflect that the helper output is now the source of truth.

Important constraints:
- Do not add new I/O (no file reads / S3 calls / DB reads) in `query_rag()` for this feature.

After changes:
- Run pytest; all tests pass.
```

---

## Prompt 4 — Update `/api/query` to return both `citations` and `citations_text` + add integration test

```text
You are working in the repo `my-tutor-bot` backend (FastAPI).

Task: Update `POST /api/query` in `backend/main.py` to return BOTH citation fields, and add a minimal integration test that mocks `rag_engine.query_rag` (no external calls).

Requirements:
1) Update `backend/main.py`:
   - In the missing-`SYSTEM_PROMPT` early return, include:
     - `citations: []`
     - `citations_text: []`
   - In the normal path after calling `query_rag()`, read:
     - `citations = result.get("citations", [])`
     - `citations_text = result.get("citations_text", [])`
   - Return JSON must include both fields.
   - In the exception path, still return both fields as empty lists.
2) Add an integration test using FastAPI’s TestClient:
   - Monkeypatch/mock `backend.main.query_rag` (or `rag_engine.query_rag` depending on import) to return:
     - `answer`, `citations` objects, `citations_text` strings
   - Call `/api/query` with a dummy request body.
   - Assert response JSON has:
     - `content` matches
     - `citations` is a list of objects with keys `file` and `page`
     - `citations_text` is a list of strings
     - `role` and `mode` exist
3) Tests must not require OpenAI keys, Chroma data, or any PDFs.

After changes:
- Run pytest; all tests pass.
```

---

## Prompt 5 — Hardening: enforce consistency + edge cases (tests first)

```text
You are working in the repo `my-tutor-bot`.

Task: Harden citation handling to ensure consistency and avoid emitting citations that can’t be navigated.

Testing-first requirements:
1) Add/extend unit tests for the citation helper:
   - Ensure non-numeric `page_label` (e.g. "?", "iv", "Page 5") yields NO structured citation and NO citation text.
   - Ensure missing `file_name` still works (falls back to "Unknown File") but only if page is valid.
2) Confirm `[NO_CITATIONS]` and "I cannot find this information" paths return:
   - `citations=[]`, `citations_text=[]`

Implementation requirements:
1) Update helper parsing to accept only strict numeric `page_label`.
2) Ensure that whenever a structured citation is skipped, the corresponding text entry is also skipped (they must stay aligned).

After changes:
- Run pytest; all tests pass.
```

---

## Prompt 6 — Final wiring + manual smoke checklist (no new codepaths)

```text
You are working in the repo `my-tutor-bot`.

Task: Do a final pass to ensure everything is wired correctly and consistent with the contract in `spec.md`.

Requirements:
1) Confirm `POST /api/query` always returns BOTH `citations` and `citations_text`:
   - success path
   - missing `SYSTEM_PROMPT` path
   - exception path
2) Ensure `citations` objects have:
   - `file` matching RAG metadata `file_name`
   - `page` as an integer 1-based
3) Do NOT add any new dependencies or external calls.

Output:
- Provide a short manual smoke-test checklist for a developer to run locally (commands + a sample curl/httpie request).
```
