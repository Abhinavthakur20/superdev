# Task Tracker

A full-stack task management application with search, filtering, and pagination. The frontend is a React single-page application and the backend is a Spring Boot REST API backed by an H2 in-memory database.

This repository contains a completed patch exercise. The codebase had several intentional bugs across the frontend, backend, and SQL layers. The fixes applied are documented below and in `NOTES.md`.

---

## Tech Stack

| Layer      | Technology                                  |
| ---------- | ------------------------------------------- |
| Frontend   | React 18, Vite 5, JavaScript (ES modules)  |
| Backend    | Spring Boot 3.2, Java 17, Spring Data JPA  |
| Database   | H2 (in-memory, auto-initialized via SQL)   |
| SQL Ref    | Oracle PL/SQL reference artifact in `db/`  |
| Build      | Maven Wrapper (backend), npm (frontend)    |

---

## Project Structure

```
.
├── backend/
│   ├── src/main/java/com/internal/tasktracker/
│   │   ├── TaskTrackerApplication.java   # Spring Boot entry point
│   │   ├── TaskController.java           # REST controller (GET /api/tasks)
│   │   ├── TaskRepository.java           # JPA repository with native SQL query
│   │   ├── Task.java                     # JPA entity
│   │   └── TaskStatus.java              # Status enum (OPEN, IN_PROGRESS, DONE)
│   ├── src/main/resources/
│   │   ├── application.properties        # Server, H2, and JPA configuration
│   │   ├── schema.sql                    # Table creation script
│   │   └── data.sql                      # Seed data
│   ├── pom.xml
│   └── mvnw / mvnw.cmd                   # Maven Wrapper
├── frontend/
│   ├── src/
│   │   ├── App.jsx                       # Root component with search, filter, pagination
│   │   ├── api.js                        # API client (fetch wrapper)
│   │   ├── main.jsx                      # React entry point
│   │   ├── styles.css                    # Application styles
│   │   ├── components/
│   │   │   ├── SearchBar.jsx             # Text search input
│   │   │   ├── StatusFilter.jsx          # Status dropdown filter
│   │   │   └── TaskTable.jsx             # Task list table with loading/error states
│   │   └── hooks/
│   │       ├── useTasks.js               # Data-fetching hook with race condition handling
│   │       └── useDebounce.js            # Debounce hook for search input
│   ├── index.html
│   ├── vite.config.js                    # Dev server and API proxy configuration
│   └── package.json
├── db/
│   ├── queries/search_tasks.sql          # H2-compatible reference query
│   └── oracle/task_search_package.sql    # Oracle PL/SQL reference artifact
├── handwritten/                          # Handwritten bug explanations (photos)
├── NOTES.md                              # Detailed patch notes and tradeoff reasoning
└── README.md
```

---

## Prerequisites

- **Java 17+** — verify with `java --version`
- **Node.js 18+** — verify with `node --version`
- No Docker or external database required. H2 runs in-memory with zero setup.

---

## Getting Started

### Backend

```bash
cd backend
mvnw.cmd spring-boot:run
```

On macOS/Linux, use `./mvnw spring-boot:run` instead.

The API starts on **http://localhost:8080**.

### Frontend

```bash
cd frontend
npm install
npm run dev
```

The app starts on **http://localhost:5173**.

The Vite dev server proxies all `/api/*` requests to `http://localhost:8080` automatically.

### Run Order

1. Start the backend first (port 8080).
2. Start the frontend second (port 5173).
3. Open **http://localhost:5173** in your browser.

---

## API Overview

The backend exposes a single endpoint:

### `GET /api/tasks`

| Parameter  | Type   | Default | Description                          |
| ---------- | ------ | ------- | ------------------------------------ |
| `q`        | string | `""`    | Search term (matches title and description, case-insensitive) |
| `status`   | string | —       | Filter by status: `OPEN`, `IN_PROGRESS`, or `DONE` |
| `page`     | int    | `1`     | Page number (1-indexed)              |
| `pageSize` | int    | `10`    | Number of results per page           |

**Response format:**

```json
{
  "items": [ ... ],
  "total": 25,
  "page": 1,
  "pageSize": 10
}
```

**Example requests:**

```bash
# All tasks (first page)
curl "http://localhost:8080/api/tasks"

# Search by term
curl "http://localhost:8080/api/tasks?q=api"

# Filter by status
curl "http://localhost:8080/api/tasks?status=OPEN"

# Combined search, filter, and pagination
curl "http://localhost:8080/api/tasks?q=api&status=OPEN&page=1&pageSize=5"
```

---

## Database

- **Engine:** H2 in-memory database, initialized automatically on startup.
- **Schema:** Defined in `backend/src/main/resources/schema.sql`.
- **Seed data:** Loaded from `backend/src/main/resources/data.sql`.
- **DDL mode:** `spring.jpa.hibernate.ddl-auto=none` (schema managed by SQL scripts, not Hibernate).

### H2 Console

The H2 web console is enabled and accessible while the backend is running:

| Setting   | Value                |
| --------- | -------------------- |
| URL       | http://localhost:8080/h2-console |
| JDBC URL  | `jdbc:h2:mem:taskdb` |
| Username  | `sa`                 |
| Password  | *(leave blank)*      |

### SQL Reference Files

- `db/queries/search_tasks.sql` — H2-compatible version of the search query used by the repository layer.
- `db/oracle/task_search_package.sql` — Oracle PL/SQL package that mirrors the application's search logic. This is a reference artifact and does not run locally.

---

## Key Bug Fixes

Five bugs were identified and fixed across the backend, frontend, and SQL layers. A brief summary is provided here; see `NOTES.md` for detailed patch notes and tradeoff reasoning.

### 1. SQL AND/OR Operator Precedence

- **Problem:** The `WHERE` clause in the search query lacked parentheses around the `(title LIKE ... OR description LIKE ...)` conditions. Due to SQL operator precedence (`AND` binds before `OR`), this could leak archived records or bypass the status filter depending on which column matched.
- **Fix:** Added explicit parentheses in `TaskRepository.java`, `search_tasks.sql`, and `task_search_package.sql`.
- **Impact:** Search results now correctly respect both the `archived` flag and the status filter in all cases.

### 2. Frontend Infinite Loading on API Error

- **Problem:** The `useTasks` hook did not reset the `loading` state on failed API requests. If the backend was unreachable, the UI would show "Loading tasks..." indefinitely instead of displaying the error.
- **Fix:** Added `.finally()` to the fetch promise chain to guarantee `loading` resets to `false`. Stale errors are cleared at the start of each new request.
- **Impact:** API failures now surface an error message to the user instead of an infinite loading state.

### 3. Stale Response Race Condition

- **Problem:** Rapid changes to the search query or status filter could cause out-of-order async responses to overwrite newer results with stale data.
- **Fix:** Implemented an `ignore` flag in the `useEffect` cleanup function inside `useTasks.js`. Stale responses are discarded when the effect is re-triggered.
- **Impact:** The displayed results always correspond to the most recent user input.

### 4. Pagination Trap on Filter/Search Change

- **Problem:** Changing the search query or status filter while on page > 1 could leave the user stranded on an empty page if the new result set had fewer total pages.
- **Fix:** Reset the page to `1` in `App.jsx` whenever the search query or status filter changes.
- **Impact:** Users always see results when changing filters, instead of landing on a nonexistent page.

### 5. Artificial Backend Thread.sleep() Blocking

- **Problem:** `TaskController.java` contained a `Thread.sleep()` call that artificially delayed every API response, blocking the HTTP request thread.
- **Fix:** Removed the `Thread.sleep()` call.
- **Impact:** API responses are no longer artificially delayed. The HTTP thread pool is not unnecessarily blocked.

---

## Validation

- The application was manually tested end-to-end: backend startup, frontend startup, search, status filtering, pagination, and error handling.
- API endpoints were verified directly via browser and `curl`.
- The frontend production build was verified with `npm run build`.
- No automated test suite is included in this repository.

---

## Notes

Detailed patch decisions, tradeoff reasoning for what was intentionally left unchanged, the biggest remaining risk in the codebase, and AI/tool usage are documented in `NOTES.md`.

Handwritten explanations for each bug (location, discovery, root cause, and fix approach) are included as photographs in the `handwritten/` folder.

---

## Submission

This repository contains the completed full-stack patch exercise. The original startup commands, folder structure, and project configuration are preserved.
