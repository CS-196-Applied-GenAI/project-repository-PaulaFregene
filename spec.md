# Spec: Clickable Citations → Jump to PDF Page

## Overview
Students currently see citations beneath each tutor response (e.g., `cs211-03-lecture.pdf (Page 5)`), but they are not interactive. This feature makes citations **clickable** so students can **jump directly** to the referenced PDF and page inside the app (desktop) or open the PDF at that page in a new browser tab (mobile).

This supports the project’s core principle: **answers must be grounded in instructor-provided course PDFs** and provide transparent, actionable source navigation.

## Goals
- Make each citation in chat **clickable** in both tutor modes (Socratic + Direct).
- On desktop, clicking a citation:
  - selects the corresponding PDF in the left sidebar (updates `activeFile`)
  - loads that PDF in the middle viewer
  - jumps to the cited page number \(N\) **exactly as returned by the backend**
  - shows a short confirmation toast for 2–3 seconds.
- On mobile (where the PDF column is hidden), clicking a citation opens the PDF at that page in a **new browser tab**.
- Use **PDF display names** (admin-configured) for what students see in the clickable citation text and in the toast.
- Be robust to missing/unavailable PDFs by showing an **error toast** and not navigating.
- Keep the system scalable (target 100+ concurrent users): avoid backend hot-path file lookups for display names.

## Non-goals (for this week)
- Highlighting the exact cited text region inside the PDF.
- In-PDF annotations or overlays.
- Building a dedicated mobile in-app PDF screen/modal.
- Changing retrieval logic or citation ranking logic.

## Current System Context (as implemented)
- Backend:
  - `POST /api/query` returns `content`, `citations` (currently strings), `role`, `mode`.
  - RAG pipeline in `backend/rag_engine.py` constructs citations from LlamaIndex `source_nodes`, producing strings like `"file.pdf (Page 16)"` (page is intended to be 1-based).
- Frontend:
  - Chat UI renders citations as plain text under each assistant message in `frontend/app/chat/ChatPage.tsx`.
  - PDF viewer is an `<iframe>`:
    - `src={fileUrls[activeFile] || \`${API_BASE_URL}/pdfs/${activeFile}\`}`
  - `/api/files` returns `files`, `display_names`, and optional `file_urls` (S3 URLs).

## User Stories
- As a student, when I see a cited source under an answer, I can click it and be taken to the exact page in the course PDF so I can verify and review.
- As an instructor/admin, I can set friendly display names; students see those names in citations rather than raw filenames.
- As a mobile user, tapping a citation opens the relevant PDF page in a new tab since the in-app PDF pane is hidden.

## UX Requirements

### Citation display format
- In chat, each citation is rendered as a clickable link-like element displaying:
  - **`{displayName} (Page {N})`**
  - If no display name exists for the file, fall back to the **raw filename**.

### Desktop click behavior
When a citation is clicked:
- Update sidebar selection to the cited PDF (set `activeFile = citation.file`).
- Update the iframe `src` so it navigates to the cited page:
  - `pdfUrlWithPage = basePdfUrl + "#page=" + N`
  - `N` is the number provided by the backend and should be used directly.
- Show a toast for ~2–3 seconds:
  - Text: **`Opened "{displayName}", Page {N}`**

### Mobile click behavior
- Open the PDF in a new browser tab/window at that page:
  - `window.open(basePdfUrl + "#page=" + N, "_blank", "noopener,noreferrer")`
- No in-app navigation required.

### Missing/unavailable PDF handling
If the citation’s `file` is not present in the current sidebar `files` list:
- Do **not** change `activeFile`.
- Show an error toast for ~2–3 seconds, e.g.:
  - **`Couldn’t open source: file not found`**
  - Optional extended copy: “Ask your instructor/admin to upload it.”

## Backend API Changes

### `POST /api/query` response shape (transition-safe)
Keep backward compatibility by returning both:
- **`citations`**: new structured citations
- **`citations_text`**: legacy string citations (existing format)

Proposed response (example):

```json
{
  "content": "…",
  "citations": [
    { "file": "cs211-03-lecture.pdf", "page": 5 },
    { "file": "cs211-04-lecture.pdf", "page": 16 }
  ],
  "citations_text": [
    "cs211-03-lecture.pdf (Page 5)",
    "cs211-04-lecture.pdf (Page 16)"
  ],
  "role": "assistant",
  "mode": "direct"
}
```

#### Structured citation schema
- `file` (string): PDF filename as known to `/api/files` and the `pdfs/` store.
- `page` (number): page number **1-based**, exactly as currently output in string citations.

### Backend implementation details
- Update `backend/rag_engine.py` return payload to build both:
  - `citations_text: list[str]` (existing behavior)
  - `citations: list[{"file": str, "page": int}]`
- Keep the existing “top 3 unique sources” constraint.
- Do **not** add `display_name` in `/api/query` to avoid extra per-request I/O. The frontend will map file → display name using `/api/files`.

### Performance constraints
- Avoid additional filesystem reads or S3 calls during `/api/query`.
- Citation generation must remain \(O(k)\) over returned `source_nodes` (k ≤ 15 retrieval nodes; citations limited to 3 unique).

## Frontend Changes

### Data model updates
Current messages likely store `citations?: string[]`. Update to support:
- `citations?: Array<{ file: string; page: number }>`
- `citations_text?: string[]` (optional fallback during transition)

### Rendering citations
- Render each citation as a clickable element (e.g., `<button>` styled as a link, for accessibility).
- Display label uses display name mapping from `/api/files`:
  - `displayName = display_names[file] ?? file`
  - Render text: `📍 {displayName} (Page {page})`

### Navigation logic
Add a single click handler:
- Validate `file` exists in current `files` list.
  - If missing → show error toast and return.
- If desktop:
  - set `activeFile` to `file`
  - update iframe `src` to include `#page={page}`
- If mobile:
  - compute PDF URL and open in new tab at `#page={page}`

### Building the PDF URL
Use the same source as the iframe currently uses:
- If S3 URL exists: `fileUrls[file]`
- Else: `${API_BASE_URL}/pdfs/${file}`

Then append `#page={page}`.

### Toast requirements
- Show toast on successful click: `Opened "{displayName}", Page {page}`
- Show toast on error: `Couldn’t open source: file not found`
- Duration: 2–3 seconds

## Edge Cases
- **Multiple citations** pointing to the same file but different pages: each click should jump appropriately.
- **Active file already selected**: clicking citation should still update iframe `src` so it navigates to the page.
- **Page number not parseable / missing**:
  - In structured citations, `page` must be a number. If backend cannot provide a page, omit the citation rather than emitting invalid objects.
- **S3 signed URLs**:
  - If the URL already contains a fragment/hash, ensure `#page=` is appended correctly (typically fragments replace; preferred approach: always append `#page=N` as the fragment).

## Success Criteria / Acceptance Tests

### Desktop
- Given an assistant response with citations, each citation appears as clickable text:
  - `Display Name (Page N)`
- Clicking a citation:
  - selects that PDF in the sidebar (visible highlight/selection matches screenshot behavior)
  - loads that PDF in the iframe
  - navigates to `#page=N` using the backend-provided N
  - shows toast: `Opened "Display Name", Page N` for ~2–3 seconds

### Mobile
- Tapping a citation opens a new tab to the correct PDF and page:
  - URL ends with `#page=N`

### Missing file
- If a citation references a PDF not in `/api/files` list:
  - no navigation occurs
  - error toast shown

## Implementation Plan (high level)
- Backend:
  - Add structured citation objects in `backend/rag_engine.py` return payload.
  - Update `backend/main.py` `/api/query` response to include both `citations` (objects) and `citations_text` (strings) during transition.
- Frontend:
  - Update message types to support new structured citations.
  - Render citations as clickable elements showing display name + page.
  - Implement desktop jump behavior by updating `activeFile` and iframe `src` with `#page=`.
  - Implement mobile click behavior via `window.open()`.
  - Add toast UI for success/error.

## Future Enhancements (next iterations)
- **Highlighting upgrade**: highlight the cited page, and optionally highlight text:
  - likely requires moving away from bare `<iframe>` to a controllable PDF viewer (e.g., PDF.js or `react-pdf`) to support text-layer selection/highlight.
- Citation tooltip transparency:
  - show raw filename on hover while displaying friendly display name in the main label.
