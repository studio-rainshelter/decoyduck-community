---
name: decoyduck-flows
description: Build, edit, run, and debug DecoyDuck flows through the decoyduck MCP server. Use when the user asks to create or change a DecoyDuck flow, canvas, node, or variable, to run a flow and check its result, to find out why a flow failed, or asks how DecoyDuck nodes, variables, templates, builtin functions, or flow execution work.
---

# DecoyDuck flows

DecoyDuck is a desktop app for building API/WebSocket test scenarios as node graphs. A **project** holds **canvases**; a canvas holds **flows**. A flow starts at a `start` node (identified by its `flowId`) and follows edges until it reaches an `end` node.

For questions about how DecoyDuck works, answer from the references; this needs no MCP connection.
- `references/nodes.md` — every node type: purpose, handles, fields, allowed values, branching.
- `references/variables-and-flows.md` — variable types, scopes and limits, template syntax, builtin functions, flow execution rules, states, and limits.

## Prerequisites

- The DecoyDuck desktop app is running and its MCP server is on (toolbar **mcp** button → toggle). The browser build has no MCP server.
- The server listens on `http://127.0.0.1:7275/mcp`. If the user changed the port in the app, set the `DECOYDUCK_MCP_PORT` environment variable to the same port and restart Claude Code.
- On WSL2, the client reaches the Windows-hosted server only with `networkingMode=mirrored` in `%USERPROFILE%\.wslconfig` (then `wsl --shutdown`).
- If the `decoyduck` tools are missing or time out, report which of the above is the likely cause instead of retrying.

## Workflow

1. **Orient** — call `get_app_context`. Use its `active` project/canvas when the user says "here" or "this canvas".
2. **Read the schema** — call `describe_schema` before creating nodes or writing `${...}` templates. It returns each node type's default field shape, the variable syntax, and the builtin functions for the running app version. Read `references/nodes.md` for node fields and `references/variables-and-flows.md` for templates and execution rules.
3. **Read before editing** — `get_canvas` (default `graph`) for structure, then `get_nodes` only for the nodes you need in full.
4. **Build**
   - New flow: call `create_flow` with the main (happy) path in order and no `edges`. It adds a start node labelled with the flow label and an end node labelled `end`, and wires them in a straight line (branching nodes use their `success` handle).
   - Branches and loops: after `create_flow`, add the extra nodes with `create_nodes`, then `connect_nodes` using the **nodeIds** returned by the earlier calls. Every flow's end node is labelled `end`, so label lookups become ambiguous once a canvas has two flows.
   - Add a step between two nodes: `insert_node`.
   - Otherwise: `create_nodes` + `connect_nodes`, `update_nodes`.
   - Give every node you create a unique, descriptive label (e.g. `Login`, `GetProfile`). Templates read node results by label (`${Login.token}`).
   - Create the variables the flow reads with `set_variables` before running it.
   - Before large or risky edits, back up with `manage_project` `duplicate`.
5. **Validate** — `validate_canvas` and fix every error it reports.
6. **Run** — `clear_executions` for the flows you are about to run, then `run_flow`. Stop a flow that loops longer than expected with `stop_flow`. It returns immediately; poll `get_execution` with the same flowIds until the state is `completed`, `failed`, or `cancelled`. `completed` only means the flow stopped normally: a node that failed with no outgoing `failure` edge also ends the flow as `completed`. Check node results and `logLevels: ["error","warn"]` before reporting success.
7. **Show** — call `reveal` on the nodes you built or fixed so the user sees them.

## Templates

- Fields of type `{ "text": "..." }` accept templates: `${variableName}`, nested paths `${res.data.items[0].id}`, node results `${Node Label.property}`, and builtin functions such as `${$guid()}`, `${$randomInt(1,100)}`.
- A node's result is stored under its label and is visible only inside the same flow. To keep a value across flows, set the node's `outputVariable` or use a `set` node.
- Canvas variables shadow project-global variables with the same name.
- Escape a literal with `\${notAVariable}`.

## Destructive actions

`delete_project`, `delete_canvas`, `delete_nodes`, `disconnect_nodes`, and `delete_variables` cannot be undone from the MCP side. Confirm with the user before calling them unless they asked for that exact deletion.
