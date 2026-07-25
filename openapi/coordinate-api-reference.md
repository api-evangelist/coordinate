# Coordinate REST API — Reference for Coding Agents

This document is the authoritative reference when writing code that talks to the Coordinate REST API. It is intentionally terse, complete, and free of marketing copy. If you are an LLM-driven coding agent (Claude, etc.), read this whole file before writing client code.

Base URL (production): `https://app.coordinatehq.com`
All paths below are relative to that base.

---

## 1. Authentication

Every endpoint requires an API key. The key is created in the Coordinate UI under **Settings → Integrations → API Keys**.

Send it as the following header:

```
Bearer: <your_api_key>
```

Missing or invalid key → `401 Unauthorized`. Server errors are returned as `500` with `{"success": false, "error": "..."}`.

All requests are scoped to the **vendor** that owns the API key. You cannot specify or override `vendor_id` — it is derived from the key and enforced on every read and write. There is no way to access another vendor's data with your key.

---

## 2. URL conventions

Two URL styles coexist; new code should use the **modern REST** form. The legacy paths are kept for backwards compatibility.

- Modern (use this): `POST /api/v1/projects`, `GET /api/v1/projects/<project_id>`, `POST /api/v1/projects/<project_id>` (update), etc.
- Legacy (don't use, but you'll see in old integrations): `POST /api/v1/create_project`, `POST /api/v1/project/project_id/<project_id>/update`, `GET /api/v1/list_projects`.

All endpoints below are listed in modern form.

---

## 3. Common patterns

### 3.1 Filtering list endpoints
Most list endpoints accept these query params:
- `last_modified_dt=<iso8601>` — return only items modified at or after this timestamp. **Important:** curl can replace `+` with a space in query strings; the server applies a `' ' → '+'` fixup, but URL-encode `+` as `%2B` to be safe.
- `sort=asc|desc` — sort by `last_modified_dt`. Default is `asc`.

### 3.2 Item shape
Every entity returned by the API includes:
- `entity_type` — `"Project" | "Task" | "Group" | "Goal" | "Stakeholder" | "ProgressReport" | "DiscussionEntry" | "Organization"`
- `entity_url` — shareable URL into the web app
- `last_modified_dt` — ISO 8601 with timezone
- `vendor_id` — your vendor (constant per key)
- `external_object_id` — optional foreign key you can set on creates/updates for integration bookkeeping

### 3.3 Error responses
- `400` — `{"success": false, "error": "<message>"}` for validation failures
- `404` — usually a plain-text body like `"Project Not Found"`
- `413` — file upload exceeded 100MB
- `500` — server error (also `{"success": false, "error": "Server Error"}`)

---

## 4. Endpoints

### 4.1 Projects

| Method | Path | Purpose |
|---|---|---|
| GET  | `/api/v1/projects` | List all projects (supports `last_modified_dt`, `sort`) |
| GET  | `/api/v1/projects/<project_id>` | Get one project |
| GET  | `/api/v1/projects/external_object_id/<external_object_id>` | Lookup by your external ID (returns a list — there may be more than one) |
| POST | `/api/v1/projects` | Create a project |
| POST | `/api/v1/projects/<project_id>` | Update a project |
| POST | `/api/v1/projects/<project_id>/apply_playbook` | Apply a template (a.k.a. "playbook") to an existing project |
| POST | `/api/v1/projects/<project_id>/files/attach` | Multipart upload, 100MB limit |
| GET  | `/api/v1/projects/<project_id>/file/<file_uid>` | Download an attached file. Returns a 302 to a fresh 5-minute S3 presigned URL on each hit. See [File downloads](#file-downloads). |

#### Project object
```json
{
  "entity_type": "Project",
  "project_id": "b73b9bde-78ff-4796-bd1e-c7c743ae5c87",
  "project_name": "MyProject",
  "project_description": null,
  "project_status": "Pre-Sale",
  "project_estimated_start_date": null,
  "project_estimated_end_date": null,
  "project_estimated_effort": null,
  "project_manager_email": "manager@example.com",
  "project_manager_full_name": "Manager Name",
  "project_tags": [],
  "project_active": true,
  "customers": [],
  "external_object_id": "somekey",
  "custom_fields": {
    "FieldCheckbox": false,
    "FieldString": "value"
  },
  "project_storage_json": {"attr1": "val71"},
  "entity_url": "https://app.coordinatehq.com/vendor/.../customer/.../CUSTOMER/...",
  "last_modified_dt": "2022-06-08T15:13:15.303225+00:00",
  "vendor_id": "0ab37cf1-fc60-4d93-b72b-89335f759581",
  "files": [
    {
      "file_uid": "abc-123",
      "file_name": "contract.pdf",
      "file_size": 84211,
      "file_content_type": "application/pdf",
      "file_dt": "2025-05-19T12:00:00+00:00",
      "download_url": "https://app.coordinatehq.com/api/v1/projects/b73b9bde-78ff-4796-bd1e-c7c743ae5c87/file/abc-123"
    }
  ]
}
```

#### Create project — full payload
```json
{
  "manager_email_address": "fixture_admin@dev.coordinate.net",  // REQUIRED. Must be an existing User in your vendor.
  "project_name": "MyProject",                                  // REQUIRED.
  "external_object_id": "somekey",                              // Optional.
  "organization_id": "uuid",                                    // Optional. Attach to an existing Org.
  "playbook_name": "Template1",                                 // Optional. Name of a PlanTemplate to apply.
  "playbook_date": "2025-04-08T16:44:48.792359+00:00",         // Optional. Anchor date for date-offset tasks.
  "playbook_date_title": "Task1",                               // Optional. Task title used as the anchor reference.
  "stakeholder_email": "client@example.com",                    // Optional. Invite this stakeholder.
  "stakeholder_invite_message": "Welcome",                      // Optional. Message in the invite email.
  "stakeholder_task_assignment_list": ["Example Task"],         // Optional. List of task TITLES (from the playbook) to assign to the invited stakeholder.
  "suppress_invite_email": false,                               // Optional. true = don't send invite email to stakeholder.
  "send_manager_assignment_email": false,                       // Optional. true = email the manager about assignment.
  "custom_fields": {"FieldName": "value"},                      // Optional. See "Custom fields" below.
  "project_storage_json": {"anything": "json"},                 // Optional. Integration bookkeeping bucket.
  "project_description": "...", "project_status": "...", "project_tags": ["..."], "..."
}
```

#### Custom fields
Custom fields must already be defined on the Vendor (in the UI). Three types:
- `Checkbox` — value coerced to bool
- `String` — value coerced to string
- `Dollars` — value coerced to `str(float(value))`

A field name not in the vendor's defined custom field list returns `404`. The supported `fieldType` values are exactly `Checkbox`, `String`, `Dollars`.

#### Update project
Send any subset of `project_*` keys. Read-only fields are silently dropped (`hasattr(project, key)` check). `manager_email_address` is special — it re-binds the project manager by looking up the user.

#### Apply playbook
Same `playbook_*` and `stakeholder_task_assignment_list` semantics as create. Also accepts:
```json
{
  "playbook_name": "T1",
  "playbook_date": "2022-04-08T16:44:48.792359+00:00",  // Anchor date for offset task dues
  "playbook_date_title": "Task1",                       // The task title whose due date == playbook_date
  "stakeholder_id": "uuid",                             // Optional. Existing stakeholder to assign tasks to.
  "stakeholder_task_assignment_list": ["Task1"]         // Task titles (in the playbook) to assign
}
```

#### File attach
`multipart/form-data` with one or more `File` parts. 100MB hard limit per request (`413` if exceeded).
```bash
curl -H "Bearer: $API_KEY" \
     -F File=@example1.txt \
     -F File=@example2.txt \
     https://app.coordinatehq.com/api/v1/projects/<project_id>/files/attach
```

---

### 4.2 Project Pages

| Method | Path | Purpose |
|---|---|---|
| GET  | `/api/v1/projects/<project_id>/pages` | List page names |
| GET  | `/api/v1/projects/<project_id>/pages/<page_name>` | Get a single page |
| POST | `/api/v1/projects/<project_id>/pages/<page_name>` | Create or overwrite a page |

#### Page object
```json
{
  "page_name": "Extended Information",
  "page_private": false,
  "page_content": "<p>HTML content</p>"
}
```

#### Create / overwrite payload
```json
{
  "page_content": "HelloWorld",   // REQUIRED. HTML allowed.
  "page_private": true            // Optional, default false. Private pages are not visible to stakeholders.
}
```

URL-encode the page name if it contains spaces (`Page Name` → `Page%20Name`).

---

### 4.3 Tasks

| Method | Path | Purpose |
|---|---|---|
| GET  | `/api/v1/projects/<project_id>/task` | List tasks in a project (supports `last_modified_dt`, `sort`) |
| GET  | `/api/v1/projects/<project_id>/task/<task_id>` | Get one task |
| GET  | `/api/v1/task/external_object_id/<external_object_id>` | Lookup tasks by external ID (returns list) |
| POST | `/api/v1/projects/<project_id>/task` | Create a task |
| POST | `/api/v1/projects/<project_id>/task/<task_id>` | Update a task |
| POST | `/api/v1/projects/<project_id>/task/<task_id>/files/attach` | Multipart upload, 100MB limit |
| GET  | `/api/v1/projects/<project_id>/task/<task_id>/file/<file_uid>` | Download an attached file. Returns a 302 to a fresh 5-minute S3 presigned URL on each hit. See [File downloads](#file-downloads). |

#### Task object
```json
{
  "entity_type": "Task",
  "task_id": "uuid",
  "project_id": "uuid",
  "project_name": "MyProject",
  "task_title": "Task Example",
  "task_description": null,
  "task_due_date": null,
  "task_status_current": null,
  "task_status_current_dt": null,
  "task_completed_dt": null,
  "task_completed_by_email": null,
  "task_completed_by_name": null,
  "group_id": null,
  "task_group_title": null,
  "task_assignee_stakeholder_id": null,
  "task_assignee_stakeholder_email_address": null,
  "task_assignee_stakeholder_full_name": null,
  "task_internal": false,
  "task_tags": ["Tag1"],
  "external_object_id": null,
  "time_tracking": [
    {
      "entry_id": "uuid",
      "entry_dt": "2023-11-09T18:17:12.329000+00:00",
      "hours": "5",
      "budget": false,
      "note": "Example",
      "stakeholder_id": "uuid",
      "stakeholder_email": "...",
      "stakeholder_full_name": "..."
    }
  ],
  "form_data": {
    "form_name": "Form",
    "completed_by": {"email": "...", "full_name": "...", "is_user": true, "user_stakeholder_id": "uuid", "completed_dt": "..."},
    "results": {"Field A": true, "Field B": "value"}
  },
  "files": [
    {
      "file_uid": "abc-123",
      "file_name": "spec.pdf",
      "file_size": 84211,
      "file_content_type": "application/pdf",
      "file_dt": "2025-05-19T12:00:00+00:00",
      "download_url": "https://app.coordinatehq.com/api/v1/projects/<project_id>/task/<task_id>/file/abc-123"
    }
  ],
  "entity_url": "...",
  "last_modified_dt": "...",
  "vendor_id": "..."
}
```

#### Create task payload
```json
{
  "task_title": "Task Example",                      // REQUIRED (effectively).
  "task_description": "<p>HTML</p>",                 // Optional.
  "task_due_date": "2025-12-31",                     // Optional. YYYY-MM-DD.
  "task_status_current": "in_progress",              // See valid values below.
  "task_internal": false,                            // Optional. If true, hidden from stakeholders.
  "task_tags": ["tag1"],                             // Optional.
  "external_object_id": "your-id",                   // Optional.
  "group_title": "Phase 1",                          // Optional. If matches an existing Group title in the project, task is placed in that group.
  "insert_top_of_group": false,                      // Optional. true = insert at top of the group's task list (re-numbers others).
  "task_assignment_json": "user@example.com"         // Optional. Assigns to user/stakeholder by email (see "Task assignment" below).
}
```

#### Valid task statuses
```
not_complete | information | in_progress | dependency_wait | blocked | complete
```

#### Task assignment
Send `task_assignment_json` as a **string** — the email address of an existing user or stakeholder on the project. The server resolves it to the proper assignment object.

---

### 4.4 Groups (Task Groups)

| Method | Path | Purpose |
|---|---|---|
| GET  | `/api/v1/projects/<project_id>/group` | List groups (supports `last_modified_dt`, `sort`) |
| GET  | `/api/v1/projects/<project_id>/group/<group_id>` | Get one group |
| POST | `/api/v1/projects/<project_id>/group` | Create a group |
| POST | `/api/v1/projects/<project_id>/group/<group_id>` | Update a group |

#### Group object
```json
{
  "entity_type": "Group",
  "group_id": "uuid",
  "group_title": "Group Example",
  "group_target_date": null,
  "group_completed_dt": null,
  "project_id": "uuid",
  "project_name": "MyProject",
  "entity_url": "...",
  "last_modified_dt": "...",
  "vendor_id": "..."
}
```

`group_*` keys on POST are translated to `milestone_*` internally — use the `group_*` names.

---

### 4.5 Stakeholders

| Method | Path | Purpose |
|---|---|---|
| GET  | `/api/v1/projects/<project_id>/stakeholder` | List stakeholders on a project (supports `last_modified_dt`, `sort`; augments with `stakeholder_roles` and `project_name`) |
| GET  | `/api/v1/projects/<project_id>/stakeholder/<stakeholder_id>` | Get one stakeholder |
| POST | `/api/v1/projects/<project_id>/stakeholder` | Invite a stakeholder to a project |

#### Stakeholder object
```json
{
  "entity_type": "Stakeholder",
  "stakeholder_id": "uuid",
  "stakeholder_email_address": "client@example.com",
  "stakeholder_full_name": null,
  "stakeholder_title": null,
  "stakeholder_phone": null,
  "stakeholder_project_id": "uuid",
  "stakeholder_related_org_id": null,
  "stakeholder_roles": ["Client Agent"],
  "external_object_id": null,
  "last_modified_dt": "...",
  "vendor_id": "..."
}
```

#### Create stakeholder payload
```json
{
  "stakeholder_email_address": "client@example.com",  // REQUIRED. Lowercased and trimmed server-side.
  "stakeholder_full_name": "Client Name",             // Optional.
  "stakeholder_title": "Director",                    // Optional.
  "stakeholder_phone": "+15555551212",                // Optional.
  "stakeholder_invite_message": "Welcome",            // Optional. Message body in invite email.
  "stakeholder_role": "Client Agent",                 // Optional. Must match a role defined on the vendor.
  "full_plan_access": true,                           // Optional, default true. false = limited-view stakeholder.
  "suppress_invite_email": false,                     // Optional, default false. true = no invite email sent.
  "send_role_assignment_emails": true,                // Optional, default true. false = no "task assigned" emails.
  "external_object_id": "your-id"                     // Optional.
}
```

**Edge case:** if the email belongs to an existing User (not a stakeholder), the user is added to the project's `user_stakeholders` list and the response is the string `"User added to project"` instead of a stakeholder object. Handle both shapes.

If the email is already a stakeholder on the project, returns `400` with `"collaborator with email address ... already exists on project"`.

---

### 4.6 Goals

| Method | Path | Purpose |
|---|---|---|
| GET  | `/api/v1/projects/<project_id>/goal` | List goals (supports `last_modified_dt`, `sort`) |
| GET  | `/api/v1/projects/<project_id>/goal/<goal_id>` | Get one goal |
| POST | `/api/v1/projects/<project_id>/goal` | Create a goal |
| POST | `/api/v1/projects/<project_id>/goal/<goal_id>` | Update a goal |
| GET  | `/api/v1/projects/<project_id>/goal/<goal_id>/file/<file_uid>` | Download an attached file. Returns a 302 to a fresh 5-minute S3 presigned URL on each hit. See [File downloads](#file-downloads). Note: there is no API endpoint to upload files to a goal today; goal files are typically attached via the web UI. |

#### Goal object
```json
{
  "entity_type": "Goal",
  "goal_id": "uuid",
  "goal_title": "Target Goal",
  "goal_description": null,
  "goal_target_date": null,
  "goal_completed_dt": null,
  "goal_completed_by_email": null,
  "goal_completed_by_name": null,
  "goal_sponsor_name": null,
  "project_id": "uuid",
  "project_name": "MyProject",
  "external_object_id": "somekey",
  "files": [
    {
      "file_uid": "abc-123",
      "file_name": "kickoff.pdf",
      "file_size": 84211,
      "file_content_type": "application/pdf",
      "file_dt": "2025-05-19T12:00:00+00:00",
      "download_url": "https://app.coordinatehq.com/api/v1/projects/<project_id>/goal/<goal_id>/file/abc-123"
    }
  ],
  "entity_url": "...",
  "last_modified_dt": "...",
  "vendor_id": "..."
}
```

---

### 4.7 Progress Reports (read-only)

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/v1/projects/<project_id>/progress_report` | List (supports `last_modified_dt`, `sort`) |
| GET | `/api/v1/projects/<project_id>/progress_report/<progress_report_id>` | Get one |

#### Progress Report object
```json
{
  "entity_type": "ProgressReport",
  "progress_report_id": "uuid",
  "progress_report_date": "2025-06-08",
  "progress_report_description": "...",
  "progress_report_next_steps": "...",
  "progress_report_project_status": "Pre-Deployment",
  "progress_report_reporter_email": "...",
  "progress_report_reporter_full_name": "...",
  "progress_report_user_id": "uuid",
  "project_id": "uuid",
  "external_object_id": null,
  "last_modified_dt": "...",
  "vendor_id": "..."
}
```

---

### 4.8 Discussion Entries (Comments)

| Method | Path | Purpose |
|---|---|---|
| GET  | `/api/v1/projects/<project_id>/discussion_entry` | Project-level comments |
| GET  | `/api/v1/projects/<project_id>/task/<task_id>/discussion_entry` | Task comments |
| GET  | `/api/v1/projects/<project_id>/goal/<goal_id>/discussion_entry` | Goal comments |
| POST | `/api/v1/projects/<project_id>/task/<task_id>/discussion_entry` | Add a comment to a task |
| GET  | `/api/v1/projects/<project_id>/comments` | **All** comments on a project (project + tasks + goals), filterable by `last_modified_dt`/`sort` |

All GET endpoints accept `last_modified_dt` and `sort`.

#### Discussion entry object
```json
{
  "entity_type": "DiscussionEntry",
  "discussion_author_name": "Author Name",
  "discussion_comment": "<p>HTML comment</p>",
  "discussion_timestamp_dt": "2025-06-08T16:00:54.701622+00:00",
  "discussion_entry_internal": false,
  "target_entity_id": "uuid",
  "target_entity_type": "Task",
  "target_entity_title": "Task Title",
  "project_id": "uuid",
  "project_name": "Example",
  "entity_url": "...",
  "last_modified_dt": "..."
}
```

#### POST a task comment
```json
{
  "sender_email_address": "fixture_admin@dev.coordinate.net",  // REQUIRED. Must be a user or stakeholder on the project.
  "comment": "Test Comment"                                    // REQUIRED. HTML allowed.
}
```
Returns `200 "DiscussionEntry Created"`. Unknown sender → `404 "Unknown email address"`.

---

### 4.9 Organizations

| Method | Path | Purpose |
|---|---|---|
| GET  | `/api/v1/organizations` | List all orgs |
| GET  | `/api/v1/organizations/<org_id>` | Get one org |
| GET  | `/api/v1/organizations/<org_id>/stakeholders` | Stakeholders linked to the org |
| GET  | `/api/v1/organizations/<org_id>/projects` | Projects linked to the org |
| POST | `/api/v1/organizations` | Create an org |
| POST | `/api/v1/organizations/<org_id>/stakeholders` | Add a stakeholder to an org (propagates to all linked projects) |
| POST | `/api/v1/organizations/<org_id>/stakeholders/<stakeholder_id>/remove` | Remove stakeholder from org (and all related stakeholders for the same email on the same org) |
| POST | `/api/v1/organizations/<org_id>/projects` | Link an existing project to an org |
| POST | `/api/v1/organizations/<org_id>/projects/<project_id>/remove` | Unlink a project from an org (removes related stakeholders) |

#### Organization object
```json
{
  "entity_type": "Organization",
  "organization_id": "uuid",
  "organization_name": "Example Org",
  "organization_description": null,
  "external_object_id": null,
  "last_modified_dt": "...",
  "vendor_id": "..."
}
```

#### Create org payload
```json
{
  "organization_name": "Example Org",     // REQUIRED.
  "external_object_id": "your-id"         // Optional.
}
```

#### Add stakeholder to org payload
```json
{
  "stakeholder_email_address": "client@example.com",  // REQUIRED.
  "stakeholder_full_name": "Client Name",             // Optional.
  "stakeholder_title": "Director",                    // Optional.
  "stakeholder_phone": "+15555551212"                 // Optional.
}
```

#### Link project to org payload
```json
{ "project_id": "uuid" }
```

---

### 4.10 The `/entity` firehose

```
GET /api/v1/entity
```

Returns all entities of all types in a vendor's account, optionally filtered. Use this for bulk sync / data warehouse extracts.

| Param | Required | Description |
|---|---|---|
| `start_dt` (alias: `last_modified_dt`) | No | ISO 8601. Only entities modified at or after this dt. |
| `end_dt` | No | ISO 8601. Only entities modified at or before this dt. |
| `entity` | No | Filter to one type: `Task`, `Goal`, `Project`, `Stakeholder`, `Org` (translated internally to `Customer`/`Milestone`/`StakeholderV2` etc.). |
| `sort` | No | `asc` (default) or `desc`, by `last_modified_dt`. |
| `paginate` | No | `true` to enable pagination. Default `false` returns up to 100 results. |
| `page` | No | 1-indexed page number. Empty array = no more pages. |
| `page_size` | No | Page size, default 100. |

Example:
```bash
curl -G -H "Bearer: $API_KEY" \
     -d "start_dt=2025-05-01T00:00:00" \
     -d "end_dt=2026-01-01T00:00:00" \
     -d "entity=Task" \
     -d "sort=desc" \
     https://app.coordinatehq.com/api/v1/entity
```

Pagination:
```bash
curl -G -H "Bearer: $API_KEY" \
     -d "paginate=true" -d "page=1" -d "page_size=50" -d "entity=Project" \
     https://app.coordinatehq.com/api/v1/entity
```

When `paginate=true`, the response is always sorted ascending by `last_modified_dt` to guarantee stable page boundaries.

---

### 4.11 Users (read-only)

```
GET /api/v1/list_users
```
Returns all users in your vendor. Use to discover valid `manager_email_address` values.

---

### 4.12 JSON Storage (scratch space)

A single per-vendor JSON blob for integration bookkeeping (e.g., "last sync cursor", "last seen external ID"). 300KB limit.

```
GET  /api/v1/json_storage           # returns {} if never set
POST /api/v1/json_storage           # body = arbitrary JSON, overwrites previous
```

---

### 4.13 File downloads <a id="file-downloads"></a>

Files attached to a Project, Task, or Goal show up as a `files` array on the entity's JSON. Each entry is self-describing and includes a `download_url`:

```json
{
  "file_uid": "abc-123",
  "file_name": "contract.pdf",
  "file_size": 84211,
  "file_content_type": "application/pdf",
  "file_dt": "2025-05-19T12:00:00+00:00",
  "download_url": "https://app.coordinatehq.com/api/v1/projects/<project_id>/file/abc-123"
}
```

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/v1/projects/<project_id>/file/<file_uid>` | Download a project-level file |
| GET | `/api/v1/projects/<project_id>/task/<task_id>/file/<file_uid>` | Download a task file |
| GET | `/api/v1/projects/<project_id>/goal/<goal_id>/file/<file_uid>` | Download a goal file |

**Semantics:**
- The `download_url` field is a **permanent API endpoint**. It does not expire and can be stored indefinitely (e.g. in your CRM or data warehouse).
- Each request to `download_url` requires your `Bearer:` API key header.
- On every hit, the server mints a **fresh 5-minute S3 presigned URL** and returns a `302` redirect to it. Follow the redirect immediately; the redirect target is short-lived and must not be cached or stored.
- `404` if the `file_uid` isn't attached to that exact entity (prevents enumeration across other entities).
- `403` if the vendor has files disabled.

**Example:**
```bash
curl -L -H "Bearer: $API_KEY" \
     https://app.coordinatehq.com/api/v1/projects/$PROJECT_ID/file/$FILE_UID \
     -o downloaded.pdf
```

If a `file_uid` contains a `#` (some files have a `#<content_type>` suffix), URL-encode it as `%23` when calling. The `download_url` field returned in entity JSON does this for you.

---

## 5. Webhooks

### 5.1 Subscription
```
POST   /api/v1/webhook_subscribe?hookUrl=<your_https_url>&endpoint_nickname=<optional>
DELETE /api/v1/webhook_subscribe/<endpoint_id>
```
The subscribe response returns `{"id": "<endpoint_id>"}`. Keep this ID for later unsubscribe.

### 5.2 Event shape
Every event has:
```
entity              - object with the current property values
entity_type         - "Project" | "Task" | "Group" | "Goal" | "Customer" | "DiscussionEntry"
entity_action       - "create" | "update"
entity_previous_values  - (update only) values before the change
entity_updated_values   - (update only) {field: bool} flags indicating which fields changed
```

Note that for webhook payloads, `entity_type: "Customer"` refers to a **customer organization linked to a project**, not a project itself. This is a legacy naming wart.

### 5.3 Events

- **Project Created/Updated** — `entity_type: "Project"`. Same shape as the Project object above; webhook payload also includes a `customers` array of `{customer_id, customer_name}`.
- **Task Created/Updated** — `entity_type: "Task"`.
- **Group (Task Group) Created/Updated** — `entity_type: "Group"`.
- **Goal Created/Updated** — `entity_type: "Goal"`.
- **Customer Created/Updated** — `entity_type: "Customer"`. The customer-org attached to a project.
- **Comment Added** — `entity_type: "DiscussionEntry"`, `entity_action: "create"` only.

To interpret an update: check `entity_updated_values.<field>` for `true`, then compare `entity_previous_values.<field>` to `entity.<field>`. Most updates also touch `last_modified_dt` — skip events where that is the only `true` flag if you want to avoid noise (the server filters those out of `webhook_sample` responses).

---

## 6. Gotchas and tips

1. **`vendor_id` is implicit.** Never send it in payloads, never expect to be able to filter by it. The API key fixes it.
2. **Project = Customer.** In any internal Python code you read, "Customer" is your "Project". Webhook events use "Customer" to mean *organization attached to a project* — entirely different concept.
3. **Group = Milestone.** `group_*` keys are translated to `milestone_*` on writes. Use `group_*` externally.
4. **`task_assignment_json` on update is a plain string (email),** not an object. The server resolves the email to the proper assignment record.
5. **Two `+` URL encodings are wrong.** `iso8601` timestamps contain `+00:00` for UTC. `curl` and many HTTP clients re-encode `+` as space. The server applies `' ' → '+'` fixup before parsing, but URL-encode `+` as `%2B` if your client lets you.
6. **File upload field name is exactly `File`** (case-sensitive). Multiple `File=@path` parts in one request are accepted.
7. **`POST /api/v1/projects/<id>/stakeholder`** can return either a Stakeholder JSON object **or** the string `"User added to project"` if the email belongs to an existing User. Be defensive.
8. **`external_object_id` lookups return lists,** even though usually one entity matches. Do not assume singleton.
9. **Custom fields must be predefined on the vendor.** Posting a value for an undefined field returns `404`, not `400`.
10. **Don't rely on the `success: true` shape on success.** Most successful responses return the JSON of the entity directly. Only failures use `{"success": false, "error": "..."}`. A few endpoints return `jsonify(success=True)` (e.g., apply_playbook, remove operations); most do not.
11. **File `download_url` is the permanent API route — not a presigned S3 URL.** The S3 presigned URL lives only in the `Location` header of the 302 response and expires in 5 minutes. Always hit `download_url` with the `Bearer:` header and follow the redirect immediately; never store the redirect target.

---

## 7. Minimal worked examples

### Create a project, apply a playbook, invite a stakeholder and assign one task
```bash
curl -X POST -H "Content-Type: application/json" -H "Bearer: $API_KEY" \
     https://app.coordinatehq.com/api/v1/projects \
     -d '{
       "manager_email_address": "manager@yourco.com",
       "project_name": "ACME Onboarding",
       "external_object_id": "crm-12345",
       "playbook_name": "Standard Onboarding",
       "stakeholder_email": "client@acme.com",
       "stakeholder_task_assignment_list": ["Sign SOW"],
       "suppress_invite_email": false
     }'
```

### Mark a task complete
```bash
curl -X POST -H "Content-Type: application/json" -H "Bearer: $API_KEY" \
     https://app.coordinatehq.com/api/v1/projects/$PROJECT_ID/task/$TASK_ID \
     -d '{"task_status_current": "complete"}'
```

### Bulk-pull recently updated tasks
```bash
curl -G -H "Bearer: $API_KEY" \
     --data-urlencode 'start_dt=2025-05-01T00:00:00+00:00' \
     -d 'entity=Task' -d 'sort=desc' \
     https://app.coordinatehq.com/api/v1/entity
```

### Add a comment on a task
```bash
curl -X POST -H "Content-Type: application/json" -H "Bearer: $API_KEY" \
     https://app.coordinatehq.com/api/v1/projects/$PROJECT_ID/task/$TASK_ID/discussion_entry \
     -d '{"sender_email_address": "user@yourco.com", "comment": "Looks good."}'
```

### Persist a sync cursor
```bash
curl -X POST -H "Content-Type: application/json" -H "Bearer: $API_KEY" \
     https://app.coordinatehq.com/api/v1/json_storage \
     -d '{"last_sync_dt": "2025-05-19T12:00:00+00:00"}'
```
