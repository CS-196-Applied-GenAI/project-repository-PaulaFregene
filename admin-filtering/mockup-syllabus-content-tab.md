# UI/UX Wireframe & Spec: Syllabus Content Tab

## 1. Visual Layout
**Style:** Dark Mode interface. Textless logo in the header.

### Top Section: Batch Upload Queue
* **Drag & Drop Zone:** A large, dashed-border area titled "Drop Instructor PDFs & Slides Here".
* **Upload Queue List:**
  * Shows files currently processing with a progress bar.
  * Uses the existing Python upload queue logic.
  * E.g., `lec04-pointers.pdf` - [|||||||||| 100%] - Ready

### Bottom Section: Active Knowledge Base (Data Table)
* Displays all currently ingested materials available to the bot.
* **Columns:**
  * **Filename:** (e.g., `practice-exam-rubric.pdf`)
  * **Date Uploaded:** (e.g., Oct 14, 2026 2:00 PM CST)
  * **Visibility Status:**
    * 👁️ **Visible** (Standard cited material)
    * 🕵️ **Hidden** (Pedagogical guide/practice answers - citation masked to students as "Source: Instructor Practice Material")
  * **Quick Actions:**
    * [ Toggle Visibility ] 
    * [ Delete / Purge from Bot ]
