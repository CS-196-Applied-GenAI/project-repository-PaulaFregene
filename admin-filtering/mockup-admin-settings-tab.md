# UI/UX Wireframe & Spec: Admin Settings Tab

## 1. Visual Layout
**Style:** Dark Mode interface. Textless logo in the header.

### Section 1: Role-Based Access Control (RBAC)
* Connected directly to Clerk authentication.
* **List of Approved Admins/Researchers:**
  * Displays the four whitelisted email addresses.
  * Shows Clerk Role: "Admin" or "Researcher".
* **Action:** [ Manage Users in Clerk Dashboard ] (External link)

### Section 2: Course Configuration
* **Active Course Database:** * Dropdown set strictly to `Comp Sci 211` (Future-proofing for multi-course expansion).
* **System Timezone Override:**
  * Status indicator: 🟢 Locked to Chicago CST.
  
### Section 3: Danger Zone
* **Action:** [ Purge All Archived Chats ]
* **Action:** [ Reset Vector Database ] (Requires double confirmation)
