# Iterative Blueprint: Research & Admin Visualization Dashboard

## Phase 1: Database & Data Pipeline Foundation
**Goal:** Establish the underlying schema changes and background processes required to support analytics without touching the student-facing AI logic.

* **Step 1.1: Schema Migrations (PostgreSQL)**
    * Add `course_id` column to relevant tables (default: 'CS 211').
    * Add `visibility` boolean (`is_hidden`) to the Syllabus/Content table.
    * Add `is_archived` boolean to the Chats/Sessions table.
    * Add `interaction_mode` tracking (Socratic vs. Direct Answer/Hint) to individual message logs.
    * *Testing:* Run migration rollbacks and verify schema integrity.
* **Step 1.2: The Topic Clustering Worker**
    * Create an asynchronous background job (using the existing `worker` architecture) to run LLM embeddings on completed chat sessions.
    * Extract and store "Trending Topics" and "Concept Categories" in a new analytics table.
    * *Testing:* Unit test the worker with mock chat logs to ensure no hallucinations bleed into the clustering logic.
* **Step 1.3: Timezone Normalization**
    * Implement database-level or ORM-level timezone conversion to ensure all stored timestamps are strictly cast and returned as Chicago CST.

## Phase 2: Core Backend APIs (Python/FastAPI)
**Goal:** Build the secure endpoints required to feed the frontend tabs.

* **Step 2.1: RBAC & Auth Middleware**
    * Integrate Clerk authentication for the `/admin` API namespace.
    * Enforce manual whitelist check for the four approved researcher/admin roles.
    * *Testing:* Write tests ensuring non-admin users receive 403 Forbidden errors.
* **Step 2.2: Chats & Analytics Endpoints**
    * Build `GET /admin/chats` with query parameters for `status` (active/archived), pagination, and CST date filtering.
    * Build `GET /admin/analytics` to return aggregated totals for the KPI cards (e.g., "Cannot Find Info" trigger count) and chart data.
    * *Testing:* Validate payload shapes and CST date filtering accuracy.
* **Step 2.3: Syllabus Management API Updates**
    * Extend the existing upload queue logic to accept the `is_hidden` flag.
    * Implement the citation masking logic: If a source is flagged as hidden, mask the citation text output to the frontend (student view) as "Source: Instructor Practice Material", but preserve the real filename for the admin API.
    * *Testing:* Verify that hidden filenames are completely stripped from student-facing API responses.

## Phase 3: Frontend Foundation & Shared UI (Next.js)
**Goal:** Setup the UI shell, ensuring it dynamically respects the user's previously selected theme (Light/Dark mode) rather than forcing a strict dark mode.

* **Step 3.1: Admin Shell & Theme Integration**
    * Create the Admin layout wrapper with a left navigation sidebar.
    * Integrate the purely graphical, textless "My Tutor Bot" logo.
    * Hook the dashboard into the existing Next.js ThemeProvider to seamlessly support both Light and Dark modes based on the user's global preference.
* **Step 3.2: Reusable Chart Components**
    * Integrate a charting library (e.g., Recharts or Chart.js).
    * Build the `HorizontalBarChart` (for topics) and `DonutChart` (for modes) components.
    * Style them using CSS variables to ensure they adapt to both Light and Dark themes.
* **Step 3.3: Reusable Data Table & KPI Cards**
    * Build a responsive, sortable Data Table component for the Students and Syllabus tabs.
    * Build the KPI Number Card components with indicator badges.

## Phase 4: Frontend Tabs Implementation
**Goal:** Wire the UI components to the backend APIs, one tab at a time.

* **Step 4.1: Dashboard Tab**
    * Assemble the KPI cards and chart components. Fetch data from `/admin/analytics`.
* **Step 4.2: Chats Tab (Live Feed)**
    * Build the two-pane layout (left feed, right transcript inspector).
    * Implement infinite scroll/pagination for the feed.
    * Add the "Active | Archived" toggle and wire up the Archive action.
* **Step 4.3: Syllabus Content Tab**
    * Integrate the drag-and-drop batch upload UI.
    * Display the upload queue progress.
    * Render the active materials table with the "Hidden/Visible" toggle.
* **Step 4.4: Students & Research Surveys Tabs**
    * Build the anonymized roster table.
    * Implement the CSV upload zone for Google Forms survey data mapping.
* **Step 4.5: Admin Settings Tab**
    * Build the course configuration and danger zone actions (e.g., purge vector DB).

## Phase 5: Export & Polish
**Goal:** Finalize research data export capabilities and ensure production readiness.

* **Step 5.1: Data Export (CSV/JSON)**
    * Implement frontend buttons that trigger full-dataset downloads from the backend.
    * *Testing:* Ensure exported CSVs strictly maintain anonymity (FERPA) and Chicago CST formatting.
* **Step 5.2: Image Export**
    * Implement client-side rendering captures (e.g., html2canvas) to let users download charts as high-res PNGs for posters.
* **Step 5.3: End-to-End Testing**
    * Perform a full user journey test: upload a hidden PDF, ask a question triggering it, verify the student sees the masked citation, and verify the admin sees the full transcript in the Chats tab.
