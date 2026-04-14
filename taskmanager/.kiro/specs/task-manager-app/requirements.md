# Requirements Document

## Introduction

A task registration and management application that allows users to create, organize, track, and complete tasks. The application supports user authentication so each user has a private task list, and provides core task lifecycle management including creation, editing, status tracking, and deletion.

## Glossary

- **System**: The task manager application as a whole
- **User**: An authenticated person interacting with the application
- **Task**: A unit of work with a title, optional description, status, optional due date, and optional priority level
- **Task_Registry**: The component responsible for storing and retrieving tasks
- **Auth_Service**: The component responsible for user registration, login, and session management
- **Task_Service**: The component responsible for creating, updating, and deleting tasks
- **Session**: An authenticated user's active login context
- **Priority**: A classification of task urgency — one of: Low, Medium, High
- **Status**: The current state of a task — one of: Pending, In Progress, Completed

---

## Requirements

### Requirement 1: User Registration

**User Story:** As a new user, I want to register an account, so that I can have a personal task list.

#### Acceptance Criteria

1. THE Auth_Service SHALL provide a registration endpoint that accepts a unique email address and a password.
2. WHEN a user submits a registration request with a valid email and a password of at least 8 characters, THE Auth_Service SHALL create a new user account and return a success response.
3. IF a user submits a registration request with an email address that already exists, THEN THE Auth_Service SHALL return an error response indicating the email is already in use.
4. IF a user submits a registration request with a password shorter than 8 characters, THEN THE Auth_Service SHALL return an error response describing the password requirement.
5. THE Auth_Service SHALL store passwords as a cryptographic hash and SHALL NOT store plaintext passwords.

---

### Requirement 2: User Authentication

**User Story:** As a registered user, I want to log in to my account, so that I can access my tasks.

#### Acceptance Criteria

1. WHEN a user submits valid credentials, THE Auth_Service SHALL create a Session and return a session token to the user.
2. IF a user submits an unrecognized email or an incorrect password, THEN THE Auth_Service SHALL return an error response without indicating which field is incorrect.
3. WHILE a Session is active, THE System SHALL allow the user to perform task operations.
4. WHEN a user logs out, THE Auth_Service SHALL invalidate the Session token.
5. THE Auth_Service SHALL expire Session tokens after 24 hours of inactivity.

---

### Requirement 3: Task Creation

**User Story:** As an authenticated user, I want to create a new task, so that I can register work I need to do.

#### Acceptance Criteria

1. WHEN an authenticated user submits a task creation request with a title, THE Task_Service SHALL create a new Task with Status set to Pending and return the created Task.
2. THE Task_Service SHALL accept an optional description, optional due date, and optional Priority when creating a Task.
3. IF a task creation request is submitted without a title, THEN THE Task_Service SHALL return an error response indicating the title is required.
4. IF a task creation request is submitted with a due date in the past, THEN THE Task_Service SHALL return an error response indicating the due date must be today or a future date.
5. THE Task_Registry SHALL associate each created Task with the authenticated user who created it.

---

### Requirement 4: Task Listing

**User Story:** As an authenticated user, I want to view all my tasks, so that I can see what work is registered.

#### Acceptance Criteria

1. WHEN an authenticated user requests their task list, THE Task_Service SHALL return all Tasks associated with that user.
2. THE Task_Service SHALL support filtering the task list by Status.
3. THE Task_Service SHALL support filtering the task list by Priority.
4. THE Task_Service SHALL support sorting the task list by due date in ascending or descending order.
5. THE Task_Service SHALL support sorting the task list by creation date in ascending or descending order.
6. WHEN a user has no tasks, THE Task_Service SHALL return an empty list.

---

### Requirement 5: Task Detail View

**User Story:** As an authenticated user, I want to view the details of a specific task, so that I can review all its information.

#### Acceptance Criteria

1. WHEN an authenticated user requests a Task by its identifier, THE Task_Service SHALL return the full Task details including title, description, status, priority, due date, creation date, and last updated date.
2. IF a user requests a Task that does not exist, THEN THE Task_Service SHALL return an error response indicating the Task was not found.
3. IF a user requests a Task that belongs to a different user, THEN THE Task_Service SHALL return an error response indicating the Task was not found.

---

### Requirement 6: Task Editing

**User Story:** As an authenticated user, I want to edit an existing task, so that I can update its details as my work evolves.

#### Acceptance Criteria

1. WHEN an authenticated user submits an update request for a Task they own, THE Task_Service SHALL update the specified fields and return the updated Task.
2. THE Task_Service SHALL allow updating the title, description, due date, and Priority of a Task.
3. IF an update request sets the title to an empty value, THEN THE Task_Service SHALL return an error response indicating the title is required.
4. IF an update request sets the due date to a date in the past, THEN THE Task_Service SHALL return an error response indicating the due date must be today or a future date.
5. IF a user submits an update request for a Task that belongs to a different user, THEN THE Task_Service SHALL return an error response indicating the Task was not found.
6. THE Task_Service SHALL record the timestamp of the most recent update on the Task.

---

### Requirement 7: Task Status Management

**User Story:** As an authenticated user, I want to update the status of a task, so that I can track its progress.

#### Acceptance Criteria

1. WHEN an authenticated user submits a status update for a Task they own, THE Task_Service SHALL update the Task Status and return the updated Task.
2. THE Task_Service SHALL accept the following Status transitions: Pending → In Progress, Pending → Completed, In Progress → Completed, In Progress → Pending, Completed → Pending.
3. IF a user submits a status update with an unrecognized Status value, THEN THE Task_Service SHALL return an error response listing the valid Status values.
4. THE Task_Service SHALL record the timestamp of the status change on the Task.

---

### Requirement 8: Task Deletion

**User Story:** As an authenticated user, I want to delete a task, so that I can remove work that is no longer relevant.

#### Acceptance Criteria

1. WHEN an authenticated user submits a deletion request for a Task they own, THE Task_Service SHALL permanently remove the Task from the Task_Registry and return a success response.
2. IF a user submits a deletion request for a Task that does not exist, THEN THE Task_Service SHALL return an error response indicating the Task was not found.
3. IF a user submits a deletion request for a Task that belongs to a different user, THEN THE Task_Service SHALL return an error response indicating the Task was not found.

---

### Requirement 9: Input Validation and Error Handling

**User Story:** As a user, I want clear error messages when I provide invalid input, so that I can correct my requests.

#### Acceptance Criteria

1. WHEN the System receives a request with malformed data, THE System SHALL return an error response with a human-readable message describing the issue.
2. THE System SHALL return error responses using consistent, structured error objects containing an error code and a message.
3. IF an unauthenticated request is made to a protected endpoint, THEN THE System SHALL return an error response indicating authentication is required.
4. IF an authenticated request is made with an expired Session token, THEN THE Auth_Service SHALL return an error response indicating the session has expired.

---

### Requirement 10: UI Navigation

**User Story:** As a user, I want a clear navigation structure in the frontend, so that I can move between pages without confusion.

#### Acceptance Criteria

1. THE frontend SHALL provide a login page at the `/login` route.
2. THE frontend SHALL provide a registration page at the `/register` route.
3. THE frontend SHALL provide a task list page at the `/tasks` route, accessible only to authenticated users.
4. THE frontend SHALL provide a task detail and edit page at the `/tasks/:id` route, accessible only to authenticated users.
5. THE frontend SHALL provide a task creation form at the `/tasks/new` route, accessible only to authenticated users.

---

### Requirement 11: Session Persistence

**User Story:** As a returning user, I want my session to persist across page refreshes, so that I do not have to log in every time I open the app.

#### Acceptance Criteria

1. WHEN a user successfully logs in, THE frontend SHALL store the session token in `localStorage`.
2. WHEN a user navigates to a protected route without a valid session token in `localStorage`, THE frontend SHALL redirect the user to the login page.
3. WHEN a user logs out, THE frontend SHALL remove the session token from `localStorage` and redirect the user to the login page.
4. THE frontend SHALL include the session token as an `Authorization: Bearer <token>` header on every API request to a protected endpoint.

---

### Requirement 12: Frontend Validation

**User Story:** As a user, I want to see inline error messages when I fill out forms incorrectly, so that I can fix mistakes before submitting.

#### Acceptance Criteria

1. IF a user submits the task creation or edit form with an empty title, THE frontend SHALL display an inline error message indicating the title is required.
2. IF a user submits the task creation or edit form with a due date in the past, THE frontend SHALL display an inline error message indicating the due date must be today or a future date.
3. IF a user submits the registration form with a password shorter than 8 characters, THE frontend SHALL display an inline error message describing the password requirement.
4. IF a user submits the login or registration form with an empty email or password field, THE frontend SHALL display an inline error message indicating the field is required.
5. THE frontend SHALL display inline validation errors adjacent to the relevant form field without navigating away from the page.

---

### Requirement 13: Responsive Layout

**User Story:** As a user, I want the application to be usable on both desktop and mobile devices, so that I can manage my tasks from any device.

#### Acceptance Criteria

1. THE frontend layout SHALL be usable on desktop screen widths (≥ 1024px) and mobile screen widths (≤ 480px).
2. THE task list page SHALL display task cards in a single-column layout on mobile and may use a wider layout on desktop.
3. Form inputs and buttons SHALL be large enough to be tappable on a mobile touchscreen.
4. THE frontend SHALL NOT require horizontal scrolling on any supported screen width.
