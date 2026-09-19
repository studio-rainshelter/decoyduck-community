# Variables, templates, and flows

## Variables

| Property | Meaning |
|---|---|
| `name` | Letters, digits, and `_` only, up to 100 characters. Referenced as `${name}`. |
| `type` | `int` (safe integer), `float` (finite number), `bool`, `string` (up to 10,000 characters), `json` (object or array, up to 100 KB). `set_variables` infers it from the value when omitted. |
| `isConst` | `true` blocks changes at run time: a `set` node or an `outputVariable` binding that targets it fails. |
| `tags` | String array for grouping in the variables panel. |
| `id` | Assigned by the app. `outputVariable` on a node takes this id, not the name. |

Scopes:
- **Canvas** variables belong to one canvas (`set_variables` with `scope: "canvas"` and `canvasId`).
- **Global** variables belong to the project and are shared by all its canvases (`scope: "global"`).
- A canvas variable hides a global variable with the same name. `list_variables` with `canvasId` reports these as `shadowedGlobalNames`.

Changes made by `set` nodes and `outputVariable` are saved as soon as the node runs, so the variables panel, MCP reads, and flows started afterwards see the new value. A running flow reads from its own copy taken at start, so flows running in parallel do not see each other's changes.

## Templates

Fields of the form `{ "text": "..." }` accept templates.

| Syntax | Result |
|---|---|
| `${name}` | Variable value. Objects and arrays become JSON text. |
| `${name.field}`, `${name[0]}`, `${name.items[0].id}` | Nested property or array element. A missing path gives an empty string. |
| `${Node Label}`, `${Node Label.field}` | Result data of a node that already ran in the same flow, stored under its label. A variable with the same name wins. |
| `${$func(args)}` | Builtin function, see below. |
| `\${text}` | Literal `${text}`, not substituted. |

- A name that matches no variable and no node result stays in the text unchanged (`${missing}`). `validate_canvas` reports these undefined references.
- Substitution runs in two passes: variables and node results first, builtin functions second. A variable value that contains `${$guid()}` is therefore not evaluated.
- Every result is a string. To send a JSON number or boolean, leave it unquoted in the body: `{ "count": ${count} }`.

## Builtin functions

All return strings.

| Function | Result |
|---|---|
| `$guid()` | UUID v4 |
| `$objectId()` | 24-character hex MongoDB-style ObjectId |
| `$nanoid(length?)` | Short random id, default 21 characters, length 1–256 |
| `$timestamp()` | Unix time in milliseconds |
| `$timestampSec()` | Unix time in seconds |
| `$isoTimestamp()` | ISO 8601 date-time |
| `$randomInt(min, max)` | Integer between min and max, inclusive |
| `$randomFloat(min, max)` | Decimal number between min and max |
| `$randomBool()` | `true` or `false` |
| `$randomString(length)` | Random letters and digits, length 1–10,000 |
| `$randomEmail()` | Fake email address |
| `$randomName()` | Fake English full name |

Example body: `{ "id": "${$guid()}", "name": "${$randomName()}", "ts": ${$timestamp()} }`

## Flows and execution

- A flow is a `start` node plus every node reachable from it. It is identified by the start node's `flowId`; its display name is the start node's label. `list_flows` lists them with tags and last state.
- Execution follows one edge at a time: `output` for single-path nodes, `success` / `failure` for branching nodes. Cycles are allowed, so loops are built from `if` and `set` nodes.
- Disabled nodes are skipped and the flow continues on their `output` / `success` path. A flow whose start node is disabled does not run; `run_flow` lists it under `skipped`.
- The flow stops when the current node has no edge for the handle it took. This ends the flow as `completed` even if that node failed; node states and error logs show the failure.
- Flow states: `waiting` (its pre-flow is running), `running`, `completed`, `failed` (runtime error such as a validation failure or exceeding the limit), `cancelled` (stopped by `stop_flow` or the user).
- Chaining: `start.preFlowId` runs another flow first; `end.postFlowId` starts another flow after this one ends without waiting for it; the `flowExecutor` node runs another flow in the middle and waits.

| Limit | Value |
|---|---|
| Node executions per flow run | 10,000 (then the flow fails) |
| Pre/post-flow chain depth | 50 |
| `flowExecutor` nesting | No depth limit. `validate_canvas` reports cycles through `preFlowId`, `postFlowId`, and `flowExecutor` targets (`flow-chain-cycle`) |

- `run_flow` with several flowIds or `all: true` starts them in parallel.
- Execution results live in memory only: they are replaced by the next run of the same flow and cleared when the app restarts. `clear_executions` removes finished ones.
- `export_canvas` returns canvases and global variables as JSON, which can be saved as a file for backup or version control. `import_canvas` loads that JSON into a project without overwriting existing global variables.
