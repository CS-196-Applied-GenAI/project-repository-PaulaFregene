# UI/UX Wireframe & Spec: Chats Tab (Live Feed)

## 1. Visual Layout
**Style:** Dark Mode interface. Textless, purely graphical 'My Tutor Bot' logo in the top-left sidebar.

### Top Navigation & Filters
* **Toggle Group:** [ Active ] | [ Archived ]
* **Dropdown:** [ Date Range (Chicago CST) ]
* **Dropdown:** [ Interaction Mode: All | Socratic | Direct Answer ]
* **Search Bar:** [ Search transcripts or anonymized IDs... ]

### Main View (Two-Pane Layout)
**Left Pane: Live Scrolling Feed (30% width)**
* Continuously updates as students ask questions.
* Each card in the feed displays:
  * **Anonymized ID:** (e.g., Student_A7X9)
  * **Timestamp:** (e.g., 10:42 AM CST)
  * **Preview:** First 40 characters of the question.
  * **Badges:** * 🔵 Socratic Mode (or) 🟢 Direct Answer
    * ⚠️ "Cannot Find Info" (Red warning if triggered)
    * ✅ Resolved (or) ❓ Confused

**Right Pane: Active Transcript Inspector (70% width)**
* Appears when a card is clicked from the left feed.
* Displays the full, anonymized chat transcript.
* **Top Right Actions:** * [ Archive Chat ] button.
  * [ Export Thread as CSV ] button.
* **Metadata Sidebar (inside right pane):**
  * Turn Count: 8
  * Cited Material: "Week 3 - Memory Allocation.pdf"
  * Code Snippet Detected: Yes (C++ Pointers)
