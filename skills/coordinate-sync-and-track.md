---
name: Sync activity and track task progress
description: Bulk-pull recently changed Coordinate entities, subscribe to webhooks, and mark tasks complete with a comment.
api: openapi/coordinate-openapi.yml
operations: [listEntities, subscribeWebhook, updateTask, createTaskDiscussionEntry, getJsonStorage, setJsonStorage]
---

# Sync activity and track task progress

Use this to keep an external system (CRM, warehouse) in sync with Coordinate and
to drive task status from your workflow.

## Auth
Send your API key in a custom header: `Bearer: <API_KEY>`.

## Incremental sync
1. **Read your cursor.** Call `getJsonStorage` (`GET /json_storage`) to fetch the
   last saved sync timestamp (returns `{}` if never set).
2. **Pull changes.** Call `listEntities` (`GET /entity`) with
   `start_dt=<last_cursor>`, optionally `entity=Task|Goal|Project|Stakeholder|Org`,
   and `sort=desc`. For large result sets pass `paginate=true`, `page`, and
   `page_size`; an empty array means no more pages. Note timestamps contain
   `+00:00` — URL-encode `+` as `%2B`.
3. **Persist the new cursor.** Call `setJsonStorage` (`POST /json_storage`) with
   the newest `last_modified_dt` you processed (300KB blob limit).

## Real-time (webhooks)
4. **Subscribe.** Call `subscribeWebhook`
   (`POST /webhook_subscribe?hookUrl=<https_url>`); keep the returned `id` for
   later `unsubscribeWebhook`. Events carry `entity`, `entity_type`,
   `entity_action`, and on updates `entity_updated_values` (per-field bool flags).
   Skip events whose only changed field is `last_modified_dt`.

## Drive task status
5. **Complete a task.** Call `updateTask`
   (`POST /projects/{project_id}/task/{task_id}`) with
   `{"task_status_current": "complete"}`. Valid statuses: `not_complete`,
   `information`, `in_progress`, `dependency_wait`, `blocked`, `complete`.
6. **Leave a note.** Call `createTaskDiscussionEntry`
   (`POST /projects/{project_id}/task/{task_id}/discussion_entry`) with
   `sender_email_address` (must be a user/stakeholder on the project) and
   `comment`. Unknown sender returns 404.

## Rules
- `vendor_id` is implicit — never send it.
- `external_object_id` lookups return lists; do not assume a singleton.
- Successful writes usually return the entity JSON directly; only failures use
  `{"success": false, "error": "..."}`.
