---
name: decoyduck-import-api
description: Turn an API definition into DecoyDuck test flows — an OpenAPI/Swagger spec (JSON or YAML, file or URL), a Postman collection (v2.x), curl commands, or a HAR file. Use when the user asks to import, convert, or generate DecoyDuck flows or requests from any of these.
---

# Import an API into DecoyDuck

Result: variables for the shared values, and one runnable flow per request, built through the decoyduck MCP tools. Follow the `decoyduck-flows` skill for tool order and node fields (`references/nodes.md` there).

## 1. Read the source and agree on the scope

- Read the whole file (or fetch the URL). Count the requests and group them: OpenAPI by `tags`, Postman by folder, curl/HAR as one group.
- Up to 20 requests: import all of them. More than 20: show the groups with their request counts and ask which groups to import.
- Target: the canvas the user names, otherwise the active canvas from `get_app_context`. For several groups, offer one canvas per group (`manage_canvas` `create`).

## 2. Variables first

Create these with `set_variables` before building any flow. Names use letters, digits, and underscores only.

| Source | Variable |
|---|---|
| OpenAPI `servers[0].url`, Postman `baseUrl`/`{{host}}`, the common origin of curl/HAR URLs | `baseUrl` (string, `scope: "global"`) |
| Postman collection/environment variables | same name, value from the collection |
| Path parameters such as `{id}` | one variable per name, value from the spec `example`, otherwise a placeholder like `1` |
| Bearer token / API key | `authToken` / `apiKey`, empty string — tell the user to fill it or build a login flow (step 4) |

Never copy real secrets from curl/HAR/Postman into flows; put them in variables and tell the user which ones hold secrets.

## 3. One flow per request

Call `create_flow` with `label` = `METHOD path` (e.g. `GET /users/{id}`), `tags` = the group name, and these nodes in order:

1. `restApi` labelled with the operation name (OpenAPI `operationId`/`summary`, Postman request name). Labels are used in `${Label.field}` references, so keep them to letters, digits, spaces, `_`, and `-` (no `.`, `{`, `}`, `$`) and unique on the canvas. Fields:
   - `url`: `{ "text": "${baseUrl}/users/${id}" }` — path params become `${name}`.
   - `method`: the HTTP method (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`; skip others and report them).
   - `queryParams`, `headers`: `[{ "key": {"text": "..."}, "value": {"text": "..."} }]`. Drop `Content-Type` (set by `bodyType`), `Content-Length`, `Host`, cookies, and browser-only headers from HAR.
   - `bodyType` + `body`: JSON bodies go into `body.text` as pretty-printed JSON. Use the spec `example`, else build a minimal valid object from the schema's required properties. Form bodies use the JSON-array entry format from `references/nodes.md`.
   - `auth`: `{ "type": "bearer", "token": {"text": "${authToken}"} }` or `{ "type": "basic", ... }` when the source declares that scheme. API keys go in the header or query param the spec names, with value `${apiKey}`.
2. `message` labelled `<name> OK` with `content` `{ "text": "${<restApi label>}" }` — logs the response.

The straight line puts the message on the success path. The `failure` handle stays unconnected: a failed request then stops the flow with the `restApi` node marked failed and an error log entry. The flow state is still `completed`, so judge results from node states and error logs.

### Source-specific mapping

- **Postman**: `{{var}}` → `${var}`. Dynamic variables: `{{$guid}}`/`{{$randomUUID}}` → `${$guid()}`, `{{$timestamp}}` → `${$timestampSec()}`, `{{$isoTimestamp}}` → `${$isoTimestamp()}`, `{{$randomInt}}` → `${$randomInt(0,1000)}`, `{{$randomEmail}}` → `${$randomEmail()}`, `{{$randomFullName}}` → `${$randomName()}`. Report pre-request/test scripts as not converted.
- **curl**: `-X` method, `-H` headers, `-d`/`--data-raw`/`--json` body (JSON if it parses), `-u` basic auth, `-F` multipart entries, `-k` → `insecureSkipTlsVerify: true`.
- **HAR**: only `entries` whose `request.url` is not a static asset (skip images, fonts, css, js). Use `postData.text` for the body.
- **OpenAPI**: resolve `$ref`s before reading schemas. `securitySchemes` decide `auth`.

## 4. Login chain (when the API has a login/token endpoint)

If a request returns a token that others need (e.g. `POST /auth/login`), offer to wire it:
1. In the login flow, add a `set` node after the login `restApi` node: `targetVariableName: "authToken"`, `operation: "assign"`, `value: { "text": "${Login.token}" }` (use the node's real label and response field). Variable changes are saved as soon as the node runs, so later flows read the new token.
2. On each authenticated flow, set the start node's `preFlowId` to the login flow's flowId (`update_nodes`), so the token is fetched first. Without it, the user runs the login flow once before the others.

## 5. Check and report

1. `validate_canvas` and fix errors.
2. Ask before running: requests hit real servers. If the user agrees, run one representative flow first (`run_flow` → `get_execution`), then the rest.
3. Report: flows created per group, variables the user must fill in (secrets, path params), and anything skipped (unsupported methods, scripts, file uploads without a local path).
4. `reveal` the first created flow.
