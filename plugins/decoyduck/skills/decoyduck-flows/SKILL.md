---
name: decoyduck-flows
description: Build, edit, run, and debug DecoyDuck flows through the decoyduck MCP server. Use when the user asks to create or change a DecoyDuck flow, canvas, node, or variable, to run a flow and check its result, or to find out why a flow failed.
---

# DecoyDuck flows

DecoyDuck is a desktop app for building API/WebSocket test scenarios as node graphs. A **project** holds **canvases**; a canvas holds **flows**. A flow starts at a `start` node (identified by its `flowId`) and follows edges until it reaches an `end` node.

## Prerequisites

- The DecoyDuck desktop app is running and its MCP server is on (toolbar **mcp** button → toggle). The browser build has no MCP server.
- The server listens on `http://127.0.0.1:7275/mcp`. If the user changed the port in the app, set the `DECOYDUCK_MCP_PORT` environment variable to the same port and restart Claude Code.
- On WSL2, the client reaches the Windows-hosted server only with `networkingMode=mirrored` in `%USERPROFILE%\.wslconfig` (then `wsl --shutdown`).
- If the `decoyduck` tools are missing or time out, report which of the above is the likely cause instead of retrying.

## Workflow

1. **Orient** — call `get_app_context`. Use its `active` project/canvas when the user says "here" or "this canvas".
2. **Read the schema** — call `describe_schema` before creating nodes or writing `${...}` templates. It returns each node type's default field shape, the variable syntax, and the builtin functions for the running app version. Read `references/nodes.md` for what each node does, its fields, allowed values, and branching rules.
3. **Read before editing** — `get_canvas` (default `graph`) for structure, then `get_nodes` only for the nodes you need in full.
4. **Build**
   - New flow: prefer one `create_flow` call with the nodes in order. Pass `edges` only for branching.
   - Add a step between two nodes: `insert_node`.
   - Otherwise: `create_nodes` + `connect_nodes`, `update_nodes`.
   - Refer to nodes by label in edges and later templates; give every node a unique, descriptive label (e.g. `Login`, `GetProfile`).
   - Create the variables the flow reads with `set_variables` before running it.
   - Before large or risky edits, back up with `manage_project` `duplicate`.
5. **Validate** — `validate_canvas` and fix every error it reports.
6. **Run** — `clear_executions` for the flows you are about to run, then `run_flow`. It returns immediately; poll `get_execution` with the same flowIds until the state is `completed`, `failed`, or `cancelled`. Use `logLevels: ["error","warn"]` to find failures quickly.
7. **Show** — call `reveal` on the nodes you built or fixed so the user sees them.

## Templates

- Fields of type `{ "text": "..." }` accept templates: `${variableName}`, nested paths `${res.data.items[0].id}`, node results `${Node Label.property}`, and builtin functions such as `${$guid()}`, `${$randomInt(1,100)}`.
- A node's result is stored under its label and is visible only inside the same flow. To keep a value across flows, set the node's `outputVariable` or use a `set` node.
- Canvas variables shadow project-global variables with the same name.
- Escape a literal with `\${notAVariable}`.

## Destructive actions

`delete_project`, `delete_canvas`, `delete_nodes`, `disconnect_nodes`, and `delete_variables` cannot be undone from the MCP side. Confirm with the user before calling them unless they asked for that exact deletion.
