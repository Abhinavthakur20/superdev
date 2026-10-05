# Engineering Notes

### Summary of Changes & Improvements
- **SQL Precedence Fix (`TaskRepository.java`, `search_tasks.sql`, `task_search_package.sql`):** Added parentheses around `(LOWER(title) LIKE :term OR LOWER(description) LIKE :term)`. Without grouping, SQL `AND` precedence leaked archived records on description matches and bypassed status filtering on title matches.
- **Frontend Error & Loading State (`useTasks.js`):** Added `.finally()` to guarantee `loading` resets to `false` on API failure so error messages display rather than hanging indefinitely. Cleared stale errors on new requests.
- **Async Race Condition Fix (`useTasks.js`):** Added an `ignore` flag in `useEffect` cleanup to discard stale/out-of-order responses from overwriting newer search results.
- **UX & Performance Debounce (`useDebounce.js`, `App.jsx`, `styles.css`):** Added a 300ms debounce hook on search input to prevent rapid API firing and input layout flicker. Added `overflow-y: scroll` in `styles.css` to prevent layout shift.
- **Pagination State Reset (`App.jsx`):** Reset page index to 1 on query or status filter change to prevent users from being trapped on empty pages.
- **Removed Artificial Latency (`TaskController.java`):** Removed `Thread.sleep()` blocking query delay on the servlet thread.

### Assumptions Made
- **API Contract:** Maintained 1-based page indexing and default 10-item page size to preserve compatibility with existing callers.
- **Search Semantics:** Substring matching across title and description is case-insensitive; an empty query returns all non-archived tasks.
- **Oracle PL/SQL:** Treated as an unexecuted production reference artifact and aligned its logic with H2 native query fixes.

### What Was Not Changed & Why
- **In-Memory Pagination (`subList`):** Kept in-memory slicing because the current dataset is small and refactoring to Spring Data `Pageable` would expand the diff beyond the focused patch scope.
- **Input Validation Fallback:** Preserved default parameter fallbacks to avoid breaking API expectations.

### Biggest Remaining Risk
The backend loads all matching database records into application memory before slicing pagination with `List.subList()`. As the dataset grows, this will cause memory pressure, GC latency, and potential `OutOfMemoryError`. It should be replaced with SQL-level `LIMIT`/`OFFSET` via Spring Data `Pageable`.

### AI & Tools Used
Used Claude / AI assistant for initial codebase inspection, SQL operator precedence verification across H2 and PL/SQL, drafting the debounce hook, and verifying build artifacts.
