# Tool Catalog

## Read Tools

### `list_paths(channel_integration_id, page=1, limit=20, order_by="name", order_direction=None, query=None)`

Use when you need to:
- find an existing Conversation by name
- verify whether a failed create workflow already produced a path
- inspect response channel metadata and recover `integration_id` values for later writes

Inputs:
- `channel_integration_id`: required chatbot-local integer id (from `list_chatbot_channel_integrations`)
- `page`, `limit`, `order_by`, `order_direction`, `query`: all optional

Example call:
- `list_paths(channel_integration_id=123, query="Order status", limit=20)`

Notes:
- returns a paginated list of paths
- useful fallback when `get_path_detail` or Hub integration lookup tools are unavailable
- response channel objects include both chatbot-local `id` and Hub `integration_id`
- when reusing channel data for a write, map response `integration_id` to request `channels[].id`

### `get_path_detail(path_id)`

Use when you need normalized path metadata from `/v2/paths/{id}`.

UI wording note:
- in the frontend, this is Conversation detail

Notes:
- returns `result.path_metadata`
- uses `meta.api_version_used = "v2"`
- useful before any path-level update

### `get_path_tree(path_id, preferred_tree_version="v3")`

Use when you need the normalized conversation graph for one path.

Notes:
- returns `result.tree`
- default tree preference is `v3`
- `v3` is the only backend version serving `tree_diagram`; the `v1`/`v2` tree routes were retired and now 404 (dead fallbacks only)
- `v3` also uniquely surfaces interactive and `whatsapp_flow` node detail
- actual fallback outcome is returned in `meta.tree_version_used`

### `get_node_detail(path_id, node_type, node_id, preferred_tree_version="v3")`

Use when you need one normalized node plus its direct children.

Inputs:
- `node_type` must be `bot_response` or `user_input`

Notes:
- good for targeting a specific branch before mutation
- returns `result.node`, `result.parent`, and `result.children`

### `list_chatbot_channel_integrations()`

Use when you need the chatbot-local integer `channel_integration_id` required by path metadata writes.

Notes:
- endpoint: `GET /v1/channel_integrations`
- default request shape: `order_by=created_at`
- returns `result.channel_integrations`
- do not substitute ids from `list_hub_integrations`

### `list_content_types(enabled=true)`

Use when you need to inspect or confirm content type codes before composing a bot response payload.

Notes:
- returns `result.items`
- useful when debugging unresolved `content_type_code`
- the bot-response build path uses `content_type_code` values `text`, `button`, `list`, and `ai_assist`; resolve the live set via this tool
- `button` and `list` require a matching `interactive` payload

### `list_ai_agents(page=0, limit=20, order_by="name", order_direction=None, status=None, query=None)`

Use to resolve an AI agent's `id` (UUID) before `create_ai_agent_response`; use that `id` as `contentable_id`.

Inputs:
- `order_by` must be one of: `name`, `created_by`, `status`, `usage`, `last_update`, `ai_agent_name`, `updated_at`
- `page`, `limit`, `order_direction`, `status`, `query`: all optional

Notes:
- endpoint: `GET /v1/ai_agents` (`meta.api_version_used = "v1"`)
- returns a paginated payload under `result.ai_agents` with `data[]` and `meta`
- each `data[]` item has `id` (string UUID), `name`, `status`, `usages`, `usage_conversations`, `created_by`, `updated_at`

## Path Tools

### `create_path(path)`

Use when you only need path metadata creation.

UI wording note:
- use this when the user asks to create a Conversation and they mean metadata only

Expected input family:
- normalized `PathMetadata`

Notes:
- creates path metadata only
- for Conversation create, keep `channel_integration_id` separate from `channels[]`
- resolve `channel_integration_id` from `list_chatbot_channel_integrations`
- `channel_integration_id` must be valid for the chatbot org, but it does not need to map 1:1 to the selected `channels[]`
- build `channels[]` from Hub selections, usually through `normalize_hub_integrations_to_path_channels`
- request `channels[].id` must be the Hub integration UUID; if your source is a path read, use response `channels[].integration_id` here
- does not initialize the seeded root reply
- prefer `workflow_create_path_with_reply` when the task also needs the first message

### `update_path_metadata(path_id, changes)`

Use when changing path-level metadata without editing the conversation tree.

UI wording note:
- use this when the user asks to update Conversation settings or Conversation metadata

Important:
- `changes.channel_integration_id` is required even for partial updates
- resolve it from `list_chatbot_channel_integrations` when needed
- `changes.channel_integration_id` must be a valid chatbot-local integer id, but it does not need to map 1:1 to the selected `changes.channels[]`
- `changes.channels` should be the minimal Hub selection payload shape, not raw Hub objects
- use `normalize_hub_integrations_to_path_channels` when the source data comes from `list_hub_integrations`
- `operational_time_type` follows the same fixed enum used by FE and BE path CRUD: `24hours`, `schedule`, `date_range`
- there is no dedicated list tool for operational time types; pass one of the supported values directly in the path payload
- when `operational_time_type` is `schedule`, each schedule item must use `day_number` in `0..6`, `start_hour` and `end_hour` in `HH:MM:SS`, and `start_hour < end_hour`
- when `operational_time_type` is `date_range`, each date range item must use valid datetime strings and `start_date < end_date`
- merged `operational_time_type` must remain valid after the change

### `publish_path(path_id, strategy="auto", body=None, preferred_tree_version="v3")`

Use when draft changes should be published and verified with a refreshed tree read.

Notes:
- returns refreshed `path_metadata` and `tree`
- `strategy` can be `v1`, `v2`, or `auto`

### `discard_path_draft(path_id, body=None, preferred_tree_version="v3")`

Use when draft changes should be discarded and verified with a refreshed tree read.

Notes:
- returns refreshed `path_metadata` and `tree`

## Conversation Write Tools

### `create_root_reply(path_id, bot_response, preferred_tree_version="v3")`

Use only for the seeded root bot response that already exists after path creation.

Notes:
- does not create a second root node
- patches the existing root through the FE-aligned v2 flow
- resolves content type at tool entry

### `create_bot_response(path_id, parent, bot_response, preferred_tree_version="v3")`

Use when you need a non-root child reply under an existing `bot_response` or `user_input` node.

Inputs:
- `parent.type` must be `bot_response` or `user_input`

Notes:
- stays on the FE child-create flow
- intended for text-content child creation plus the JSON-safe component groups validated for child create
- for interactive, attachment, CRM, knowledge, or conversation-closure edits, update an existing node instead

ai_assist support:
- this tool now creates a knowledge-backed `ai_assist` reply: pass `content.content_type_code="ai_assist"` plus TOP-LEVEL `ai_api_knowledge` and/or `knowledge_sources` (siblings of `content`/`components`)
- `ai_api_knowledge` shape: `{ "enabled": true, "sources": [{ "id": <int>, "api_connection_id": <int> }] }`
- `knowledge_sources` shape: `[{ "id": <int> }]`
- the backend `POST /v1/bot_responses` ai_assist branch wires the knowledge during create (`meta.api_version_used = "v1"`)
- for the seeded root node use `create_root_reply` instead, where these knowledge families nest INSIDE `components`

### `add_branch(path_id, parent_bot_response_id, user_input, bot_response, preferred_tree_version="v3")`

Use when the task is "add a branch under this bot response and attach the next reply".

Required fields on `user_input`:
- `input`: The user-facing trigger text
- `channel_integration_id`: The chatbot-local integer id (from `list_chatbot_channel_integrations`)
- `is_default`: Boolean flag; required at creation time (exactly one branch per parent should have `is_default=true`)

Notes:
- creates the `user_input` first, then the child `bot_response`
- returns both created nodes and the refreshed tree
- partial failures keep created ids and do not auto-delete them
- do NOT use `is_default` with `update_user_input`; use `set_default_user_input` for toggling later
- the user_input creation may return an error like "Created user input did not include an id", but the branch may still be created (verify with re-read)

### `create_exit_condition(path_id, parent_bot_response_id, exit_condition, is_default=False, preferred_tree_version="v3")`

Use when you need an exit-condition user input under a bot response (an intent that routes to a typed downstream node).

`exit_condition` fields:
- `name`: optional
- `description`: optional
- `trigger`: optional
- `next_intent_type`: optional; one of `TEXT`, `BUTTON`, `LIST`, `BRANCH`, `AI_AGENT`, `WHATSAPP_FLOW` (auto-uppercased)
- `ai_agent_id`: required only when `next_intent_type=AI_AGENT`

Notes:
- creates a user input with `response_type="exit_condition"` via `POST /v2/user_inputs` (`meta.api_version_used = "v2"`)
- the tool auto-sets `parameters.path_id`; do not pass it yourself
- when `next_intent_type=AI_AGENT`, the feature may be billing-gated and the backend can return 403
- returns `result.user_input` and `result.tree`

### `create_ai_agent_response(path_id, ai_agent, parent=None, preferred_tree_version="v3")`

Use when you need an `ai_agent` reply node bound to a pre-existing AI agent.

`ai_agent` fields:
- `name`: required, must be unique
- `contentable_id`: required; the agent UUID resolved from `list_ai_agents`
- `content_type_version`: required (e.g. `1`)
- `channel_integration_id`: injected from path metadata when omitted
- exactly one of `previous_bot_response_id` / `previous_user_input_id` (or pass `parent` as `{ type, id }`)

Notes:
- writes to `POST /v2/bot_responses` with `contentable_type` fixed to `ai_agent` server-side (`meta.api_version_used = "v2"`)
- the agent must already exist; resolve `contentable_id` with `list_ai_agents` first
- the endpoint returns no id, so the tool resolves the new node by diffing the tree and matching the unique `name`
- returns `result.bot_response` and `result.tree`

### `create_branch_condition(path_id, branch, parent=None, preferred_tree_version="v3")`

Use to create the CONDITION branch node (content type `branch`), whose API-driven `response_conditions` route to one downstream node per matched condition.

Distinction:
- this is DISTINCT from `add_branch`, which adds a conversational user-input tree branch (a menu choice). `create_branch_condition` builds the condition-routing node, not a menu option.

`branch` fields:
- `name`: required, must be unique
- `channel_integration_id`: injected when omitted
- exactly one of `previous_bot_response_id` / `previous_user_input_id` (or pass `parent` as `{ type, id }`)
- `id`: optional; re-creates an existing intent as a branch, destroying its children
- `api`: optional block (see below)

`api` block fields:
- `enabled`: required bool
- `connection_id`, `path`, `method` (one of `GET`/`POST`/`PUT`/`PATCH`/`DELETE`/`COPY`/`HEAD`/`OPTIONS`): all required when `enabled=true`
- optional: `headers`, `body`, `success_key`, `success_value`, `fallback_bot_response_id`, `is_api_entity`, `entities[]`, `response_conditions[]`

`response_conditions[]` fields:
- `sequence`: required int
- `next_intent_type`: required on create; one of `TEXT`, `BUTTON`, `LIST`, `AI_ASSIST`
- `is_default`: optional
- `criterias[]`: see below

`criterias[]` fields:
- `key`: required
- `operator`: required
- `sequence`: required int
- `value`: optional
- `condition_operator`: optional

Notes:
- writes to `POST /v1/bot_responses/branches` (`meta.api_version_used = "v1"`)
- the backend auto-creates each condition's downstream node
- the endpoint returns no id, so the node is resolved by name diff
- returns `result.bot_response` and `result.tree`

### `update_branch_condition(path_id, bot_response_id, changes, preferred_tree_version="v3")`

Use to edit an existing condition-branch node.

`changes` fields:
- `name`: required
- `channel_integration_id`: injected when omitted
- `api`: optional; same `api` / `response_conditions` / `criterias` structure as `create_branch_condition`

Differences from create:
- `next_intent_type` is not enforced on update
- `response_conditions[]` and `criterias[]` may additionally carry `id` and `action_status` (`create` / `update` / `delete`) for diff-style edits of existing rows

Notes:
- writes to `PATCH /v1/bot_responses/branches/{bot_response_id}` (`meta.api_version_used = "v1"`)
- returns `result.bot_response` and `result.tree`

### `update_bot_response(path_id, bot_response_id, bot_response, transport="auto", preferred_tree_version="v3")`

Use when editing content and component configuration for an existing bot response.

Transport rules:
- `auto` keeps v2 JSON for ordinary JSON-safe edits
- `auto` routes conversation closure to v3 JSON
- `auto` routes file-backed attachment uploads to v3 multipart
- `json` rejects multipart-only payloads

Notes:
- this is the main tool for advanced component edits
- returns the refreshed node and tree

### `update_user_input(path_id, user_input_id, changes, preferred_tree_version="v3")`

Use for ordinary user input edits.

Accepted `changes` fields (validates with extra=forbid, so anything else is rejected):
- `input`: string, the user-facing trigger text
- `organization_entity_id`: integer
- `utterances`: list of `{ "text": "...", "organization_nlp_connector_id"?: int }`
- `organization_nlp_connector_id`: integer

Example `changes`:
- `{ "input": "Check status", "utterances": [ { "text": "where is my order" } ] }`

Important:
- `is_default`, `channel_integration_id`, and `bot_response_id` are NOT accepted here and will be rejected
- default toggling belongs to `set_default_user_input`

### `set_default_user_input(path_id, user_input_id, is_default=true, preferred_tree_version="v3")`

Use to toggle the default branch state for one user input.

### `delete_user_input(path_id, user_input_id, mode="single_block", preferred_tree_version="v3")`

Use to delete a user input.

Modes:
- `single_block`
- `with_children`
- `only_user`

Notes:
- safe default is `single_block`
- `meta.api_version_used` varies by mode

### `delete_bot_response(path_id, bot_response_id, mode="single", preferred_tree_version="v3")`

Use to delete a non-root bot response.

Modes:
- `single`
- `with_children`
- `unused`

Important:
- root bot responses cannot be deleted through this tool

## WhatsApp Flow Tools

### `create_whatsapp_flow(path_id, whatsapp_flow, parent=None, preferred_tree_version="v3")`

Use when you need to attach a WhatsApp Flow bot response under an existing node.

Notes:
- writes to v1 `POST /v1/bot_responses/whatsapp_flow` (`meta.api_version_used = "v1"`)
- read-back is verified via the v3 tree
- `whatsapp_flow` nested object required fields: `template_id`, `external_id`, `template_name`, `message_content`, `button_text` (<= 60 chars), `next_intent_type` in `{TEXT, BUTTON, LIST, AI_ASSIST, BRANCH}`
- `whatsapp_flow.header_text` is optional
- parent-level fields: `name`; `channel_integration_id` (injected from the path when omitted); and exactly one of `previous_bot_response_id` / `previous_user_input_id`
- alternatively pass `parent` as `{type, id}` to express the parent linkage

### `update_whatsapp_flow(path_id, bot_response_id, changes, preferred_tree_version="v3")`

Use when editing an existing WhatsApp Flow bot response.

Notes:
- writes to v1 `PATCH /v1/bot_responses/whatsapp_flow/{id}` (`meta.api_version_used = "v1"`)
- read-back is verified via the v3 tree
- `changes` accepts the same nested `whatsapp_flow` fields as create, all optional

## Workflow Tools

### `workflow_create_path_with_reply(path, root_reply=None, preferred_tree_version="v3")`

Use when the task is "create a path and set the first reply".

Required create-path details:
- `path.channel_integration_id`: chatbot-local integer id
- `path.operational_time_type`: include an explicit value such as `24hours`
- `path.channels[]`: if channels are selected, send normalized items using Hub integration UUIDs as `id`
- `root_reply.name`: required; prefer a simple safe value like `Root` when validation is unclear
- `root_reply.content.is_send_message`: required for ordinary user-facing messages

Returns:
- `path_metadata`
- `tree`
- `bot_response` when a root reply was initialized

Warnings:
- this workflow is not guaranteed to be atomic; path creation can succeed even if root initialization fails later
- after a failed call, verify whether the path already exists before retrying
- successful creation still leaves the Conversation in draft state until a later `publish_path`

### `workflow_add_branch_with_reply(path_id, parent_bot_response_id, branch, preferred_tree_version="v3")`

Use when the task is "add a user option and its reply".

Expected wrapper:
- `branch.user_input`
- `branch.bot_response`

Required fields inside `branch.user_input`:
- `input`
- `channel_integration_id`
- `is_default`

Notes:
- this wraps the same logical flow as `add_branch`: create user input first, then create child bot response
- ambiguous failures can still partially succeed; do not assume the whole branch failed just because the tool returned an error
- verify ambiguous outcomes with `get_node_detail` on the parent bot response and fall back to `get_path_tree` when the parent read is incomplete or unclear
- duplicate-input errors after retry usually mean a previous call already created the branch trigger

### `workflow_change_node_component(path_id, bot_response_id, bot_response, transport="auto", preferred_tree_version="v3")`

Use when the task is phrased as "change the component or payload on this existing node".

Notes:
- delegates to `update_bot_response`
- best default entrypoint for editing interactive, attachment, API, CRM, or knowledge-source behavior

## Preferred Selection Rules

1. Use a workflow tool when it already expresses the user intent.
2. Use direct tools when the user explicitly wants a low-level step or when only one mutation is required.
3. Read the tree before child creation, deletion, or targeted node updates.
4. Read `warnings` and `meta` after every write to confirm transport and verification behavior.

## Hub Tools

Use the Hub tool family when the task depends on FE-confirmed Hub lookups or uploads rather than chatbot path mutations.

Detailed FE grounding, defaults, and quirks live in `hub-tools.md`.
Detailed Hub defaults and quirks live in `hub-tools.md`.

### `get_hub_user_profile()`

Use when the FE flow needs the current Hub user profile.

Notes:
- endpoint: `GET /core/v1/users/me`
- returns `result.profile`
- typical role: auth bootstrap

### `list_hub_users(payload=None)`

Use when assignment or agent-pick flows need Hub users.

Notes:
- endpoint: `GET /core/v1/users`
- default payload: `offset=1`, `limit=10`
- returns `result.users` and optional `result.pagination`

### `get_hub_organization_me()`

Use when the FE flow needs the current organization profile.

Notes:
- endpoint: `GET /core/v1/organizations/me`
- returns `result.organization`
- wrapper quirk: the dedicated FE organization service currently merges tag defaults

### `list_hub_integrations(payload=None)`

Use when the FE flow needs channel integrations before conversation or component setup.

Notes:
- endpoint: `GET /core/v1/integrations`
- default payload: `offset=1`, `limit=100`
- returns `result.integrations` and optional `result.pagination`
- document the current pagination-shape quirk if debugging
- these ids are Hub-side identifiers, not substitutes for chatbot `channel_integration_id`

### `normalize_hub_integrations_to_path_channels(integrations)`

Use when selected Hub integrations need to become the minimal `channels[]` payload for path create or update.

Input shape:
- `integrations`: a top-level list argument; each item needs non-blank `id`, `name` (or `label`), and `target_channel`
- example: `[ { "id": "<hub-uuid>", "name": "WhatsApp Sales", "target_channel": "whatsapp" } ]`

Output shape:
- `{ "channels": [ { "id": "<hub-uuid>", "name": "WhatsApp Sales", "target_channel": "whatsapp" }, ... ] }`

Notes:
- keeps only `id`, `name`, and `target_channel`
- falls back from `name` to `label` when needed
- output `channels` can be copied directly into `path.channels` or `changes.channels`
- this does not resolve `channel_integration_id`; use `list_chatbot_channel_integrations` for that separate integer id

### `list_hub_divisions(payload=None)`

Use when assignment or routing forms need divisions.

Notes:
- endpoint: `GET /core/v1/divisions`
- default payload: `offset=1`, `limit=100`
- returns `result.divisions` and optional `result.pagination`

### `list_hub_tags(payload=None)`

Use when tag pickers need Hub tags.

Notes:
- endpoint: `GET /core/v1/tags`
- default payload: `offset=1`, `limit=100`, `query=""`
- returns `result.tags` and optional `result.pagination`

### `upload_hub_message_file(file_name, content_bytes_base64, content_type, payload=None)`

Use when the FE flow uploads an attachment before editing a bot response.

Notes:
- endpoint: `POST /core/v1/file_uploader/message`
- returns `result.upload`
- upstream transport should be multipart

### `get_hub_qontak_integration_uniq(payload)`

Use when Qontak CRM forms need uniq-state lookup.

Notes:
- endpoint: `GET /core/v1/qontak/integration/uniq`
- `payload` is required (passed as query params); there is no default, so it must be supplied
- returns `result.integration`

### `get_hub_billing_info(payload=None)`

Use when the FE auth flow or settings flow needs billing state.

Notes:
- endpoint: `GET /core/v1/billings/info`
- returns `result.billing`
- no FE default params are confirmed

### `get_hub_client_config()`

Use when the FE auth bootstrap needs config before relying on seamless auth behavior.

Notes:
- endpoint: `GET /core/v1/client_configs/config`
- returns `result.config`
- a documented staging override may apply to this call only
