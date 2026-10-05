# Engineering Notes

### Summary of Changes
- **SQL Precedence Fix (`TaskRepository.java`, `search_tasks.sql`, `task_search_package.sql`):** Added parentheses around the `(title OR description)` LIKE conditions. This prevents `AND/OR` operator precedence from leaking archived records when matching description and bypassing status filtering when matching title.
- **Frontend Error & Loading State (`useTasks.js`):** Added `.finally()` to guarantee `loading` resets to `false` on both success and failure, ensuring API errors are displayed rather than indefinitely showing a loading message. Cleared stale errors on new request triggers.
- **Frontend Race Condition (`useTasks.js`):** Implemented an `ignore` flag in `useEffect` cleanup so out-of-order/stale asynchronous responses do not overwrite the latest user query results.
- **Search Debounce (`useDebounce.js`, `App.jsx`):** Added a 300ms debounce hook to eliminate rapid keystroke layout flickering and unnecessary API calls.
- **Pagination State Reset (`App.jsx`):** Reset active page to 1 when changing search query or status filter to prevent trapping users on empty, nonexistent pages.
- **Removed Artificial Latency (`TaskController.java`):** Removed `Thread.sleep()` blocking query delay on the HTTP thread.

### What Was Not Changed & Why
- **In-Memory Pagination (`subList`):** Retained current in-memory slicing because the dataset is small and refactoring to database-level `Pageable` would expand the diff beyond the focused patch scope.
- **Controller Input Validation:** Left default parameter fallback behavior intact to avoid speculative API contract shifts.

### Biggest Remaining Risk
The backend loads all matching database records into application memory before slicing pagination with `List.subList()`. As the dataset grows, this will cause memory pressure, GC latency, and potential `OutOfMemoryError`. It should eventually be replaced with SQL `LIMIT`/`OFFSET` via Spring Data `Pageable`.

### AI & Tools Used
Used Claude / AI assistant for initial repository inspection, operator precedence verification in SQL/PL-SQL, drafting the debounce hook, and verifying build artifacts.
