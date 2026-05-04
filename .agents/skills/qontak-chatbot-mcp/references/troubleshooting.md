# Troubleshooting

## Read The Response Envelope First

Every tool uses a normalized envelope.

Check these fields in order:
- `ok`
- `operation`
- `error.type`
- `error.message`
- `warnings`
- `meta.api_version_used`
- `meta.tree_version_used`

This usually tells you whether the issue is:
- input validation
- content type resolution
- missing graph context
- wrong transport choice
- tree verification fallback or failure

## Common Failure Cases

### `content_type_code could not be resolved`

Cause:
- `bot_response.content.content_type_code` does not match a known upstream content type

What to do:
1. Call `list_content_types(enabled=true)`.
2. Compare the actual content type codes returned by the server.
3. Retry the write with a valid code.

### `Unsupported content_type_code: <blank>` during path creation

Cause:
- `workflow_create_path_with_reply` received a malformed `root_reply` shape
- the tool could not resolve `root_reply.content.content_type_code` from the provided payload

What to do:
1. Send `root_reply` in the FE-aligned bot-response shape, not a simplified `{ type, payload }` shape.
2. Include `root_reply.name`.
3. Include `root_reply.content.content_text`, `content_type_code`, `channel_integration_id`, and `is_send_message`.
4. Retry only after confirming whether the path was already created.

### `Channel is not valid` during path creation

Cause:
- `path.channels[]` used the wrong identifier format
- request `channels[].id` was not the Hub integration UUID expected by path writes
- request may have included response-only fields like `integration_id`

What to do:
1. Use the Hub integration UUID as request `channels[].id`.
2. Do not send `integration_id` in the write payload.
3. Preserve the matching `name` and `target_channel`.
4. If your source is a path read, convert response `channels[].integration_id` into request `channels[].id`.

### `The name field is invalid` on `root_reply`

Cause:
- upstream root bot-response name validation rejected the supplied value
- descriptive names may be rejected even though a simple identifier succeeds

What to do:
1. Keep `root_reply.name` present.
2. Retry with a simple safe value like `Root`.
3. Treat the user-facing message text as `content.content_text`, not as the node name.

### Parent node was not found

Cause:
- the target parent id is wrong for `create_bot_response` or `add_branch`
- the path tree changed since the last read

What to do:
1. Re-read the tree with `get_path_tree`.
2. Confirm the current parent node id and node type.
3. Retry with the live parent reference.

### `changes.channel_integration_id` missing on `update_path_metadata`

Cause:
- the path patch payload cannot be reconstructed safely without it

What to do:
- always include `channel_integration_id` inside `changes`, even for partial updates
- resolve it from `list_chatbot_channel_integrations` if you only have Hub lookup data

### A Hub integration UUID was used as `channel_integration_id`

Cause:
- ids from `list_hub_integrations` or other Hub tools were reused in a chatbot path write payload

What to do:
1. Call `list_chatbot_channel_integrations`.
2. Pick the chatbot-local integer `id` for the correct channel integration record.
3. Retry the write with that integer id instead of the Hub identifier.

### Invalid operational-time payload on path update

Cause:
- merged path metadata becomes inconsistent

What to do:
- if `operational_time_type` is `schedule`, provide at least one schedule item
- if `operational_time_type` is `schedule`, make sure every `day_number` is in `0..6`
- if `operational_time_type` is `schedule`, make sure `start_hour` and `end_hour` use `HH:MM:SS`
- if `operational_time_type` is `schedule`, make sure `start_hour < end_hour`
- if `operational_time_type` is `date_range`, provide at least one date range item
- if `operational_time_type` is `date_range`, make sure every date value is a valid datetime string
- if `operational_time_type` is `date_range`, make sure `start_date < end_date`

### `bot_response.content must be an object`

Cause:
- the workflow wrapper or direct node update request is malformed

What to do:
- ensure `bot_response` is an object and contains a `content` object

### `branch.user_input must be an object` or `branch.bot_response must be an object`

Cause:
- `workflow_add_branch_with_reply` received an invalid wrapper shape

What to do:
- send `branch` with both `user_input` and `bot_response` objects

### Root bot responses cannot be deleted

Cause:
- `delete_bot_response` was pointed at the seeded root node

What to do:
- use `create_root_reply` to initialize the root
- use `update_bot_response` to edit it later
- do not attempt to remove it through `delete_bot_response`

### Multipart-only payload sent through JSON-only transport

Cause:
- `transport="json"` was forced while the payload required multipart or v3 routing

What to do:
- use `transport="auto"` unless there is a clear reason not to
- for file-backed attachments, allow multipart routing
- for conversation closure, allow v3 JSON routing

### Successful write but tree verification warnings appeared

Cause:
- the write succeeded but verification had to fall back or could not synthesize a refreshed tree cleanly

What to do:
1. Read `warnings`.
2. Inspect `meta.tree_version_used`.
3. If needed, manually re-read the tree with `get_path_tree`.
4. Treat the write result as authoritative for the mutation itself when the server reports a successful write with warnings.

### `workflow_create_path_with_reply` failed but duplicate paths appeared after retry

Cause:
- the workflow created the path first and failed later while initializing the root reply
- retries created additional paths because the earlier partial success was not checked

What to do:
1. After a failed create workflow, do not blindly retry.
2. Check recent paths with `list_paths` or a path read scoped to the intended name and channel.
3. Reuse the already created path when possible by initializing or updating its root reply.
4. Discard or clean up duplicate draft paths only when the user explicitly wants that cleanup.

### Hub lookup tools are unavailable during path creation

Cause:
- `list_hub_integrations` or normalization helpers are disabled or unavailable
- you still need a valid `path.channels[]` payload for the selected channel

What to do:
1. Use `list_paths` to find a trusted existing path already attached to the desired channel.
2. Read that response channel object.
3. Reuse response `integration_id` as request `channels[].id`.
4. Preserve the response `name` and `target_channel`.

### `add_branch` errors for `is_default is missing`

Cause:
- the `user_input` payload on `add_branch` is missing the required `is_default` field
- this field is required at branch creation time, unlike `update_user_input` which forbids it

What to do:
1. Include `is_default` as a boolean on every `user_input` passed to `add_branch`
2. Set `is_default=true` for exactly one branch per parent, `false` for others
3. If you need to toggle default branches later, use `set_default_user_input` after creation

### `add_branch` internal error "Created user input did not include an id"

Cause:
- the server created the user input but returned a malformed response
- the branch creation may still have succeeded despite the error response
- re-read with `get_node_detail` to verify actual state

What to do:
1. Do NOT immediately retry; the branch may already exist
2. Call `get_node_detail(path_id, "bot_response", parent_bot_response_id)` to check current children
3. If parent children are still unclear, read `get_path_tree(path_id, preferred_tree_version="v3")` as the primary branch-truth source
4. If `user_input` exists but child `bot_response` is missing, repair with `create_bot_response` on that `user_input` and `preferred_tree_version="v3"`
5. If the branch already exists with child reply, treat the tool call as successful despite the error response
6. If needed for additional compatibility context, read `get_path_tree(path_id, preferred_tree_version="v2")`

### `add_branch` error "Data user_input already exist with input = X"

Cause:
- a branch with the same user input text already exists under this parent
- this typically means a previous `add_branch` call succeeded, then was retried
- branching is not idempotent; duplicate user inputs are rejected

What to do:
1. Check the current tree with `get_node_detail` for the parent bot response
2. Verify whether the desired branch already exists with the correct child response
3. If it exists and has the correct content, consider the operation complete
4. If you need a different branch, use different user input text
5. If you need to update an existing branch's response, use `update_bot_response` instead

### Duplicate-input error appeared but the parent `get_node_detail` read still shows no children

Cause:
- the earlier branch write may have succeeded, but the immediate parent-node verification read is incomplete or stale
- `get_node_detail` is useful first, but it is not the only verification surface

What to do:
1. Do not retry again yet.
2. Re-read the full path with `get_path_tree(path_id, preferred_tree_version="v3")`.
3. Look for the new `user_input` and child `bot_response` in the full tree, not only the parent read.
4. If `user_input` exists without child `bot_response`, repair it with `create_bot_response` on that `user_input` and `preferred_tree_version="v3"`.
5. Treat the duplicate-input error as evidence of prior creation unless the full tree disproves it.

### `workflow_add_branch_with_reply` failed but a later retry hit duplicate user input

Cause:
- the workflow likely created the branch trigger during the earlier attempt
- the retry collided with the already-created `user_input`

What to do:
1. Verify the current branch state before any further write.
2. Use `get_node_detail` on the parent bot response first.
3. If the parent read is still unclear, use `get_path_tree` for the authoritative full-graph view.
4. Update the existing child bot response instead of trying to recreate the branch when the branch already exists.

### `add_branch` error "channel_integration_id is missing"

Cause:
- the `user_input` payload lacks the required `channel_integration_id` field
- this field must be the chatbot-local integer id from `list_chatbot_channel_integrations`

What to do:
1. Resolve `channel_integration_id` from `list_chatbot_channel_integrations` if not already known
2. Include it in the `user_input` object alongside `input` and `is_default`
3. Retry with the complete user_input payload

## Safe Recovery Pattern

When a write fails and the next step is unclear:

1. Re-read path metadata with `get_path_detail`.
2. Re-read the tree with `get_path_tree`.
3. Re-read the target node with `get_node_detail` if the task is node-scoped.
4. Compare the intended payload against the normalized examples in `payloads.md`.
5. Retry with the narrowest valid tool.

## When To Switch Tools

Switch from `create_bot_response` to `update_bot_response` when:
- the node already exists
- the payload includes interactive data
- the payload includes attachments
- the payload includes CRM, knowledge-source, or conversation-closure data

Switch from direct tools to workflow tools when:
- the task asks for path creation plus the first reply
- the task asks for branch creation plus the next reply
- the task asks to change an existing node component without caring about the lower-level mutation details

## Hub-Specific Failure Cases

### Hub request is pointed at the chatbot base URL

Cause:
- the Hub tool used `CHATBOT_API_BASE_URL` instead of `HUB_API_BASE_URL`

What to do:
1. confirm the tool is a Hub tool, not a chatbot path tool
2. verify `HUB_API_BASE_URL`
3. keep auth shared, but keep base URLs separate

### Client config works in FE but fails in MCP

Cause:
- FE forces staging for `GET /core/v1/client_configs/config`
- MCP or another client used the normal Hub base URL instead

What to do:
1. check whether the task explicitly needs FE-equivalent staging behavior
2. if yes, document and apply that override explicitly for the client-config call only
3. do not silently generalize the staging override to every Hub tool

### Seamless auth works in FE but not in a raw Hub caller

Cause:
- the Hub client config can enable auth behavior that later Hub calls depend on

What to do:
1. read Hub client config first when the flow depends on seamless auth
2. do not assume `X-Auth-Token` is always required
3. keep the rule tied to config, not to the whole Hub surface

### Integration list params look nested or wrong

Cause:
- some callers send a nested or malformed pagination shape instead of the flat payload the MCP contract expects

What to do:
1. treat `offset=1` and `limit=100` as the intended FE default values
2. call out the pagination-shape mismatch explicitly when debugging
3. do not turn the accidental nested shape into the MCP contract

### Organization-me request unexpectedly carries tag-style params

Cause:
- some callers add tag-style params to organization lookup even though the endpoint does not require them

What to do:
1. treat this as a FE wrapper quirk
2. do not require tag-style params for `get_hub_organization_me`
3. pass params only when the caller explicitly wants wrapper-level parity

### Qontak uniq returns `401`

Cause:
- FE treats this endpoint differently from most Hub requests and does not trigger the generic `/error` redirect path for it

What to do:
1. report the auth failure directly
2. do not assume the broader FE redirect behavior applies to this endpoint

### File upload fails after switching to JSON transport

Cause:
- FE sends an opaque body for `POST /core/v1/file_uploader/message`
- the upstream endpoint expects multipart-style upload semantics

What to do:
1. keep the MCP contract on explicit file content fields
2. translate those fields to multipart upstream
3. do not force JSON transport for Hub file upload
