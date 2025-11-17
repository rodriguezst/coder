# Tasks

Tasks are AI-powered workspaces that provide automated development assistance. They allow users to create workspaces from templates with AI capabilities, interact with them through prompts, and monitor their status.

**Note:** All Tasks endpoints are currently experimental and located under `/api/experimental/tasks`.

## List tasks

### Code samples

```shell
# Example request using curl
curl -X GET http://coder-server:8080/api/experimental/tasks \
  -H 'Accept: application/json' \
  -H 'Coder-Session-Token: API_KEY'

# Filter by owner
curl -X GET 'http://coder-server:8080/api/experimental/tasks?q=owner:me' \
  -H 'Accept: application/json' \
  -H 'Coder-Session-Token: API_KEY'

# Filter by status
curl -X GET 'http://coder-server:8080/api/experimental/tasks?q=status:active' \
  -H 'Accept: application/json' \
  -H 'Coder-Session-Token: API_KEY'

# Multiple filters
curl -X GET 'http://coder-server:8080/api/experimental/tasks?q=owner:me%20status:active' \
  -H 'Accept: application/json' \
  -H 'Coder-Session-Token: API_KEY'
```

`GET /experimental/tasks`

Returns a list of tasks with optional filtering by owner, organization, and status.

### Parameters

| Name | In    | Type   | Required | Description                                                          |
|------|-------|--------|----------|----------------------------------------------------------------------|
| `q`  | query | string | false    | Search query. Supports filters: `owner:`, `organization:`, `status:` |

### Query Filter Syntax

The `q` parameter supports the following filters:

- `owner:<username|uuid|me>` - Filter by task owner
- `organization:<org-name|uuid>` - Filter by organization
- `status:<status>` - Filter by task status (pending, initializing, active, paused, error, unknown)

Multiple filters can be combined with spaces, e.g., `owner:me status:active`.

### Example responses

> 200 Response

```json
{
  "tasks": [
    {
      "id": "497f6eca-6276-4993-bfeb-53cbbbba6f08",
      "organization_id": "7c60d51f-b44e-4682-87d6-449835ea4de6",
      "owner_id": "8826ee2e-7933-4665-aef2-2393f84a0d05",
      "owner_name": "user@example.com",
      "owner_avatar_url": "https://example.com/avatar.png",
      "name": "fix-authentication-bug",
      "initial_prompt": "Fix the authentication bug in the login flow",
      "template_id": "c6d67e98-83ea-49f0-8812-e4abae2b68bc",
      "template_version_id": "0ba39c92-1f1b-4c32-aa3e-9925d7713eb1",
      "template_name": "ai-developer",
      "template_display_name": "AI Developer",
      "template_icon": "/icon/code.svg",
      "workspace_id": "0967198e-ec7b-4c6b-b4d3-f71244cadbe9",
      "workspace_name": "fix-authentication-bug",
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
        "message": "Analyzing code...",
        "uri": "https://github.com/user/repo/pull/123"
      },
      "created_at": "2019-08-24T14:15:22Z",
      "updated_at": "2019-08-24T14:15:22Z"
    }
  ],
  "count": 1
}
```

### Responses

| Status | Meaning                                                         | Description  | Schema                                                             |
|--------|-----------------------------------------------------------------|--------------|--------------------------------------------------------------------|
| 200    | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1)         | OK           | [codersdk.TasksListResponse](schemas.md#codersdktaskslistresponse) |
| 401    | [Unauthorized](https://tools.ietf.org/html/rfc7235#section-3.1) | Unauthorized | -                                                                  |

---

## Create task

### Code samples

```shell
# Example request using curl
curl -X POST http://coder-server:8080/api/experimental/tasks/{user} \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -H 'Coder-Session-Token: API_KEY' \
  -d '{
    "template_version_id": "0ba39c92-1f1b-4c32-aa3e-9925d7713eb1",
    "input": "Create a new REST API endpoint for user management",
    "name": "user-api-endpoint"
  }'
```

`POST /experimental/tasks/{user}`

Create a new AI task for the specified user. The task creates an associated workspace from a template version that has AI capabilities enabled (`has_ai_task: true`).

> Body parameter

```json
{
  "template_version_id": "0ba39c92-1f1b-4c32-aa3e-9925d7713eb1",
  "template_version_preset_id": "512a53a7-30da-446e-a1fc-713c630baff1",
  "input": "Create a new REST API endpoint for user management",
  "name": "user-api-endpoint"
}
```

### Parameters

| Name   | In   | Type                                                               | Required | Description                |
|--------|------|--------------------------------------------------------------------|----------|----------------------------|
| `user` | path | string                                                             | true     | Username, user ID, or 'me' |
| `body` | body | [codersdk.CreateTaskRequest](schemas.md#codersdkcreatetaskrequest) | true     | Create task request        |

### Request Body Schema

| Name                         | Type         | Required | Description                                         |
|------------------------------|--------------|----------|-----------------------------------------------------|
| `template_version_id`        | string(uuid) | true     | Template version ID (must have AI task capability)  |
| `template_version_preset_id` | string(uuid) | false    | Template version preset ID for parameter values     |
| `input`                      | string       | true     | Initial prompt/instruction for the AI task          |
| `name`                       | string       | false    | Task name (auto-generated using AI if not provided) |

### Example responses

> 201 Response

```json
{
  "id": "497f6eca-6276-4993-bfeb-53cbbbba6f08",
  "organization_id": "7c60d51f-b44e-4682-87d6-449835ea4de6",
  "owner_id": "8826ee2e-7933-4665-aef2-2393f84a0d05",
  "owner_name": "user@example.com",
  "owner_avatar_url": "https://example.com/avatar.png",
  "name": "user-api-endpoint",
  "initial_prompt": "Create a new REST API endpoint for user management",
  "template_id": "c6d67e98-83ea-49f0-8812-e4abae2b68bc",
  "template_version_id": "0ba39c92-1f1b-4c32-aa3e-9925d7713eb1",
  "template_name": "ai-developer",
  "template_display_name": "AI Developer",
  "template_icon": "/icon/code.svg",
  "workspace_id": "0967198e-ec7b-4c6b-b4d3-f71244cadbe9",
  "workspace_name": "user-api-endpoint",
  "workspace_status": "starting",
  "workspace_build_number": 1,
  "workspace_agent_id": null,
  "workspace_agent_lifecycle": null,
  "workspace_agent_health": null,
  "workspace_app_id": null,
  "status": "pending",
  "current_state": null,
  "created_at": "2019-08-24T14:15:22Z",
  "updated_at": "2019-08-24T14:15:22Z"
}
```

### Responses

| Status | Meaning                                                          | Description  | Schema                                   |
|--------|------------------------------------------------------------------|--------------|------------------------------------------|
| 201    | [Created](https://tools.ietf.org/html/rfc7231#section-6.3.2)     | Created      | [codersdk.Task](schemas.md#codersdktask) |
| 400    | [Bad Request](https://tools.ietf.org/html/rfc7231#section-6.5.1) | Bad Request  | -                                        |
| 401    | [Unauthorized](https://tools.ietf.org/html/rfc7235#section-3.1)  | Unauthorized | -                                        |
| 404    | [Not Found](https://tools.ietf.org/html/rfc7231#section-6.5.4)   | Not Found    | -                                        |

---

## Get task

### Code samples

```shell
# Example request using curl by task ID
curl -X GET http://coder-server:8080/api/experimental/tasks/{user}/{task} \
  -H 'Accept: application/json' \
  -H 'Coder-Session-Token: API_KEY'

# Example request using task name
curl -X GET http://coder-server:8080/api/experimental/tasks/me/user-api-endpoint \
  -H 'Accept: application/json' \
  -H 'Coder-Session-Token: API_KEY'
```

`GET /experimental/tasks/{user}/{task}`

Get a specific task by ID or name. The task parameter supports both UUID lookup and name-based lookup scoped to the specified user.

### Parameters

| Name   | In   | Type   | Required | Description                 |
|--------|------|--------|----------|-----------------------------|
| `user` | path | string | true     | Username, user ID, or 'me'  |
| `task` | path | string | true     | Task ID (UUID) or task name |

### Example responses

> 200 Response

```json
{
  "id": "497f6eca-6276-4993-bfeb-53cbbbba6f08",
  "organization_id": "7c60d51f-b44e-4682-87d6-449835ea4de6",
  "owner_id": "8826ee2e-7933-4665-aef2-2393f84a0d05",
  "owner_name": "user@example.com",
  "owner_avatar_url": "https://example.com/avatar.png",
  "name": "user-api-endpoint",
  "initial_prompt": "Create a new REST API endpoint for user management",
  "template_id": "c6d67e98-83ea-49f0-8812-e4abae2b68bc",
  "template_version_id": "0ba39c92-1f1b-4c32-aa3e-9925d7713eb1",
  "template_name": "ai-developer",
  "template_display_name": "AI Developer",
  "template_icon": "/icon/code.svg",
  "workspace_id": "0967198e-ec7b-4c6b-b4d3-f71244cadbe9",
  "workspace_name": "user-api-endpoint",
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
    "message": "Implementing user API endpoints...",
    "uri": "file:///workspace/src/api/users.ts"
  },
  "created_at": "2019-08-24T14:15:22Z",
  "updated_at": "2019-08-24T14:15:22Z"
}
```

### Responses

| Status | Meaning                                                         | Description  | Schema                                   |
|--------|-----------------------------------------------------------------|--------------|------------------------------------------|
| 200    | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1)         | OK           | [codersdk.Task](schemas.md#codersdktask) |
| 401    | [Unauthorized](https://tools.ietf.org/html/rfc7235#section-3.1) | Unauthorized | -                                        |
| 404    | [Not Found](https://tools.ietf.org/html/rfc7231#section-6.5.4)  | Not Found    | -                                        |

---

## Delete task

### Code samples

```shell
# Example request using curl
curl -X DELETE http://coder-server:8080/api/experimental/tasks/{user}/{task} \
  -H 'Coder-Session-Token: API_KEY'
```

`DELETE /experimental/tasks/{user}/{task}`

Delete a task and its associated workspace. This operation is asynchronous and returns immediately. The workspace will be deleted through the normal workspace deletion flow.

### Parameters

| Name   | In   | Type   | Required | Description                 |
|--------|------|--------|----------|-----------------------------|
| `user` | path | string | true     | Username, user ID, or 'me'  |
| `task` | path | string | true     | Task ID (UUID) or task name |

### Responses

| Status | Meaning                                                         | Description  | Schema |
|--------|-----------------------------------------------------------------|--------------|--------|
| 202    | [Accepted](https://tools.ietf.org/html/rfc7231#section-6.3.3)   | Accepted     | -      |
| 401    | [Unauthorized](https://tools.ietf.org/html/rfc7235#section-3.1) | Unauthorized | -      |
| 404    | [Not Found](https://tools.ietf.org/html/rfc7231#section-6.5.4)  | Not Found    | -      |

---

## Send input to task

### Code samples

```shell
# Example request using curl
curl -X POST http://coder-server:8080/api/experimental/tasks/{user}/{task}/send \
  -H 'Content-Type: application/json' \
  -H 'Coder-Session-Token: API_KEY' \
  -d '{
    "input": "Now add validation for email addresses"
  }'
```

`POST /experimental/tasks/{user}/{task}/send`

Send additional input/prompt to an active task. The task must be in 'active' status and have a workspace app available.

> Body parameter

```json
{
  "input": "Now add validation for email addresses"
}
```

### Parameters

| Name   | In   | Type                                                           | Required | Description                 |
|--------|------|----------------------------------------------------------------|----------|-----------------------------|
| `user` | path | string                                                         | true     | Username, user ID, or 'me'  |
| `task` | path | string                                                         | true     | Task ID (UUID) or task name |
| `body` | body | [codersdk.TaskSendRequest](schemas.md#codersdktasksendrequest) | true     | Task input request          |

### Request Body Schema

| Name    | Type   | Required | Description                        |
|---------|--------|----------|------------------------------------|
| `input` | string | true     | Additional prompt/instruction text |

### Responses

| Status | Meaning                                                          | Description                                       | Schema |
|--------|------------------------------------------------------------------|---------------------------------------------------|--------|
| 204    | [No Content](https://tools.ietf.org/html/rfc7231#section-6.3.5)  | No Content (success)                              | -      |
| 400    | [Bad Request](https://tools.ietf.org/html/rfc7231#section-6.5.1) | Bad Request (task not active or no workspace app) | -      |
| 401    | [Unauthorized](https://tools.ietf.org/html/rfc7235#section-3.1)  | Unauthorized                                      | -      |
| 404    | [Not Found](https://tools.ietf.org/html/rfc7231#section-6.5.4)   | Not Found                                         | -      |

---

## Get task logs

### Code samples

```shell
# Example request using curl
curl -X GET http://coder-server:8080/api/experimental/tasks/{user}/{task}/logs \
  -H 'Accept: application/json' \
  -H 'Coder-Session-Token: API_KEY'
```

`GET /experimental/tasks/{user}/{task}/logs`

Retrieve the conversation history (logs) for a task, including both user inputs and AI responses.

### Parameters

| Name   | In   | Type   | Required | Description                 |
|--------|------|--------|----------|-----------------------------|
| `user` | path | string | true     | Username, user ID, or 'me'  |
| `task` | path | string | true     | Task ID (UUID) or task name |

### Example responses

> 200 Response

```json
{
  "logs": [
    {
      "id": 1,
      "content": "Create a new REST API endpoint for user management",
      "type": "input",
      "time": "2019-08-24T14:15:22Z"
    },
    {
      "id": 2,
      "content": "I'll help you create a new REST API endpoint for user management...",
      "type": "output",
      "time": "2019-08-24T14:15:25Z"
    },
    {
      "id": 3,
      "content": "Now add validation for email addresses",
      "type": "input",
      "time": "2019-08-24T14:20:10Z"
    },
    {
      "id": 4,
      "content": "I'll add email validation to the user API endpoint...",
      "type": "output",
      "time": "2019-08-24T14:20:15Z"
    }
  ]
}
```

### Responses

| Status | Meaning                                                          | Description  | Schema                                                           |
|--------|------------------------------------------------------------------|--------------|------------------------------------------------------------------|
| 200    | [OK](https://tools.ietf.org/html/rfc7231#section-6.3.1)          | OK           | [codersdk.TaskLogsResponse](schemas.md#codersdktasklogsresponse) |
| 400    | [Bad Request](https://tools.ietf.org/html/rfc7231#section-6.5.1) | Bad Request  | -                                                                |
| 401    | [Unauthorized](https://tools.ietf.org/html/rfc7235#section-3.1)  | Unauthorized | -                                                                |
| 404    | [Not Found](https://tools.ietf.org/html/rfc7231#section-6.5.4)   | Not Found    | -                                                                |

---

## Task Status Types

Tasks have a computed status based on the state of the associated workspace, agent, and app:

| Status         | Description                                                         |
|----------------|---------------------------------------------------------------------|
| `pending`      | No workspace provisioned yet or status cannot be determined         |
| `initializing` | Workspace is being built, agent is connecting, or apps are starting |
| `active`       | Workspace is running, agent is ready, and apps are healthy          |
| `paused`       | Workspace has been stopped or deleted successfully                  |
| `error`        | Workspace build failed or apps are unhealthy                        |
| `unknown`      | Status cannot be determined from current workspace/agent/app state  |

The status is computed hierarchically:
1. If no workspace exists → `pending`
2. Else if build status is not "active" → use build status
3. Else if agent status is not "active" → use agent status
4. Else → use app status

## Task State Types

The `current_state` field provides additional context about what the task is currently doing:

| State      | Description                                 |
|------------|---------------------------------------------|
| `idle`     | Task is waiting for user input              |
| `working`  | Task is actively processing a request       |
| `complete` | Task has finished its current objective     |
| `failed`   | Task encountered an error during processing |

The `current_state` object also includes:
- `timestamp` - When this state was last updated
- `message` - Human-readable description of current activity
- `uri` - Optional URI reference (e.g., file path, GitHub PR URL)

## Implementation Notes

### Workspace Integration

Tasks are tightly integrated with the workspace system:
- Each task creates an associated workspace using a template version with `has_ai_task: true`
- The workspace name defaults to the task name
- Task status is derived from workspace build, agent, and app status
- Deleting a task triggers workspace deletion

### AI Capabilities

Templates must have AI task capabilities enabled to be used for tasks:
- Set `has_ai_task: true` on the template version
- Configure a workspace app to handle AI interactions
- The app receives input via the agent API

### Polling and Real-time Updates

The API does not currently support WebSocket or Server-Sent Events for real-time updates. Clients should:
- Poll `GET /experimental/tasks/{user}/{task}` every 5-10 seconds for task updates
- Poll `GET /experimental/tasks` every 10-60 seconds for task list updates
- Use the `current_state` field to determine if the task needs user attention

### Name Generation

If a task name is not provided during creation:
- The system will attempt to use configured AI (Anthropic API) to generate a descriptive name from the prompt
- Falls back to a generated name if AI generation fails
- Names must be unique per user and follow standard naming validation rules
