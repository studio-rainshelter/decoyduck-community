# Node reference

`describe_schema` gives the default field shape of each node type. This file explains what the fields mean.

Common fields on every node: `label` (unique per canvas, used for `${Label.prop}` references and edge lookups), `disabled`, `outputVariable` (the **id** of an existing variable — not its name — that receives the node's result data on success and on failure; get the id from `set_variables` or `list_variables`).

Handles: `input` receives edges. Outgoing handles are `output` (single path), or `success` / `failure` (branching). Omitting `handleType` in `connect_nodes` picks `output`, or `success` for branching nodes.

Template fields are objects `{ "text": "..." }` and accept `${...}`. `HttpHeader` is `{ "key": { "text": "" }, "value": { "text": "" } }`.

## start — flow entry point
Handles: `output`.

| Field | Meaning |
|---|---|
| `flowId` | Unique flow id. `run_flow`, `get_execution`, and other nodes refer to the flow by this id. |
| `tags` | String array for grouping. |
| `preFlowId` | flowId to run before this flow, or `null`. This flow is skipped only when the pre-flow ends `failed` or `cancelled`; a failed request inside it that just stops the pre-flow still counts as `completed`. |

A flow whose start node is disabled does not run and leaves no execution record; `run_flow` lists it under `skipped`. As a pre/post flow or a `flowExecutor` target it counts as completed, so the calling flow continues.

## end — flow exit point
Handles: `input`.

| Field | Meaning |
|---|---|
| `postFlowId` | flowId to run after this flow reaches the end node, or `null`. |

Pre/post flow chaining stops at depth 50.

## message — write a line to the execution log
Handles: `input` → `output`. Always succeeds.

| Field | Meaning |
|---|---|
| `content` | Template text to log. Useful for checking variable values and node results. |

## sleep — pause the flow
Handles: `input` → `output`. Stopping the flow releases the wait at once.

| Field | Meaning |
|---|---|
| `sleepMode` | `fixed` or `random`. |
| `duration` | Milliseconds to wait in `fixed` mode. |
| `minDuration`, `maxDuration` | Range in milliseconds for `random` mode. |

## restApi — send an HTTP request
Handles: `input` → `success` / `failure`.

| Field | Meaning |
|---|---|
| `url` | Template. |
| `method` | `GET`, `POST`, `PUT`, `PATCH`, `DELETE`. The body is sent only for `POST`, `PUT`, `PATCH`, `DELETE`. |
| `headers`, `queryParams` | `HttpHeader[]`. Entries with an empty key are skipped. The default `User-Agent` and `Accept` headers fall back to built-in values when their value is empty. |
| `bodyType` | `application/json`, `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`, `application/xml`, `application/octet-stream`. Sets `Content-Type` unless a header already does. |
| `body` | Template with the request body. For `application/x-www-form-urlencoded` and `multipart/form-data`, the text is a JSON array of entries: `[{"key":{"text":"name"},"value":{"text":"value"}}]` (multipart entries also take `"type":"text"` or `"type":"file"` with `filePath`). |
| `bodyTexts` | Per-bodyType body storage used by the app UI. If `bodyTexts[bodyType]` exists it is sent instead of `body.text`, so update or remove that entry when you change the body. |
| `auth` | `{ "type": "none" }`, `{ "type": "basic", "username": {text}, "password": {text} }`, or `{ "type": "bearer", "token": {text} }`. Overrides a manual `Authorization` header. |
| `insecureSkipTlsVerify` | `true` skips TLS certificate checks for https (desktop app only). |

Branching: status < 400 → `success`; status ≥ 400 or a network error → `failure`.
Result data: on success, the parsed JSON body (or text), so `${Login.token}` reads the `token` field of the response. On failure, `{ requestUrl, method, requestHeaders, statusCode, statusText, responseHeaders, responseBody }` or `{ ..., error }`.

## websocketConnection — open a WebSocket connection
Handles: `input` → `success` / `failure`.

| Field | Meaning |
|---|---|
| `url` | Template, `ws://` or `wss://`. |
| `headers` | Handshake headers, `HttpHeader[]` (desktop app only). |
| `queryParams` | `HttpHeader[]`, e.g. an auth token. |
| `insecureSkipTlsVerify` | `true` skips TLS certificate checks for `wss://`. |

The connection stays open for the rest of the flow. Close it with `websocketDisconnect`.

## websocketRequest — send a message on the open connection
Handles: `input` → `success` / `failure`.

| Field | Meaning |
|---|---|
| `body` | Template with the message to send. |
| `waitForResponse` | `true` waits for the next server message and stores it as the result data. |

Takes `failure` when no connection is open.

## websocketDisconnect — close the open connection
Handles: `input` → `output`. No fields.

## if — branch on a comparison
Handles: `input` → `success` (true) / `failure` (false).

| Field | Meaning |
|---|---|
| `left`, `right` | Templates compared as strings after substitution. |
| `operator` | `eq`, `neq`: numeric compare when both sides are numbers, otherwise string compare. `gt`, `lt`, `gte`, `lte`: numeric only, false if either side is not a number. `contains`: `left` includes `right`. |

## set — assign or change a variable
Handles: `input` → `output`.

| Field | Meaning |
|---|---|
| `targetVariableName` | Variable name (without `${}`). |
| `operation` | `assign` (overwrite with the string value), `add`, `subtract`, `multiply`, `divide` (both sides converted to float; dividing by zero fails). |
| `value` | Template. |
| `createIfMissing` | `true` creates the variable at run time if it does not exist. |

Fails when the target variable is `isConst`.

## flowExecutor — run another flow as a subroutine
Handles: `input` → `output`. There is no `failure` handle.

| Field | Meaning |
|---|---|
| `targetFlowId` | flowId of a flow on the same canvas. Empty means none selected. |

Waits for the sub-flow to finish. If the target is missing or the sub-flow ends `failed` / `cancelled`, this node fails and the current flow stops there. Handle failures inside the sub-flow with an `if` node. The target cannot be the flow that contains this node.
