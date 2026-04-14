# Implementation Plan: Task Manager App

## Overview

Implement a full-stack task manager application consisting of a Python RESTful API backend and a React (Vite) frontend. The backend covers user authentication (registration, login, logout, session management) and full task lifecycle (create, list, detail, edit, status transitions, delete). The frontend covers routing, session persistence via `localStorage`, form validation, and responsive UI. The backend uses Hypothesis for property-based testing; the frontend uses Vitest + React Testing Library.

## Tasks

- [ ] 1. Set up project structure, dependencies, and core data models
  - Create project directory layout: `app/`, `app/models/`, `app/repositories/`, `app/services/`, `app/routes/`, `tests/`
  - Define `User`, `Session`, and `Task` data models (SQLAlchemy or dataclasses) matching the design schemas
  - Define `Status` and `Priority` enums: `Pending`, `In Progress`, `Completed`; `Low`, `Medium`, `High`
  - Set up database connection, migrations (e.g., Alembic), and test database configuration
  - Define the uniform error response structure `{ "error_code": "...", "message": "..." }`
  - _Requirements: 1.1, 2.3, 3.1, 9.2_

- [ ] 2. Implement Task_Registry (persistence layer)
  - [ ] 2.1 Implement `UserRepository` with `create`, `find_by_email`, `find_by_id`
    - Write SQL/ORM queries for each method
    - _Requirements: 1.1, 1.2, 2.1_

  - [ ]* 2.2 Write property test for `UserRepository` — Property 1: Valid registration creates a user
    - **Property 1: Valid registration creates a user**
    - Generate random valid emails and passwords ≥ 8 chars, verify `create` persists the user and `find_by_email` returns it
    - **Validates: Requirements 1.2**

  - [ ] 2.3 Implement `TaskRepository` with `create`, `find_by_id`, `find_all_by_user`, `update`, `delete`
    - Write SQL/ORM queries; `find_all_by_user` must scope results to a single `user_id`
    - _Requirements: 3.5, 4.1, 5.1, 6.1, 8.1_

  - [ ]* 2.4 Write property test for `TaskRepository` — Property 11: Tasks are associated with their creating user
    - **Property 11: Tasks are associated with their creating user**
    - Generate random users and tasks, verify `task.user_id` equals the creating user's id and `find_all_by_user` returns only that user's tasks
    - **Validates: Requirements 3.5**

  - [ ]* 2.5 Write property test for `TaskRepository` — Property 12: Task list returns all and only the user's tasks
    - **Property 12: Task list returns all and only the user's tasks**
    - Generate random users with random task sets, verify `find_all_by_user` returns exactly those tasks with no extras or omissions
    - **Validates: Requirements 4.1**

- [ ] 3. Implement Auth_Service — registration and password hashing
  - [ ] 3.1 Implement user registration logic
    - Validate email uniqueness (return `EMAIL_ALREADY_IN_USE` / 409 on duplicate)
    - Validate password length ≥ 8 (return `PASSWORD_TOO_SHORT` / 422 on failure)
    - Hash password using `bcrypt` or `argon2` before storing; never store plaintext
    - Return `201 Created` on success
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

  - [ ]* 3.2 Write property test for registration — Property 2: Duplicate email is rejected
    - **Property 2: Duplicate email is rejected**
    - Generate random email, register twice, verify second returns `EMAIL_ALREADY_IN_USE`
    - **Validates: Requirements 1.3**

  - [ ]* 3.3 Write property test for registration — Property 3: Short password is rejected
    - **Property 3: Short password is rejected**
    - Generate passwords of length 0–7, verify registration returns `PASSWORD_TOO_SHORT`
    - **Validates: Requirements 1.4**

  - [ ]* 3.4 Write property test for registration — Property 4: Password is never stored as plaintext
    - **Property 4: Password is never stored as plaintext**
    - Generate random passwords, register, verify stored `password_hash` ≠ plaintext password
    - **Validates: Requirements 1.5**

  - [ ] 3.5 Expose `POST /auth/register` route wired to registration logic
    - Parse `{ email, password }` request body; return structured error on malformed input (`MALFORMED_REQUEST` / 400)
    - _Requirements: 1.1, 9.1, 9.2_

- [ ] 4. Implement Auth_Service — login, logout, and session management
  - [ ] 4.1 Implement login logic
    - Verify credentials; return `INVALID_CREDENTIALS` / 401 for wrong email or wrong password (same response for both)
    - On success, generate an opaque random session token, persist `Session` with `last_active_at`, return `{ session_token }`
    - _Requirements: 2.1, 2.2_

  - [ ]* 4.2 Write property test for login — Property 5: Valid login returns a session token
    - **Property 5: Valid login returns a session token**
    - Generate random users, register + login, verify a non-empty token is returned
    - **Validates: Requirements 2.1**

  - [ ]* 4.3 Write property test for login — Property 6: Login failure response is ambiguous
    - **Property 6: Login failure response is ambiguous**
    - Generate random credentials, attempt login with wrong email and wrong password separately, verify both return identical `INVALID_CREDENTIALS` error
    - **Validates: Requirements 2.2**

  - [ ] 4.4 Implement session validation middleware
    - On each protected request, look up the session token, check `last_active_at` expiry (24h), update `last_active_at` on valid sessions
    - Return `AUTHENTICATION_REQUIRED` / 401 for missing/invalid tokens; `SESSION_EXPIRED` / 401 for expired tokens
    - _Requirements: 2.3, 2.5, 9.3, 9.4_

  - [ ] 4.5 Implement logout logic and expose `POST /auth/logout` route
    - Invalidate the session token immediately on logout
    - Expose `POST /auth/login` and `POST /auth/logout` routes wired to their respective logic
    - _Requirements: 2.4_

  - [ ]* 4.6 Write property test for logout — Property 7: Logout invalidates the session token
    - **Property 7: Logout invalidates the session token**
    - Generate random users, login → logout → use token on a protected endpoint, verify `AUTHENTICATION_REQUIRED` is returned
    - **Validates: Requirements 2.4**

  - [ ]* 4.7 Write unit tests for session expiry and error responses
    - Test expired session with mocked clock returns `SESSION_EXPIRED`
    - Test unauthenticated request to protected endpoint returns `AUTHENTICATION_REQUIRED`
    - _Requirements: 2.5, 9.3, 9.4_

- [ ] 5. Checkpoint — Ensure all auth tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Implement Task_Service — task creation
  - [ ] 6.1 Implement task creation logic
    - Require non-empty title (return `TITLE_REQUIRED` / 422 if missing or empty)
    - Validate due date is today or future (return `DUE_DATE_IN_PAST` / 422 for past dates)
    - Accept optional `description`, `due_date`, `priority`
    - Set `status` to `Pending` on creation; associate task with the authenticated user
    - Return the created task
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

  - [ ]* 6.2 Write property test for task creation — Property 8: Newly created tasks have Pending status
    - **Property 8: Newly created tasks have Pending status**
    - Generate random task titles and optional fields, create tasks, verify `status == Pending`
    - **Validates: Requirements 3.1**

  - [ ]* 6.3 Write property test for task creation — Property 9: Optional task fields are accepted and stored correctly
    - **Property 9: Optional task fields are accepted and stored correctly**
    - Generate random combinations of optional fields, create tasks, verify fields are stored and returned correctly
    - **Validates: Requirements 3.2**

  - [ ]* 6.4 Write property test for task creation — Property 10: Past due dates are rejected
    - **Property 10: Past due dates are rejected**
    - Generate random past dates, attempt task creation, verify `DUE_DATE_IN_PAST` error
    - **Validates: Requirements 3.4**

  - [ ] 6.5 Expose `POST /tasks` route wired to task creation logic
    - Parse request body; return `MALFORMED_REQUEST` / 400 on malformed input
    - _Requirements: 3.1, 9.1_

- [ ] 7. Implement Task_Service — task listing with filters and sorting
  - [ ] 7.1 Implement task listing logic
    - Return all tasks for the authenticated user; return `[]` when user has no tasks
    - Support `status` filter (`Pending | In Progress | Completed`)
    - Support `priority` filter (`Low | Medium | High`)
    - Support `sort_by` (`due_date | created_at`) and `order` (`asc | desc`)
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6_

  - [ ]* 7.2 Write property test for task listing — Property 13: Filter correctness
    - **Property 13: Filter correctness**
    - Generate random task sets with mixed statuses/priorities, apply filters, verify every result matches the filter and no matching task is absent
    - **Validates: Requirements 4.2, 4.3**

  - [ ]* 7.3 Write property test for task listing — Property 14: Sort ordering invariant
    - **Property 14: Sort ordering invariant**
    - Generate random task sets, sort by `due_date` and `created_at` in both orders, verify every adjacent pair satisfies the ordering relation
    - **Validates: Requirements 4.4, 4.5**

  - [ ]* 7.4 Write unit test for empty task list
    - Verify `GET /tasks` returns `[]` when the user has no tasks
    - _Requirements: 4.6_

  - [ ] 7.5 Expose `GET /tasks` route wired to listing logic
    - Parse and validate query parameters; ignore unknown parameters
    - _Requirements: 4.1_

- [ ] 8. Implement Task_Service — task detail, edit, and deletion
  - [ ] 8.1 Implement task detail retrieval
    - Fetch task by id scoped to the authenticated user; return `TASK_NOT_FOUND` / 404 if not found or owned by another user
    - Return all fields: title, description, status, priority, due_date, created_at, updated_at
    - _Requirements: 5.1, 5.2, 5.3_

  - [ ]* 8.2 Write property test for task detail — Property 15: Task detail contains all fields
    - **Property 15: Task detail contains all fields**
    - Generate random tasks with all fields populated, fetch by id, verify all fields are present in the response
    - **Validates: Requirements 5.1**

  - [ ] 8.3 Implement task update logic
    - Allow updating `title`, `description`, `due_date`, `priority`
    - Validate non-empty title (return `TITLE_REQUIRED` / 422) and future due date (return `DUE_DATE_IN_PAST` / 422)
    - Record `updated_at` timestamp on every update
    - Return `TASK_NOT_FOUND` / 404 for tasks not owned by the requesting user
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6_

  - [ ]* 8.4 Write property test for task update — Property 17: Update round-trip
    - **Property 17: Update round-trip**
    - Generate random tasks and valid updates, apply update, verify returned task reflects exactly the updated values
    - **Validates: Requirements 6.1, 6.2**

  - [ ]* 8.5 Write property test for task update — Property 10 (update path): Past due dates are rejected on update
    - **Property 10 (update path): Past due dates are rejected**
    - Generate random past dates, attempt task update, verify `DUE_DATE_IN_PAST` error
    - **Validates: Requirements 6.4**

  - [ ] 8.6 Implement task deletion logic
    - Permanently remove the task; return `TASK_NOT_FOUND` / 404 if not found or owned by another user
    - Return success response on deletion
    - _Requirements: 8.1, 8.2, 8.3_

  - [ ]* 8.7 Write property test for task deletion — Property 20: Deletion removes the task permanently
    - **Property 20: Deletion removes the task permanently**
    - Generate random tasks, delete them, verify subsequent fetch returns `TASK_NOT_FOUND`
    - **Validates: Requirements 8.1**

  - [ ] 8.8 Expose `GET /tasks/:id`, `PUT /tasks/:id`, and `DELETE /tasks/:id` routes
    - Wire each route to its respective service logic
    - _Requirements: 5.1, 6.1, 8.1_

- [ ] 9. Implement Task_Service — status transitions
  - [ ] 9.1 Implement status update logic
    - Accept `PATCH /tasks/:id/status` with a new status value
    - Validate the transition is permitted (Pending→In Progress, Pending→Completed, In Progress→Completed, In Progress→Pending, Completed→Pending)
    - Return `INVALID_STATUS` / 422 for unrecognized status values (include valid values in message)
    - Return `INVALID_STATUS_TRANSITION` / 422 for disallowed transitions
    - Record `updated_at` timestamp on status change
    - Return `TASK_NOT_FOUND` / 404 for tasks not owned by the requesting user
    - _Requirements: 7.1, 7.2, 7.3, 7.4_

  - [ ]* 9.2 Write property test for status transitions — Property 19: Valid transitions accepted; invalid ones rejected
    - **Property 19: Valid status transitions are accepted; invalid ones are rejected**
    - For each permitted transition verify success and returned status; for each disallowed transition verify `INVALID_STATUS_TRANSITION`
    - **Validates: Requirements 7.1, 7.2**

  - [ ]* 9.3 Write unit test for invalid status string
    - Verify submitting an unrecognized status value returns `INVALID_STATUS` with valid values listed
    - _Requirements: 7.3_

- [ ] 10. Implement cross-cutting: task isolation and error response consistency
  - [ ] 10.1 Verify task isolation between users across all task endpoints
    - Ensure `GET /tasks/:id`, `PUT /tasks/:id`, `PATCH /tasks/:id/status`, `DELETE /tasks/:id` all return `TASK_NOT_FOUND` when the task belongs to a different user
    - _Requirements: 5.3, 6.5, 8.3_

  - [ ]* 10.2 Write property test for task isolation — Property 16: Task isolation between users
    - **Property 16: Task isolation between users**
    - Generate two random users, create tasks for user A, attempt read/update/delete as user B, verify `TASK_NOT_FOUND` for all operations
    - **Validates: Requirements 5.3, 6.5, 8.3**

  - [ ]* 10.3 Write property test for update timestamp — Property 18: Update timestamp is monotonically non-decreasing
    - **Property 18: Update timestamp is monotonically non-decreasing**
    - Generate random tasks, apply multiple updates, verify `updated_at` is ≥ previous `updated_at` after each change
    - **Validates: Requirements 6.6, 7.4**

  - [ ]* 10.4 Write property test for error response structure — Property 21: Error responses have a consistent structure
    - **Property 21: Error responses have a consistent structure**
    - Generate error-triggering inputs for all endpoints, verify every error response contains both `error_code` and `message` fields
    - **Validates: Requirements 9.1, 9.2**

  - [ ]* 10.5 Write property test for unauthenticated access — Property 22: Unauthenticated requests to protected endpoints are rejected
    - **Property 22: Unauthenticated requests to protected endpoints are rejected**
    - For each protected endpoint, make a request without a valid session token, verify `AUTHENTICATION_REQUIRED` is returned
    - **Validates: Requirements 9.3**

- [ ] 11. Final checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 12. Set up React frontend project structure
  - Scaffold a new Vite + React project inside `frontend/`
  - Install React Router v6 and axios (or configure fetch wrapper)
  - Create directory structure: `src/pages/`, `src/components/`, `src/api/`, `src/context/`
  - Set up `App.jsx` with `BrowserRouter` and placeholder routes for `/login`, `/register`, `/tasks`, `/tasks/new`, `/tasks/:id`
  - Create `ProtectedRoute` wrapper component that reads `AuthContext` and redirects to `/login` if unauthenticated
  - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5_

- [ ] 13. Implement authentication pages
  - [ ] 13.1 Implement `AuthForm` shared component
    - Render email and password fields, a submit button, and an `ErrorMessage` slot
    - Accept `onSubmit`, `error`, and `loading` props
    - _Requirements: 12.4, 12.5_

  - [ ] 13.2 Implement `LoginPage`
    - Wire `AuthForm` to `POST /auth/login` via `src/api/auth.js`
    - On success call `AuthContext.login(token)` and navigate to `/tasks`
    - Display API error messages returned by the backend
    - _Requirements: 10.1, 11.1_

  - [ ] 13.3 Implement `RegisterPage`
    - Wire `AuthForm` to `POST /auth/register` via `src/api/auth.js`
    - Validate password length ≥ 8 client-side before submitting; show inline error via `ErrorMessage`
    - On success navigate to `/login`
    - _Requirements: 10.2, 12.3_

  - [ ]* 13.4 Write unit tests for `LoginPage` and `RegisterPage`
    - Test that short password shows inline error and does not call the API
    - Test that empty email/password shows inline error
    - _Requirements: 12.3, 12.4_

- [ ] 14. Implement session management
  - [ ] 14.1 Implement `AuthContext` provider
    - On mount, read token from `localStorage`; expose `isAuthenticated`, `login(token)`, and `logout()`
    - `login()` writes token to `localStorage` and updates state
    - `logout()` removes token from `localStorage`, clears state, and navigates to `/login`
    - _Requirements: 11.1, 11.3_

  - [ ] 14.2 Implement API client with auth header injection
    - Create `src/api/client.js` that wraps fetch/axios; reads token from `AuthContext` and attaches `Authorization: Bearer <token>` on every request
    - _Requirements: 11.4_

  - [ ]* 14.3 Write property test for session token attachment — Property 24
    - **Property 24: Session token is attached to every protected API call**
    - Mock the HTTP client; for any token stored in context, verify every outgoing request includes the correct `Authorization` header
    - **Validates: Requirements 11.4**

  - [ ]* 14.4 Write property test for unauthenticated redirect — Property 23
    - **Property 23: Unauthenticated users are redirected to login**
    - For each protected route, render without a token in context, verify the router redirects to `/login`
    - **Validates: Requirements 11.2**

- [ ] 15. Implement task list page
  - [ ] 15.1 Implement `TaskCard` and `StatusBadge` components
    - `TaskCard` displays title, `StatusBadge`, priority, and due date
    - `StatusBadge` renders a colour-coded label for `Pending`, `In Progress`, `Completed`
    - _Requirements: 10.3_

  - [ ] 15.2 Implement `TaskListPage`
    - On mount, fetch tasks via `GET /tasks` using the API client
    - Render a list of `TaskCard` components; show empty-state message when list is empty
    - Add filter controls for status and priority and sort controls for due date / created date; re-fetch or filter client-side on change
    - _Requirements: 10.3, 13.1, 13.2_

  - [ ]* 15.3 Write unit tests for `TaskListPage`
    - Test empty state renders correctly
    - Test filter controls update the displayed list
    - _Requirements: 10.3_

- [ ] 16. Implement task create/edit form
  - [ ] 16.1 Implement `TaskForm` component
    - Render fields: title (required), description, due date, priority (select)
    - Validate title non-empty and due date not in the past before calling the API; show inline `ErrorMessage` per field
    - Accept `initialValues` and `onSubmit` props to support both create and edit modes
    - _Requirements: 12.1, 12.2, 12.5_

  - [ ] 16.2 Implement `TaskFormPage` (create mode)
    - Wire `TaskForm` to `POST /tasks`; on success navigate to `/tasks`
    - _Requirements: 10.5_

  - [ ]* 16.3 Write property test for form validation — Property 25
    - **Property 25: Form errors are shown before submission reaches the API**
    - For any combination of empty title or past due date, render `TaskForm`, submit, verify inline errors are shown and no API call is made
    - **Validates: Requirements 12.1, 12.2, 12.4, 12.5**

- [ ] 17. Implement task detail page and status update UI
  - [ ] 17.1 Implement `TaskDetailPage`
    - Fetch task via `GET /tasks/:id` on mount; render all fields using `StatusBadge`
    - Render status transition buttons for each valid next status; call `PATCH /tasks/:id/status` on click and refresh the displayed task
    - Render an "Edit" button that switches to inline `TaskForm` (edit mode) wired to `PUT /tasks/:id`
    - _Requirements: 10.4_

  - [ ]* 17.2 Write unit tests for `TaskDetailPage`
    - Test that only valid status transition buttons are rendered for each current status
    - Test that a successful status update refreshes the displayed status badge
    - _Requirements: 10.4_

- [ ] 18. Implement delete confirmation and final integration
  - [ ] 18.1 Add delete confirmation to `TaskDetailPage`
    - Render a "Delete" button; show a confirmation prompt before calling `DELETE /tasks/:id`
    - On success navigate back to `/tasks`
    - _Requirements: 10.4_

  - [ ] 18.2 Wire all pages together and handle API errors in UI
    - Ensure navigation links between pages are consistent (e.g., back to list from detail, "New Task" button on list page)
    - Display user-facing error messages for all API error responses (e.g., `TASK_NOT_FOUND`, `SESSION_EXPIRED`)
    - On `SESSION_EXPIRED` or `AUTHENTICATION_REQUIRED` response, call `AuthContext.logout()` to clear state and redirect to `/login`
    - _Requirements: 10.3, 10.4, 10.5, 11.2, 11.3_

  - [ ]* 18.3 Write property test for logout flow — Property 26
    - **Property 26: Logout clears the session and redirects**
    - Render the app with a token in context, trigger logout, verify `localStorage` is cleared, context token is null, and the router navigates to `/login`
    - **Validates: Requirements 11.3**

- [ ] 19. Final frontend checkpoint — Ensure all pages work end-to-end
  - Ensure all frontend tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Backend property-based tests use **Hypothesis** (Python); each test runs a minimum of 100 iterations
- Frontend property-based tests use **fast-check** with Vitest + React Testing Library
- Each property test must include a comment: `# Feature: task-manager-app, Property N: <property_text>` (backend) or `// Feature: task-manager-app, Property N: <property_text>` (frontend)
- `TASK_NOT_FOUND` is returned for both "doesn't exist" and "belongs to another user" to avoid leaking ownership information
- `INVALID_CREDENTIALS` does not distinguish between wrong email and wrong password to prevent user enumeration
- All 4xx and 5xx responses use the uniform `{ "error_code": "...", "message": "..." }` envelope
