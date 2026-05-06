# Feature Specification: Research & Admin Visualization Dashboard

## 1. Overview
An expansion of the existing admin interface for "My Tutor Bot" designed to provide high-level research visualizations, real-time operational tracking, and deep-dive qualitative analysis tools. The dashboard allows instructors and researchers to analyze aggregated student interactions, identify common conceptual hurdles in CS 211, and evaluate the effectiveness of AI tutoring modes without compromising student privacy or the strict boundaries of the underlying LLM.

## 2. Technical Stack & UI Guidelines
* **Frontend:** Next.js.
* **Backend/Data:** Python, PostgreSQL.
* **UI/UX:** The dashboard must maintain strict dark mode compatibility. The top navigation bar will utilize the purely graphical, textless logo to ensure a clean, highly scalable, and professional appearance.
* **Timezone:** All database queries, UI filters, chat timestamps, and export data must strictly operate in and display the **Chicago CST** timezone.

## 3. Core Architecture: Topic Clustering & AI Isolation
* **Background Processing:** The system will utilize LLM embeddings to process and cluster student chat logs into "Trending Topics" (e.g., pointers, dynamic memory, segmentation faults).
* **Isolation Constraint:** This embedding and clustering process must happen asynchronously on the backend *after* the conversation. The student-facing AI agent must remain completely isolated and continue to strictly rely *only* on the uploaded syllabus PDFs to prevent any risk of hallucination.

## 4. Main Dashboard Visualizations (Aggregated Snapshot)
The main view will provide a broad, quantitative summary of the past 5 months of data.
* **KPI Number Cards:** Highlight critical baseline metrics, most notably the total count of times the bot responded with: *"I cannot find this information in the provided course content."*
* **Horizontal Bar Chart:** Ranks the top 10 most frequent conceptual questions or error types based on the embedding clusters.
* **Donut Chart:** Displays the proportion of interactions utilizing **Socratic Mode** versus **Direct Answer Mode**.
* **Practice Question Frequency:** A dedicated visual widget mapping the frequency of students asking for "practice questions" against the specific course topics requested.

## 5. Chats Tab (Live Feed & History)
A dedicated interface for real-time operational tracking of student queries.
* **Feed UI:** A continuous, scrolling list displaying incoming student messages in real-time.
* **State Management:** Features an "Active" vs. "Archived" toggle. Conversations remain in the active view until manually archived by an admin.
* **History Access:** The UI supports deep scrolling to view the latest 100+ conversations natively in the browser across both active and archived states.
* **Filtering:** Includes robust date range filters (Chicago CST) and interaction mode toggles.

## 6. Syllabus Content Management
The control center for the bot's RAG (Retrieval-Augmented Generation) knowledge base.
* **Upload Queue:** Maintains the existing batch-upload queue logic for processing multiple instructor PDFs simultaneously.
* **Hidden vs. Visible Content:** Admins can tag uploaded materials as "Hidden" (e.g., grading rubrics, explicit practice exam solutions). The bot can use hidden content to formulate answers and guide students.
* **Citation Handling:** * *Frontend (Student View):* If the bot uses hidden content, it will dynamically mask the citation and display: *"Source: Instructor Practice Material"* to maintain trust without exposing the file.
    * *Backend (Admin View):* The actual filename of the hidden document is preserved in the database and fully visible in the admin drill-down tables.

## 7. Drill-Down View (Qualitative Data Table)
Clicking on any chart element (e.g., a specific topic bar) triggers a drill-down into a data table displaying the underlying raw logs.
* **Anonymization:** All queries fetching these logs must strip personally identifiable information (FERPA compliant) before rendering.
* **Table Columns:**
    * **Chat Transcript:** The raw text of the conversation.
    * **Cited Material:** The exact uploaded PDF document/slide the bot referenced (revealing hidden filenames to admins).
    * **Mode Switching:** A flag capturing if/when the student toggled between Socratic and Direct Answer modes during the session.
    * **Code Snippet Detected:** A boolean flag (derived via regex for C/C++ syntax or markdown blocks) indicating if code was shared.
    * **Resolution Flag:** An automated tag based on keyword matching (e.g., "idk", "why", "thanks") indicating if the session resolved successfully or ended in confusion.

## 8. Export Functionality
To support academic research and course auditing, the dashboard provides dual export capabilities:
* **Raw Data Export:** One-click download (CSV/JSON) of the currently filtered, anonymized dataset (inclusive of all archived/active records).
* **Visual Export:** UI buttons on all charts to export clean, high-resolution images of the graphs for direct use in research posters and academic papers.

## 9. Data Management & Security
* **Course Scope:** The database architecture will include a `course_id` column to future-proof the platform for broader adoption, defaulting strictly to CS 211 for current deployments.
* **Authentication (RBAC):** Access to the `/admin` routes and underlying data APIs is strictly protected via Clerk authentication. Access is manually restricted via Clerk roles to a predefined whitelist of four approved administrators/researchers.
