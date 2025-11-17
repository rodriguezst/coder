# Tasks Functionality Implementation Report

**Version:** 1.0
**Date:** 2025-11-17
**Status:** Experimental API

This comprehensive report documents the Tasks functionality in Coder, providing detailed information for implementing Tasks support in alternative frontends (mobile apps, third-party integrations, etc.).

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [REST API Endpoints](#rest-api-endpoints)
4. [Data Models](#data-models)
5. [Task Lifecycle](#task-lifecycle)
6. [Frontend Implementation Guide](#frontend-implementation-guide)
7. [Mobile App Considerations](#mobile-app-considerations)
8. [Real-time Updates Strategy](#real-time-updates-strategy)
9. [Error Handling](#error-handling)
10. [Security & Permissions](#security--permissions)
11. [Code References](#code-references)

---

## Overview

### What are Tasks?

Tasks are **AI-powered workspaces** that provide automated development assistance in Coder. They allow users to:

- Create ephemeral workspaces from AI-enabled templates
- Interact with AI through natural language prompts
- Monitor workspace provisioning and AI task status
- View task outputs and results (file URIs, GitHub PRs, etc.)
- Delete tasks when complete

### Key Characteristics

- **Experimental API:** All endpoints are under `/api/experimental/tasks`
- **Workspace-backed:** Each task creates an associated Coder workspace
- **Template-based:** Tasks use template versions with `has_ai_task: true`
- **Stateful:** Tasks maintain conversation history (logs) between user and AI
- **Status-driven:** Task status computed from workspace build + agent + app health

---

## Architecture

### High-Level Components

```
┌─────────────────────────────────────────────────────────────┐
│                        Frontend Client                       │
│  (Web App / Mobile App / CLI)                               │
└────────────┬────────────────────────────────────────────────┘
             │ REST API (Polling)
             │ GET/POST/DELETE /api/experimental/tasks
             │
┌────────────▼────────────────────────────────────────────────┐
│                     Coder API Server                         │
│  (coderd/aitasks.go)                                        │
│  - Task CRUD operations                                     │
│  - Query filtering & search                                 │
│  - Status computation                                       │
└────────────┬────────────────────────────────────────────────┘
             │
             ├──► Database (PostgreSQL)
             │    - tasks table
             │    - task_workspace_apps table
             │    - tasks_with_status VIEW
             │
             ├──► Workspace Service
             │    - Workspace provisioning
             │    - Build management
             │    - Agent coordination
             │
             └──► AI Integration (via Workspace App)
                  - Task app receives prompts
                  - Communicates via agentapi
                  - Returns outputs/state updates
```

### Database Schema

#### 1. `tasks` Table

```sql
CREATE TABLE tasks (
  id                  UUID        PRIMARY KEY,
  organization_id     UUID        NOT NULL REFERENCES organizations(id),
  owner_id            UUID        NOT NULL REFERENCES users(id),
  name                TEXT        NOT NULL,
  workspace_id        UUID        REFERENCES workspaces(id),
  template_version_id UUID        NOT NULL REFERENCES template_versions(id),
  template_parameters JSONB       NOT NULL DEFAULT '{}',
  prompt              TEXT        NOT NULL,
  created_at          TIMESTAMPTZ NOT NULL,
  deleted_at          TIMESTAMPTZ
);

-- Indexes
CREATE INDEX tasks_workspace_id_idx ON tasks (workspace_id);
CREATE INDEX tasks_owner_id_idx ON tasks (owner_id);
CREATE INDEX tasks_organization_id_idx ON tasks (organization_id);
```

#### 2. `task_workspace_apps` Table

```sql
CREATE TABLE task_workspace_apps (
  task_id                UUID NOT NULL REFERENCES tasks(id),
  workspace_build_number INT  NOT NULL,
  workspace_agent_id     UUID REFERENCES workspace_agents(id),
  workspace_app_id       UUID REFERENCES workspace_apps(id),
  PRIMARY KEY (task_id, workspace_build_number)
);
```

#### 3. `tasks_with_status` VIEW

A complex SQL view that computes task status by joining:
- Task data
- Workspace build status
- Agent lifecycle state
- Workspace app health

**Status Computation Logic:**

```
IF no workspace EXISTS THEN
  status = 'pending'
ELSE IF build_status != 'active' THEN
  status = build_status  -- (pending, initializing, error, etc.)
ELSE IF agent_status != 'active' THEN
  status = agent_status
ELSE
  status = app_status
END IF
```

---

## REST API Endpoints

### Base URL

```
/api/experimental/tasks
```

### Authentication

All endpoints require authentication via:
- **Header:** `Coder-Session-Token: <token>`
- **Cookie:** `coder_session_token=<token>`

### Endpoint Summary

| Method   | Endpoint                                     | Description                      |
|----------|----------------------------------------------|----------------------------------|
| `GET`    | `/api/experimental/tasks`                    | List tasks with optional filters |
| `POST`   | `/api/experimental/tasks/{user}`             | Create new task                  |
| `GET`    | `/api/experimental/tasks/{user}/{task}`      | Get single task                  |
| `DELETE` | `/api/experimental/tasks/{user}/{task}`      | Delete task and workspace        |
| `POST`   | `/api/experimental/tasks/{user}/{task}/send` | Send additional input to task    |
| `GET`    | `/api/experimental/tasks/{user}/{task}/logs` | Get task conversation history    |

---

### 1. List Tasks

**Endpoint:** `GET /api/experimental/tasks`

**Query Parameters:**

| Parameter | Type   | Required | Description                                   |
|-----------|--------|----------|-----------------------------------------------|
| `q`       | string | No       | Query filter (e.g., `owner:me status:active`) |

**Filter Syntax:**

- `owner:<username|uuid|me>` - Filter by task owner
- `organization:<org-name|uuid>` - Filter by organization
- `status:<status>` - Filter by status (pending, active, etc.)

**Example Requests:**

```bash
# Get all tasks for current user
curl -H "Coder-Session-Token: $TOKEN" \
  "https://coder.example.com/api/experimental/tasks?q=owner:me"

# Get active tasks
curl -H "Coder-Session-Token: $TOKEN" \
  "https://coder.example.com/api/experimental/tasks?q=status:active"

# Combine filters
curl -H "Coder-Session-Token: $TOKEN" \
  "https://coder.example.com/api/experimental/tasks?q=owner:me%20status:active"
```

**Response:** `200 OK`

```json
{
  "tasks": [
    {
      "id": "497f6eca-6276-4993-bfeb-53cbbbba6f08",
      "organization_id": "7c60d51f-b44e-4682-87d6-449835ea4de6",
      "owner_id": "8826ee2e-7933-4665-aef2-2393f84a0d05",
      "owner_name": "alice",
      "owner_avatar_url": "https://example.com/avatar.png",
      "name": "fix-auth-bug",
      "initial_prompt": "Fix the authentication bug in login flow",
      "template_id": "c6d67e98-83ea-49f0-8812-e4abae2b68bc",
      "template_version_id": "0ba39c92-1f1b-4c32-aa3e-9925d7713eb1",
      "template_name": "ai-developer",
      "template_display_name": "AI Developer",
      "template_icon": "/icon/code.svg",
      "workspace_id": "0967198e-ec7b-4c6b-b4d3-f71244cadbe9",
      "workspace_name": "fix-auth-bug",
      "workspace_status": "running",
      "workspace_build_number": 1,
      "workspace_agent_id": "2b1e3b65-2c04-4fa2-a2d7-467901e98978",
      "workspace_agent_lifecycle": {
        "state": "ready",
        "changed_at": "2019-08-24T14:15:22Z"
      },
      "workspace_agent_health": {
        "healthy": true
      },
      "workspace_app_id": "affd1d10-9538-4fc8-9e0b-4594a28c1335",
      "status": "active",
      "current_state": {
        "timestamp": "2019-08-24T14:15:22Z",
        "state": "working",
        "message": "Analyzing authentication code...",
        "uri": "file:///workspace/src/auth/login.ts"
      },
      "created_at": "2019-08-24T14:15:22Z",
      "updated_at": "2019-08-24T14:15:22Z"
    }
  ],
  "count": 1
}
```

---

### 2. Create Task

**Endpoint:** `POST /api/experimental/tasks/{user}`

**Path Parameters:**

| Parameter | Type   | Description                  |
|-----------|--------|------------------------------|
| `user`    | string | Username, user ID, or `"me"` |

**Request Body:**

```json
{
  "template_version_id": "0ba39c92-1f1b-4c32-aa3e-9925d7713eb1",
  "template_version_preset_id": "512a53a7-30da-446e-a1fc-713c630baff1",
  "input": "Create a REST API endpoint for user management",
  "name": "user-api-endpoint"
}
```

**Request Schema:**

| Field                        | Type   | Required | Description                               |
|------------------------------|--------|----------|-------------------------------------------|
| `template_version_id`        | UUID   | Yes      | Template version with `has_ai_task: true` |
| `template_version_preset_id` | UUID   | No       | Preset for template parameter values      |
| `input`                      | string | Yes      | Initial prompt/instruction for AI         |
| `name`                       | string | No       | Task name (auto-generated if omitted)     |

**Example Request:**

```bash
curl -X POST \
  -H "Coder-Session-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "template_version_id": "0ba39c92-1f1b-4c32-aa3e-9925d7713eb1",
    "input": "Create a REST API for user management",
    "name": "user-api"
  }' \
  "https://coder.example.com/api/experimental/tasks/me"
```

**Response:** `201 Created`

Returns the created `Task` object (see Data Models section).

**Backend Processing:**

1. Validates template version has `has_ai_task: true`
2. Generates task name using AI if not provided (requires Anthropic API configuration)
3. Creates task record in database
4. Provisions workspace using template version
5. Returns task object with `status: "pending"` or `"initializing"`

---

### 3. Get Task

**Endpoint:** `GET /api/experimental/tasks/{user}/{task}`

**Path Parameters:**

| Parameter | Type   | Description                  |
|-----------|--------|------------------------------|
| `user`    | string | Username, user ID, or `"me"` |
| `task`    | string | Task ID (UUID) or task name  |

**Lookup Behavior:**

- First attempts UUID lookup by task ID
- Falls back to case-insensitive name lookup scoped to user

**Example Requests:**

```bash
# Get by UUID
curl -H "Coder-Session-Token: $TOKEN" \
  "https://coder.example.com/api/experimental/tasks/me/497f6eca-6276-4993-bfeb-53cbbbba6f08"

# Get by name
curl -H "Coder-Session-Token: $TOKEN" \
  "https://coder.example.com/api/experimental/tasks/alice/user-api"
```

**Response:** `200 OK`

Returns single `Task` object.

---

### 4. Delete Task

**Endpoint:** `DELETE /api/experimental/tasks/{user}/{task}`

**Path Parameters:**

| Parameter | Type   | Description                  |
|-----------|--------|------------------------------|
| `user`    | string | Username, user ID, or `"me"` |
| `task`    | string | Task ID or task name         |

**Example Request:**

```bash
curl -X DELETE \
  -H "Coder-Session-Token: $TOKEN" \
  "https://coder.example.com/api/experimental/tasks/me/user-api"
```

**Response:** `202 Accepted`

**Backend Processing:**

1. Creates workspace delete build (if workspace exists)
2. Soft-deletes task record (sets `deleted_at`)
3. Returns immediately (async operation)
4. Workspace deletion proceeds in background

---

### 5. Send Input to Task

**Endpoint:** `POST /api/experimental/tasks/{user}/{task}/send`

**Path Parameters:**

| Parameter | Type   | Description                  |
|-----------|--------|------------------------------|
| `user`    | string | Username, user ID, or `"me"` |
| `task`    | string | Task ID or task name         |

**Request Body:**

```json
{
  "input": "Now add email validation"
}
```

**Requirements:**

- Task must have `status: "active"`
- Task must have an associated workspace app
- User must have `ApplicationConnect` permission on workspace

**Example Request:**

```bash
curl -X POST \
  -H "Coder-Session-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"input": "Add email validation"}' \
  "https://coder.example.com/api/experimental/tasks/me/user-api/send"
```

**Response:** `204 No Content`

**Backend Processing:**

1. Validates task is active
2. Dials workspace agent via tailnet
3. Sends input to task app via agentapi
4. Returns immediately (AI processes asynchronously)

---

### 6. Get Task Logs

**Endpoint:** `GET /api/experimental/tasks/{user}/{task}/logs`

**Path Parameters:**

| Parameter | Type   | Description                  |
|-----------|--------|------------------------------|
| `user`    | string | Username, user ID, or `"me"` |
| `task`    | string | Task ID or task name         |

**Example Request:**

```bash
curl -H "Coder-Session-Token: $TOKEN" \
  "https://coder.example.com/api/experimental/tasks/me/user-api/logs"
```

**Response:** `200 OK`

```json
{
  "logs": [
    {
      "id": 1,
      "content": "Create a REST API for user management",
      "type": "input",
      "time": "2019-08-24T14:15:22Z"
    },
    {
      "id": 2,
      "content": "I'll help you create a user management API...",
      "type": "output",
      "time": "2019-08-24T14:15:25Z"
    },
    {
      "id": 3,
      "content": "Add email validation",
      "type": "input",
      "time": "2019-08-24T14:20:10Z"
    },
    {
      "id": 4,
      "content": "I'll add email validation to the API...",
      "type": "output",
      "time": "2019-08-24T14:20:15Z"
    }
  ]
}
```

**Backend Processing:**

1. Validates task has workspace app
2. Dials agent and fetches messages from task app
3. Returns conversation history

---

## Data Models

### Task

**TypeScript Definition:**

```typescript
interface Task {
  // Identifiers
  id: string;                           // UUID
  organization_id: string;              // UUID
  owner_id: string;                     // UUID
  owner_name: string;                   // Username
  owner_avatar_url: string;             // Avatar URL

  // Task metadata
  name: string;                         // Task name (unique per owner)
  initial_prompt: string;               // Original user prompt

  // Template information
  template_id: string;                  // UUID
  template_version_id: string;          // UUID
  template_name: string;                // Template identifier
  template_display_name: string;        // Human-readable name
  template_icon: string;                // Icon path/URL

  // Workspace information
  workspace_id: string | null;          // UUID (null if not provisioned)
  workspace_name: string;               // Workspace name
  workspace_status: WorkspaceStatus | null;
  workspace_build_number: number;       // Current build number

  // Agent information
  workspace_agent_id: string | null;    // UUID
  workspace_agent_lifecycle: {
    state: string;                      // Agent lifecycle state
    changed_at: string;                 // ISO 8601 timestamp
  } | null;
  workspace_agent_health: {
    healthy: boolean;
  } | null;

  // App information
  workspace_app_id: string | null;      // UUID

  // Task status
  status: TaskStatus;                   // Computed status
  current_state: TaskStateEntry | null; // Current AI state

  // Timestamps
  created_at: string;                   // ISO 8601
  updated_at: string;                   // ISO 8601
}
```

### TaskStatus

**Enum Values:**

```typescript
type TaskStatus =
  | "pending"       // No workspace or build pending
  | "initializing"  // Workspace building, agent connecting, or apps starting
  | "active"        // Workspace running, agent ready, apps healthy
  | "paused"        // Workspace stopped/deleted
  | "error"         // Build failed or apps unhealthy
  | "unknown";      // Status cannot be determined
```

### TaskState

**Enum Values:**

```typescript
type TaskState =
  | "idle"      // Waiting for user input
  | "working"   // Processing a request
  | "complete"  // Finished current objective
  | "failed";   // Encountered an error
```

### TaskStateEntry

**TypeScript Definition:**

```typescript
interface TaskStateEntry {
  timestamp: string;  // ISO 8601 timestamp
  state: TaskState;   // Current state
  message: string;    // Human-readable description
  uri: string;        // Optional URI (file://, https://, etc.)
}
```

**URI Examples:**

- `file:///workspace/src/api/users.ts` - File being edited
- `https://github.com/org/repo/pull/123` - Created PR
- `https://github.com/org/repo/issues/456` - Created issue
- Empty string if no relevant URI

### TaskLogEntry

**TypeScript Definition:**

```typescript
interface TaskLogEntry {
  id: number;         // Sequential log entry ID
  content: string;    // Message content
  type: "input" | "output";  // User input or AI output
  time: string;       // ISO 8601 timestamp
}
```

### TasksFilter

**TypeScript Definition:**

```typescript
interface TasksFilter {
  owner?: string;         // Username, UUID, or "me"
  organization?: string;  // Org name or UUID
  status?: TaskStatus;    // Filter by status
  filter_query?: string;  // Raw query string
}
```

### CreateTaskRequest

**TypeScript Definition:**

```typescript
interface CreateTaskRequest {
  template_version_id: string;        // UUID (required)
  template_version_preset_id?: string; // UUID (optional)
  input: string;                      // Prompt (required)
  name?: string;                      // Task name (optional)
}
```

### TaskSendRequest

**TypeScript Definition:**

```typescript
interface TaskSendRequest {
  input: string;  // Additional prompt/instruction
}
```

---

## Task Lifecycle

### 1. Creation Flow

```
User submits CreateTaskRequest
         ↓
Backend validates template version (has_ai_task: true)
         ↓
Backend generates task name (if not provided)
         ↓
Backend creates task record in database
         ↓
Backend provisions workspace from template version
         ↓
Backend returns Task object (status: "pending" or "initializing")
         ↓
Client polls GET /tasks/{user}/{task} for status updates
```

### 2. Workspace Provisioning

```
Task created (status: "pending")
         ↓
Workspace build starts (status: "initializing")
         ↓
Build job runs (provisioner creates resources)
         ↓
Build completes successfully (status: "initializing")
         ↓
Agent connects and starts (status: "initializing")
         ↓
Agent reaches "ready" state (status: "initializing")
         ↓
Workspace apps initialize (status: "initializing")
         ↓
Task app becomes healthy (status: "active")
```

### 3. Active Task Interaction

```
Task status: "active", current_state.state: "idle"
         ↓
User sends input via POST /tasks/{user}/{task}/send
         ↓
Backend forwards to task app via agentapi
         ↓
AI processes request (current_state.state: "working")
         ↓
AI updates state with progress messages
         ↓
AI completes work (current_state.state: "complete" or "idle")
         ↓
Optional: AI sets current_state.uri (e.g., GitHub PR URL)
```

### 4. Status Transitions

**State Diagram:**

```
pending → initializing → active → paused
   ↓            ↓          ↓
 error ← error ← error
```

**Detailed Transitions:**

| From         | To           | Trigger                                     |
|--------------|--------------|---------------------------------------------|
| pending      | initializing | Workspace build starts                      |
| pending      | error        | Build fails before starting                 |
| initializing | active       | Build succeeds, agent ready, apps healthy   |
| initializing | error        | Build fails, agent fails, or apps unhealthy |
| active       | paused       | Workspace stopped/deleted                   |
| active       | error        | Apps become unhealthy                       |
| any          | unknown      | Status cannot be determined                 |

### 5. Deletion Flow

```
User requests DELETE /tasks/{user}/{task}
         ↓
Backend creates workspace delete build
         ↓
Backend soft-deletes task (sets deleted_at)
         ↓
Backend returns 202 Accepted immediately
         ↓
Workspace deletion proceeds asynchronously
         ↓
Workspace and resources cleaned up
```

---

## Frontend Implementation Guide

### Required UI Components

#### 1. Tasks List View

**Purpose:** Display all tasks with filtering and navigation

**Required Elements:**

- **Task table/list** with columns:
  - Task name
  - Template icon/name
  - Owner (if showing other users' tasks)
  - Status badge
  - Created date
  - Actions (view, delete)

- **Filter controls:**
  - Owner dropdown (me, all users if admin)
  - Status filter (all, active, initializing, error, etc.)
  - Search by name

- **Create task button** → navigates to task creation

**API Calls:**

```typescript
// Fetch tasks
const response = await fetch(
  `/api/experimental/tasks?q=owner:${owner} status:${status}`,
  {
    headers: {
      'Coder-Session-Token': token
    }
  }
);
const data: TasksListResponse = await response.json();
```

**Recommended Polling:**

- Poll every 10-30 seconds
- Stop polling when component unmounts
- Consider exponential backoff on errors

---

#### 2. Task Creation Form

**Purpose:** Allow users to create new tasks

**Required Elements:**

- **Template selector:**
  - Fetch templates with `has_ai_task: true` filter
  - Display template name, icon, description

- **Template version selector** (optional, for admins):
  - Show active version by default
  - Allow override if user has update permissions

- **Preset selector** (if template has presets):
  - Load presets for selected template version
  - Show preset name and description

- **Prompt input:**
  - Large textarea for user instruction
  - Character count/limit display
  - Example prompts (contextual help)

- **Task name input** (optional):
  - Auto-suggest based on prompt
  - Validate naming rules

**API Calls:**

```typescript
// Fetch AI-enabled templates
const templates = await fetch(
  `/api/v2/organizations/${orgId}/templates?q=has-ai-task:true`,
  { headers: { 'Coder-Session-Token': token } }
).then(r => r.json());

// Create task
const task = await fetch(
  `/api/experimental/tasks/me`,
  {
    method: 'POST',
    headers: {
      'Coder-Session-Token': token,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      template_version_id: selectedVersionId,
      input: promptText,
      name: taskName  // optional
    })
  }
).then(r => r.json());

// Navigate to task detail page
navigate(`/tasks/me/${task.id}`);
```

---

#### 3. Task Detail View

**Purpose:** Display task status, interact with AI, view outputs

**Required Elements:**

- **Header:**
  - Task name
  - Back button to task list
  - Status badge
  - Template info
  - Delete button

- **Status display:**
  - Current status with icon
  - Status message (from `current_state.message`)
  - URI link (if `current_state.uri` is set)
  - Timestamp

- **Workspace status** (during initialization):
  - Build progress
  - Agent connection status
  - App health status
  - Build logs link

- **Chat/interaction area** (when active):
  - Conversation history (from GET /tasks/.../logs)
  - Input field for new prompts
  - Send button
  - Typing indicator (when state is "working")

- **Output display:**
  - File viewer (if URI is file://)
  - Link to GitHub PR/issue (if URI is GitHub)
  - Embedded preview (if applicable)

**API Calls:**

```typescript
// Fetch task details (poll every 5 seconds)
const task = await fetch(
  `/api/experimental/tasks/me/${taskId}`,
  { headers: { 'Coder-Session-Token': token } }
).then(r => r.json());

// Fetch conversation logs
const logs = await fetch(
  `/api/experimental/tasks/me/${taskId}/logs`,
  { headers: { 'Coder-Session-Token': token } }
).then(r => r.json());

// Send new input
await fetch(
  `/api/experimental/tasks/me/${taskId}/send`,
  {
    method: 'POST',
    headers: {
      'Coder-Session-Token': token,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ input: userMessage })
  }
);
```

**State Management:**

```typescript
interface TaskDetailState {
  task: Task | null;
  logs: TaskLogEntry[];
  isLoading: boolean;
  isSending: boolean;
  error: string | null;
}

// Poll task every 5 seconds
useEffect(() => {
  const interval = setInterval(async () => {
    const updatedTask = await fetchTask(taskId);
    setTask(updatedTask);

    // Refresh logs if state changed
    if (updatedTask.current_state?.state !== task?.current_state?.state) {
      const updatedLogs = await fetchLogs(taskId);
      setLogs(updatedLogs);
    }
  }, 5000);

  return () => clearInterval(interval);
}, [taskId]);
```

---

#### 4. Status Indicator Component

**Purpose:** Visual status representation

**Status → Color/Icon Mapping:**

| Status       | Color  | Icon         | Description           |
|--------------|--------|--------------|-----------------------|
| pending      | Gray   | Clock        | Waiting to start      |
| initializing | Blue   | Spinner      | Starting up           |
| active       | Green  | CheckCircle  | Ready for interaction |
| paused       | Yellow | Pause        | Stopped               |
| error        | Red    | AlertCircle  | Failed                |
| unknown      | Orange | QuestionMark | Unknown state         |

**State → Visual Indicator:**

| State    | Indicator                        |
|----------|----------------------------------|
| idle     | "Waiting for input" (calm)       |
| working  | Animated spinner + "Processing…" |
| complete | "Task complete" with checkmark   |
| failed   | "Task failed" with error icon    |

---

### Recommended UI Flow

**Flow 1: Creating a Task**

```
1. User clicks "New Task" button
2. Navigate to /tasks/new
3. Display template selector (AI-enabled templates only)
4. User selects template
5. Optional: User selects preset
6. User enters prompt
7. Optional: User provides task name
8. User clicks "Create Task"
9. POST /api/experimental/tasks/me
10. Navigate to /tasks/me/{taskId}
11. Display initializing state with build progress
12. Poll task status every 5 seconds
13. When status becomes "active", show chat interface
```

**Flow 2: Interacting with Active Task**

```
1. User on /tasks/me/{taskId}
2. Task status is "active", state is "idle"
3. User types message in input field
4. User clicks "Send" or presses Enter
5. POST /api/experimental/tasks/me/{taskId}/send
6. Disable input, show "sending" state
7. Poll task every 2-3 seconds
8. When current_state changes to "working", show typing indicator
9. Poll logs endpoint to get AI response
10. Display AI response in conversation
11. When state returns to "idle", enable input
```

**Flow 3: Viewing Task Output**

```
1. Task current_state.state is "complete"
2. current_state.uri is set (e.g., "https://github.com/org/repo/pull/123")
3. Parse URI scheme:
   - file:// → Display file path with "Open in workspace" button
   - https://github.com/.../pull/N → Display "View Pull Request #N" link
   - https://github.com/.../issues/N → Display "View Issue #N" link
   - Other → Display as hyperlink
4. Show completion message: "Task complete! [Link to output]"
```

---

## Mobile App Considerations

### Platform-Specific Challenges

#### 1. Polling & Battery Life

**Problem:** Continuous polling drains battery

**Solutions:**

- **Adaptive polling intervals:**
  ```typescript
  const getPollingInterval = (status: TaskStatus): number => {
    switch (status) {
      case 'pending':
      case 'initializing':
        return 5000;  // Poll every 5s during setup
      case 'active':
        return 10000; // Poll every 10s when active
      case 'paused':
      case 'error':
        return 0;     // Stop polling for terminal states
      default:
        return 30000; // Default: 30s
    }
  };
  ```

- **Background task limits:**
  - iOS: Use `BackgroundTasks` framework with discretionary scheduling
  - Android: Use `WorkManager` with constraints

- **Push notifications (future):**
  - When backend adds webhook/notification support
  - Push on state changes: "Your task is ready", "Task complete", "Task failed"

#### 2. Limited Screen Space

**Recommendations:**

- **Compact task list:**
  - Show only: name, status icon, time
  - Tap to expand for full details

- **Collapsible sections:**
  - Workspace details (hidden by default)
  - Build logs (modal or separate screen)
  - Full prompt (truncated with "Read more")

- **Tab-based detail view:**
  - Tab 1: Chat/Interaction
  - Tab 2: Status/Info
  - Tab 3: Logs

#### 3. Offline Support

**Strategy:**

- **Cache task list:**
  - Store last-fetched task list locally
  - Display cached data with "offline" badge
  - Refresh when network returns

- **Queue user inputs:**
  - Allow user to type messages offline
  - Queue POST /send requests
  - Upload when network available
  - Show "queued" status

#### 4. Deep Linking

**Implement URL schemes:**

```
coder://tasks                      → Task list
coder://tasks/new                  → Create task
coder://tasks/:username/:taskId    → Task detail
```

**Handle notifications:**

```
"Your task 'user-api' is ready"
  → Deep link to coder://tasks/me/user-api
```

---

### Mobile API Client Example (TypeScript)

```typescript
class TasksAPIClient {
  constructor(
    private baseUrl: string,
    private token: string
  ) {}

  // List tasks with filters
  async listTasks(filter: TasksFilter): Promise<Task[]> {
    const params = new URLSearchParams();

    const queryParts: string[] = [];
    if (filter.owner) queryParts.push(`owner:${filter.owner}`);
    if (filter.status) queryParts.push(`status:${filter.status}`);
    if (filter.organization) queryParts.push(`organization:${filter.organization}`);

    if (queryParts.length > 0) {
      params.set('q', queryParts.join(' '));
    }

    const response = await fetch(
      `${this.baseUrl}/api/experimental/tasks?${params}`,
      {
        headers: {
          'Coder-Session-Token': this.token
        }
      }
    );

    if (!response.ok) {
      throw new Error(`Failed to fetch tasks: ${response.statusText}`);
    }

    const data: TasksListResponse = await response.json();
    return data.tasks;
  }

  // Get single task
  async getTask(user: string, taskId: string): Promise<Task> {
    const response = await fetch(
      `${this.baseUrl}/api/experimental/tasks/${user}/${taskId}`,
      {
        headers: {
          'Coder-Session-Token': this.token
        }
      }
    );

    if (!response.ok) {
      throw new Error(`Failed to fetch task: ${response.statusText}`);
    }

    return response.json();
  }

  // Create task
  async createTask(user: string, request: CreateTaskRequest): Promise<Task> {
    const response = await fetch(
      `${this.baseUrl}/api/experimental/tasks/${user}`,
      {
        method: 'POST',
        headers: {
          'Coder-Session-Token': this.token,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(request)
      }
    );

    if (!response.ok) {
      const error = await response.text();
      throw new Error(`Failed to create task: ${error}`);
    }

    return response.json();
  }

  // Send input to task
  async sendInput(user: string, taskId: string, input: string): Promise<void> {
    const response = await fetch(
      `${this.baseUrl}/api/experimental/tasks/${user}/${taskId}/send`,
      {
        method: 'POST',
        headers: {
          'Coder-Session-Token': this.token,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ input })
      }
    );

    if (!response.ok) {
      const error = await response.text();
      throw new Error(`Failed to send input: ${error}`);
    }
  }

  // Get task logs
  async getLogs(user: string, taskId: string): Promise<TaskLogEntry[]> {
    const response = await fetch(
      `${this.baseUrl}/api/experimental/tasks/${user}/${taskId}/logs`,
      {
        headers: {
          'Coder-Session-Token': this.token
        }
      }
    );

    if (!response.ok) {
      throw new Error(`Failed to fetch logs: ${response.statusText}`);
    }

    const data: TaskLogsResponse = await response.json();
    return data.logs;
  }

  // Delete task
  async deleteTask(user: string, taskId: string): Promise<void> {
    const response = await fetch(
      `${this.baseUrl}/api/experimental/tasks/${user}/${taskId}`,
      {
        method: 'DELETE',
        headers: {
          'Coder-Session-Token': this.token
        }
      }
    );

    if (!response.ok) {
      throw new Error(`Failed to delete task: ${response.statusText}`);
    }
  }
}
```

---

### React Native Hook Example

```typescript
import { useState, useEffect } from 'react';
import { TasksAPIClient } from './api-client';

export function useTask(user: string, taskId: string) {
  const [task, setTask] = useState<Task | null>(null);
  const [logs, setLogs] = useState<TaskLogEntry[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const client = new TasksAPIClient(CODER_URL, SESSION_TOKEN);

  // Fetch task and logs
  useEffect(() => {
    let interval: NodeJS.Timeout;

    const fetchData = async () => {
      try {
        const [taskData, logsData] = await Promise.all([
          client.getTask(user, taskId),
          client.getLogs(user, taskId)
        ]);

        setTask(taskData);
        setLogs(logsData);
        setLoading(false);

        // Set polling interval based on status
        const pollingInterval = getPollingInterval(taskData.status);
        if (pollingInterval > 0) {
          interval = setTimeout(fetchData, pollingInterval);
        }
      } catch (err) {
        setError(err.message);
        setLoading(false);
      }
    };

    fetchData();

    return () => {
      if (interval) clearTimeout(interval);
    };
  }, [user, taskId]);

  // Send input
  const sendInput = async (input: string) => {
    try {
      await client.sendInput(user, taskId, input);

      // Immediately fetch updated logs
      const updatedLogs = await client.getLogs(user, taskId);
      setLogs(updatedLogs);
    } catch (err) {
      throw new Error(`Failed to send input: ${err.message}`);
    }
  };

  return { task, logs, loading, error, sendInput };
}

function getPollingInterval(status: TaskStatus): number {
  switch (status) {
    case 'pending':
    case 'initializing':
      return 5000;
    case 'active':
      return 10000;
    default:
      return 0;
  }
}
```

---

## Real-time Updates Strategy

### Current Limitation

**No WebSocket or SSE support** - The Tasks API is currently REST-only with no real-time push mechanism.

### Polling Strategy

#### Recommended Intervals

| Context                | Polling Interval | Reasoning                                  |
|------------------------|------------------|--------------------------------------------|
| Task list view         | 10-30 seconds    | Low-priority, batch updates acceptable     |
| Task detail (init)     | 3-5 seconds      | User waiting for workspace to become ready |
| Task detail (active)   | 5-10 seconds     | Moderate updates, check for state changes  |
| Task detail (terminal) | Stop polling     | No more updates expected                   |
| Background/inactive    | 60 seconds       | Reduce battery drain                       |

#### Adaptive Polling Implementation

```typescript
class AdaptivePoller {
  private interval: NodeJS.Timeout | null = null;
  private currentStatus: TaskStatus | null = null;

  constructor(
    private fetchFn: () => Promise<Task>,
    private onUpdate: (task: Task) => void,
    private onError: (error: Error) => void
  ) {}

  start() {
    this.poll();
  }

  stop() {
    if (this.interval) {
      clearTimeout(this.interval);
      this.interval = null;
    }
  }

  private async poll() {
    try {
      const task = await this.fetchFn();
      this.currentStatus = task.status;
      this.onUpdate(task);

      // Schedule next poll based on status
      const interval = this.getInterval(task.status);
      if (interval > 0) {
        this.interval = setTimeout(() => this.poll(), interval);
      }
    } catch (error) {
      this.onError(error);

      // Retry with exponential backoff on error
      this.interval = setTimeout(() => this.poll(), 10000);
    }
  }

  private getInterval(status: TaskStatus): number {
    switch (status) {
      case 'pending':
      case 'initializing':
        return 5000;  // Fast polling during setup
      case 'active':
        return 10000; // Moderate polling when active
      case 'paused':
      case 'error':
        return 0;     // Stop polling for terminal states
      default:
        return 30000;
    }
  }
}

// Usage
const poller = new AdaptivePoller(
  () => api.getTask('me', taskId),
  (task) => setTask(task),
  (error) => console.error(error)
);

poller.start();

// Cleanup
useEffect(() => {
  return () => poller.stop();
}, []);
```

#### Exponential Backoff on Errors

```typescript
class PollerWithBackoff {
  private retries = 0;
  private maxRetries = 5;

  private getBackoffDelay(): number {
    // 2s, 4s, 8s, 16s, 32s
    return Math.min(2000 * Math.pow(2, this.retries), 32000);
  }

  private async poll() {
    try {
      const task = await this.fetchFn();
      this.retries = 0; // Reset on success
      this.onUpdate(task);

      const interval = this.getInterval(task.status);
      if (interval > 0) {
        this.interval = setTimeout(() => this.poll(), interval);
      }
    } catch (error) {
      this.retries++;

      if (this.retries >= this.maxRetries) {
        this.onError(new Error('Max retries exceeded'));
        return;
      }

      const backoff = this.getBackoffDelay();
      this.interval = setTimeout(() => this.poll(), backoff);
    }
  }
}
```

### Future: WebSocket Support

**If Coder adds WebSocket support in the future:**

```typescript
// Hypothetical WebSocket implementation
const ws = new WebSocket(`wss://coder.example.com/api/experimental/tasks/${taskId}/watch`);

ws.onmessage = (event) => {
  const update = JSON.parse(event.data);

  switch (update.type) {
    case 'status_changed':
      setTask(update.task);
      break;
    case 'state_changed':
      setTask(prev => ({ ...prev, current_state: update.state }));
      break;
    case 'log_entry':
      setLogs(prev => [...prev, update.log]);
      break;
  }
};
```

---

## Error Handling

### Common Error Scenarios

#### 1. Task Not Found (404)

**Scenario:** Task ID/name doesn't exist or user lacks permission

**Response:**

```json
{
  "message": "Task not found",
  "detail": "Task '12345' does not exist or you do not have access"
}
```

**Handling:**

```typescript
try {
  const task = await api.getTask('me', taskId);
} catch (error) {
  if (error.status === 404) {
    // Navigate to task list with error message
    navigate('/tasks', {
      state: { error: 'Task not found or deleted' }
    });
  }
}
```

---

#### 2. Unauthorized (401)

**Scenario:** Session token expired or invalid

**Response:**

```json
{
  "message": "Unauthorized",
  "detail": "Session token is invalid or expired"
}
```

**Handling:**

```typescript
api.on('unauthorized', () => {
  // Clear local session
  sessionStorage.clear();

  // Redirect to login
  navigate('/login', {
    state: { returnUrl: location.pathname }
  });
});
```

---

#### 3. Task Not Active (400)

**Scenario:** Sending input to non-active task

**Response:**

```json
{
  "message": "Bad Request",
  "detail": "Task must be in 'active' status to send input"
}
```

**Handling:**

```typescript
const sendInput = async (input: string) => {
  if (task.status !== 'active') {
    showError('Task is not ready to receive input');
    return;
  }

  try {
    await api.sendInput('me', taskId, input);
  } catch (error) {
    if (error.status === 400) {
      showError('Cannot send input: task is not active');
    }
  }
};
```

---

#### 4. Template Missing AI Capability (400)

**Scenario:** Creating task with non-AI template

**Response:**

```json
{
  "message": "Bad Request",
  "detail": "Template version does not have AI task capability enabled"
}
```

**Handling:**

```typescript
// Pre-filter templates when fetching
const templates = await api.getTemplates({
  filter: 'has-ai-task:true'
});

// Or validate before submission
const createTask = async (request: CreateTaskRequest) => {
  const templateVersion = await api.getTemplateVersion(
    request.template_version_id
  );

  if (!templateVersion.has_ai_task) {
    throw new Error('Selected template does not support AI tasks');
  }

  return api.createTask('me', request);
};
```

---

#### 5. Workspace Build Failed (status: error)

**Scenario:** Workspace provisioning failed

**Task Object:**

```json
{
  "status": "error",
  "current_state": {
    "state": "failed",
    "message": "Terraform apply failed: insufficient quota",
    "uri": ""
  }
}
```

**Handling:**

```tsx
if (task.status === 'error') {
  return (
    <ErrorView>
      <AlertIcon />
      <h2>Task Failed</h2>
      <p>{task.current_state?.message || 'Workspace provisioning failed'}</p>
      <Button onClick={() => navigate(`/workspaces/${task.workspace_name}/builds/${task.workspace_build_number}`)}>
        View Build Logs
      </Button>
      <Button onClick={() => deleteTask(task.id)}>
        Delete Task
      </Button>
    </ErrorView>
  );
}
```

---

### Error Recovery Strategies

#### Retry with Exponential Backoff

```typescript
async function fetchWithRetry<T>(
  fetchFn: () => Promise<T>,
  maxRetries = 3
): Promise<T> {
  let retries = 0;

  while (retries < maxRetries) {
    try {
      return await fetchFn();
    } catch (error) {
      retries++;

      if (retries >= maxRetries) {
        throw error;
      }

      // Wait before retry: 1s, 2s, 4s
      await new Promise(resolve =>
        setTimeout(resolve, 1000 * Math.pow(2, retries - 1))
      );
    }
  }

  throw new Error('Max retries exceeded');
}

// Usage
const task = await fetchWithRetry(() => api.getTask('me', taskId));
```

#### Offline Queue

```typescript
class OfflineQueue {
  private queue: Array<() => Promise<void>> = [];

  async enqueue(operation: () => Promise<void>) {
    this.queue.push(operation);

    if (navigator.onLine) {
      await this.flush();
    }
  }

  async flush() {
    while (this.queue.length > 0) {
      const operation = this.queue[0];

      try {
        await operation();
        this.queue.shift(); // Remove on success
      } catch (error) {
        // Stop flushing on error, will retry later
        break;
      }
    }
  }
}

// Usage
const queue = new OfflineQueue();

window.addEventListener('online', () => {
  queue.flush();
});

// Enqueue send input
await queue.enqueue(() =>
  api.sendInput('me', taskId, userMessage)
);
```

---

## Security & Permissions

### Authentication

All API endpoints require authentication via:

```
Header: Coder-Session-Token: <token>
```

**Token Acquisition:**

1. User logs in via `/api/v2/users/login`
2. Receives session token in response
3. Token stored in:
   - **Web:** Cookie (`coder_session_token`)
   - **Mobile:** Secure storage (Keychain/Keystore)

**Token Expiration:**

- Tokens expire after configurable duration (default: 24 hours)
- 401 Unauthorized indicates expired token
- Client must re-authenticate

---

### Authorization & RBAC

Tasks leverage Coder's existing RBAC system:

| Permission            | Required For                         |
|-----------------------|--------------------------------------|
| `workspace:create`    | Creating tasks (creates workspace)   |
| `workspace:read`      | Viewing tasks                        |
| `workspace:update`    | Sending input to tasks               |
| `workspace:delete`    | Deleting tasks                       |
| `application:connect` | Sending input (connects to task app) |
| `template:read`       | Viewing templates for task creation  |
| `organization:read`   | Viewing tasks in organization        |

**Viewing Other Users' Tasks:**

- Users can only view their own tasks by default
- Organization admins can view all tasks in their org
- Use `owner:username` filter to view specific user's tasks (if permitted)

---

### Data Privacy

**Sensitive Fields:**

- `initial_prompt` - Contains user instruction (may include sensitive info)
- `logs[].content` - Contains conversation history
- `current_state.message` - May contain file paths, URLs, etc.

**Best Practices:**

1. **Don't log sensitive data** - Avoid logging full task objects
2. **Encrypt in transit** - Always use HTTPS for API calls
3. **Secure token storage** - Use platform-specific secure storage
4. **Minimize data retention** - Delete completed tasks when no longer needed

---

## Code References

For developers implementing alternative frontends, here are key file references in the Coder codebase:

### Backend (Go)

| File                                           | Purpose                              |
|------------------------------------------------|--------------------------------------|
| `coderd/aitasks.go`                            | API endpoint handlers (lines 1-833)  |
| `coderd/httpmw/taskparam.go`                   | Task parameter extraction middleware |
| `coderd/searchquery/search.go` (lines 400-429) | Query filter parsing                 |
| `coderd/database/queries/tasks.sql`            | SQL queries for tasks                |
| `coderd/database/migrations/000366*.sql`       | Base table schema                    |
| `coderd/database/migrations/000379*.sql`       | Status view + indexes                |
| `coderd/database/migrations/000398*.sql`       | Updated status computation           |

### Frontend (TypeScript/React)

| File                                                   | Purpose                     |
|--------------------------------------------------------|-----------------------------|
| `site/src/pages/TasksPage/TasksPage.tsx`               | Task list page              |
| `site/src/pages/TaskPage/TaskPage.tsx`                 | Task detail page            |
| `site/src/modules/tasks/TaskPrompt/TaskPrompt.tsx`     | Task creation form          |
| `site/src/modules/tasks/TasksSidebar/TasksSidebar.tsx` | Task navigation sidebar     |
| `site/src/modules/tasks/TaskStatus/TaskStatus.tsx`     | Status indicator component  |
| `site/src/api/api.ts` (lines 2664-2718)                | API client methods          |
| `site/src/api/typesGenerated.ts` (lines 4715-4855)     | TypeScript type definitions |

---

## Appendix: Full API Response Examples

### GET /api/experimental/tasks

```json
{
  "tasks": [
    {
      "id": "a1b2c3d4-e5f6-4a5b-8c9d-0e1f2a3b4c5d",
      "organization_id": "7c60d51f-b44e-4682-87d6-449835ea4de6",
      "owner_id": "8826ee2e-7933-4665-aef2-2393f84a0d05",
      "owner_name": "alice",
      "owner_avatar_url": "https://example.com/avatars/alice.png",
      "name": "implement-user-api",
      "initial_prompt": "Create a REST API for user management with CRUD operations",
      "template_id": "c6d67e98-83ea-49f0-8812-e4abae2b68bc",
      "template_version_id": "0ba39c92-1f1b-4c32-aa3e-9925d7713eb1",
      "template_name": "ai-developer",
      "template_display_name": "AI Developer",
      "template_icon": "/icon/code.svg",
      "workspace_id": "0967198e-ec7b-4c6b-b4d3-f71244cadbe9",
      "workspace_name": "implement-user-api",
      "workspace_status": "running",
      "workspace_build_number": 1,
      "workspace_agent_id": "2b1e3b65-2c04-4fa2-a2d7-467901e98978",
      "workspace_agent_lifecycle": {
        "state": "ready",
        "changed_at": "2019-08-24T14:15:22Z"
      },
      "workspace_agent_health": {
        "healthy": true
      },
      "workspace_app_id": "affd1d10-9538-4fc8-9e0b-4594a28c1335",
      "status": "active",
      "current_state": {
        "timestamp": "2019-08-24T14:20:45Z",
        "state": "complete",
        "message": "Created pull request with user API implementation",
        "uri": "https://github.com/acme/backend/pull/456"
      },
      "created_at": "2019-08-24T14:10:00Z",
      "updated_at": "2019-08-24T14:20:45Z"
    },
    {
      "id": "f9e8d7c6-b5a4-4321-9876-543210fedcba",
      "organization_id": "7c60d51f-b44e-4682-87d6-449835ea4de6",
      "owner_id": "8826ee2e-7933-4665-aef2-2393f84a0d05",
      "owner_name": "alice",
      "owner_avatar_url": "https://example.com/avatars/alice.png",
      "name": "fix-login-bug",
      "initial_prompt": "Debug and fix the authentication error on login",
      "template_id": "c6d67e98-83ea-49f0-8812-e4abae2b68bc",
      "template_version_id": "0ba39c92-1f1b-4c32-aa3e-9925d7713eb1",
      "template_name": "ai-developer",
      "template_display_name": "AI Developer",
      "template_icon": "/icon/code.svg",
      "workspace_id": "1234abcd-5678-90ef-ghij-klmnopqrstuv",
      "workspace_name": "fix-login-bug",
      "workspace_status": "running",
      "workspace_build_number": 1,
      "workspace_agent_id": "abcd1234-5678-90ef-ghij-klmnopqrstuv",
      "workspace_agent_lifecycle": {
        "state": "ready",
        "changed_at": "2019-08-24T15:05:10Z"
      },
      "workspace_agent_health": {
        "healthy": true
      },
      "workspace_app_id": "wxyz9876-5432-10fe-dcba-9876543210fe",
      "status": "active",
      "current_state": {
        "timestamp": "2019-08-24T15:15:30Z",
        "state": "idle",
        "message": "Waiting for your next instruction",
        "uri": ""
      },
      "created_at": "2019-08-24T15:00:00Z",
      "updated_at": "2019-08-24T15:15:30Z"
    }
  ],
  "count": 2
}
```

### GET /api/experimental/tasks/{user}/{task}/logs

```json
{
  "logs": [
    {
      "id": 1,
      "content": "Create a REST API for user management with CRUD operations",
      "type": "input",
      "time": "2019-08-24T14:10:05Z"
    },
    {
      "id": 2,
      "content": "I'll help you create a comprehensive user management API. I'll start by setting up the basic structure with routes for Create, Read, Update, and Delete operations.\n\nFirst, let me create the API routes file...",
      "type": "output",
      "time": "2019-08-24T14:10:12Z"
    },
    {
      "id": 3,
      "content": "I've created the basic structure. Here's what I've implemented:\n\n1. User model with validation\n2. CRUD endpoints:\n   - POST /api/users - Create user\n   - GET /api/users - List users\n   - GET /api/users/:id - Get user by ID\n   - PUT /api/users/:id - Update user\n   - DELETE /api/users/:id - Delete user\n3. Input validation middleware\n4. Error handling\n\nThe implementation is in `src/api/users.ts`. Would you like me to add authentication or additional features?",
      "type": "output",
      "time": "2019-08-24T14:12:45Z"
    },
    {
      "id": 4,
      "content": "Add email validation and password hashing",
      "type": "input",
      "time": "2019-08-24T14:15:00Z"
    },
    {
      "id": 5,
      "content": "I'll add email validation using a regex pattern and implement password hashing with bcrypt. Let me update the user model and routes...",
      "type": "output",
      "time": "2019-08-24T14:15:08Z"
    },
    {
      "id": 6,
      "content": "Done! I've added:\n\n1. Email validation with regex pattern in the user model\n2. Password hashing using bcrypt (10 rounds)\n3. Password comparison method for authentication\n4. Updated create and update endpoints to hash passwords\n\nThe passwords are now securely hashed before storage and never returned in API responses. Would you like me to create a pull request with these changes?",
      "type": "output",
      "time": "2019-08-24T14:18:20Z"
    },
    {
      "id": 7,
      "content": "Yes, create a PR",
      "type": "input",
      "time": "2019-08-24T14:19:00Z"
    },
    {
      "id": 8,
      "content": "I've created a pull request with all the changes:\n\nhttps://github.com/acme/backend/pull/456\n\nThe PR includes:\n- User API with CRUD operations\n- Email validation\n- Bcrypt password hashing\n- Comprehensive tests\n- API documentation\n\nThe PR is ready for review!",
      "type": "output",
      "time": "2019-08-24T14:20:45Z"
    }
  ]
}
```

---

## Conclusion

This report provides a comprehensive guide to implementing Tasks functionality in alternative Coder frontends. The Tasks API is straightforward to integrate using standard REST calls with polling for updates.

**Key Takeaways:**

1. **All endpoints are experimental** - API may change
2. **Polling is required** - No WebSocket/SSE support currently
3. **Status is computed** - Derived from workspace/agent/app state
4. **Workspace-backed** - Tasks are tightly coupled with workspaces
5. **Template requirements** - Only AI-enabled templates can be used

For the most up-to-date API information, refer to:
- API Documentation: `/docs/reference/api/tasks.md`
- Backend Implementation: `coderd/aitasks.go`
- Frontend Reference: `site/src/pages/TaskPage/`

**Questions or Issues?**

- Check the Coder documentation at https://coder.com/docs
- Review the source code in the Coder repository
- Contact the Coder team for API clarifications
