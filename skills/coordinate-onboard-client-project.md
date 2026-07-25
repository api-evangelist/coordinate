---
name: Onboard a client project
description: Create a Coordinate project from a playbook, invite the client stakeholder, and assign their first task.
api: openapi/coordinate-openapi.yml
operations: [listUsers, createProject, applyPlaybook, inviteStakeholder, listTasks]
---

# Onboard a client project

Use this to spin up a new client project in Coordinate, seed it from a template
(playbook), and invite the client as a stakeholder.

## Auth
Send your API key in a custom header: `Bearer: <API_KEY>`. Every request is
scoped to the vendor that owns the key.

## Steps
1. **Find a valid manager email.** Call `listUsers` (`GET /list_users`) and pick
   the `email` of the user who will own the project — `manager_email_address`
   must be an existing user in your vendor.
2. **Create the project.** Call `createProject` (`POST /projects`) with required
   `manager_email_address` and `project_name`. You can seed it in one shot by
   also passing `playbook_name`, `stakeholder_email`, and
   `stakeholder_task_assignment_list`. Set `external_object_id` to your CRM id
   for later reconciliation. Capture `project_id` from the response.
3. **Apply a playbook (if not done at create).** Call `applyPlaybook`
   (`POST /projects/{project_id}/apply_playbook`) with `playbook_name` and an
   optional `playbook_date` anchor for date-offset tasks.
4. **Invite the client stakeholder.** Call `inviteStakeholder`
   (`POST /projects/{project_id}/stakeholder`) with `stakeholder_email_address`.
   Be defensive: the response is a Stakeholder object OR the string
   `"User added to project"` if the email is an existing user. A duplicate
   returns `400`.
5. **Verify tasks.** Call `listTasks` (`GET /projects/{project_id}/task`) to
   confirm the playbook created the expected tasks.

## Rules
- `manager_email_address` is required and must resolve to a real user (else 404).
- Custom fields must be predefined on the vendor; an undefined field returns 404.
- No idempotency key exists — use `external_object_id` to detect an existing
  project before creating a duplicate (`getProjectsByExternalId`).
