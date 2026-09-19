---
name: decoyduck-scenarios
description: Build common DecoyDuck test scenarios from ready-made patterns — login then call authenticated APIs, repeat N times (counter loop), poll until a condition or timeout, WebSocket request/response checks, and shared setup flows reused by other flows. Use when the user asks for one of these, or describes a test that needs branching, repetition, waiting, or token reuse.
---

# DecoyDuck scenario patterns

Each pattern below lists nodes and edges. Adapt labels, URLs, and fields to the user's API, and ask only for what you cannot infer (URL, credentials, response field names). Follow the `decoyduck-flows` skill for the tool workflow and `references/nodes.md` there for node fields.

## How to build a pattern

1. `set_variables` for every variable the pattern lists.
2. `create_flow` with the nodes of the **main path** in order (no `edges`). It returns every node's `nodeId` and every edge's id.
3. Rewire with ids, never labels (every flow's end node is labelled `end`):
   - `create_nodes` for nodes outside the main path.
   - `disconnect_nodes` with `edgeIds` for main-path edges the pattern does not have (e.g. the edge into `end` when the loop goes back instead).
   - `connect_nodes` with `source`/`target` nodeIds and `handleType` for the remaining edges.
4. Several branches may end on the same `end` node. A node pair can be connected only once.
5. `validate_canvas`, then run and check node results and error logs — a flow that stops on an unconnected `failure` still reports `completed`.

Edge notation: `A -success-> B`; handle omitted means `output`.

## 1. Login, then authenticated requests

Variables: `baseUrl` (string, global), `username`, `password` (string), `authToken` (string, `""`).

| Node | Type | Key fields |
|---|---|---|
| Login | restApi | `POST ${baseUrl}/auth/login`, JSON body with `${username}`/`${password}` |
| Save Token | set | `targetVariableName: authToken`, `assign`, value `${Login.token}` (real field name) |
| Get Profile | restApi | `GET ${baseUrl}/users/me`, `auth: bearer ${authToken}` |
| Show Profile | message | `${Get Profile}` |
| Login Failed | message | `${Login}` |
| Profile Failed | message | `${Get Profile}` |

Main path: Login → Save Token → Get Profile → Show Profile → end.
Extra edges: `Login -failure-> Login Failed`, `Login Failed -> end`, `Get Profile -failure-> Profile Failed`, `Profile Failed -> end`.

## 2. Shared setup flow (reuse login in many flows)

Build pattern 1 without Get Profile as its own flow `Auth Setup`. For every flow that needs the token, set its start node's `preFlowId` to Auth Setup's flowId (`update_nodes`). The setup flow runs first and its `authToken` change is visible to the flow that follows. A failed login request still ends the setup flow as `completed`, so the following flow runs anyway with the old token — check the Login node's result when requests return 401.
Use a `flowExecutor` node (`targetFlowId`) instead when the setup must run in the middle of a flow.

## 3. Repeat N times (counter loop)

Variables: `counter` (int, `0`), `maxIterations` (int).

| Node | Type | Key fields |
|---|---|---|
| Init Counter | set | `counter` `assign` `0`, `createIfMissing: true` |
| Check Limit | if | `${counter}` `lt` `${maxIterations}` |
| Call API | restApi | the request to repeat; builtins like `${$guid()}`, `${$randomEmail()}` give per-iteration data |
| Delay | sleep | optional; `sleepMode: random`, 200–1000 ms |
| Increment | set | `counter` `add` `1` |
| Loop Done | message | `Done after ${counter} iterations` |
| API Error | message | `${Call API}` |

Main path: Init Counter → Check Limit → Call API → Delay → Increment → end.
Rewire: remove `Increment -> end`; add `Increment -> Check Limit`, `Check Limit -failure-> Loop Done`, `Loop Done -> end`, `Call API -failure-> API Error`, `API Error -> end`.
A flow stops after 10,000 node executions, so keep `maxIterations × nodes per iteration` below that.

## 4. Poll until ready, with timeout

Variables: `attempts` (int, `0`), `maxAttempts` (int, e.g. `10`).

| Node | Type | Key fields |
|---|---|---|
| Init Attempts | set | `attempts` `assign` `0`, `createIfMissing: true` |
| Check Status | restApi | `GET` the status endpoint |
| Is Ready | if | `${Check Status.status}` `eq` `done` (real field and value) |
| Ready | message | `${Check Status}` |
| Wait | sleep | `fixed`, e.g. 2000 ms |
| Count Attempt | set | `attempts` `add` `1` |
| Attempts Left | if | `${attempts}` `lt` `${maxAttempts}` |
| Timed Out | message | `Not ready after ${attempts} attempts` |

Main path: Init Attempts → Check Status → Is Ready → Ready → end.
Extra edges: `Is Ready -failure-> Wait`, `Wait -> Count Attempt`, `Count Attempt -> Attempts Left`, `Attempts Left -success-> Check Status`, `Attempts Left -failure-> Timed Out`, `Timed Out -> end`, and optionally `Check Status -failure-> Wait` to retry on HTTP errors.

## 5. WebSocket request and response check

Variables: `wsUrl` (string).

| Node | Type | Key fields |
|---|---|---|
| WS Connect | websocketConnection | `url` `${wsUrl}` |
| Subscribe | websocketRequest | JSON message body, `waitForResponse: true` |
| Check Reply | if | `${Subscribe.status}` `eq` `ok` (real field and value) |
| Reply OK | message | `${Subscribe}` |
| WS Disconnect | websocketDisconnect | — |
| Unexpected Reply | message | `${Subscribe}` |
| Connect Failed | message | `${WS Connect}` |

Main path: WS Connect → Subscribe → Check Reply → Reply OK → WS Disconnect → end.
Extra edges: `Check Reply -failure-> Unexpected Reply`, `Unexpected Reply -> WS Disconnect`, `Subscribe -failure-> Unexpected Reply`, `WS Connect -failure-> Connect Failed`, `Connect Failed -> end`.
Every `websocketRequest` sends its body before waiting, and there is no receive-only node. When the test needs a server push that follows a reply, ask the user which message prompts it and chain another `websocketRequest` with that body.
