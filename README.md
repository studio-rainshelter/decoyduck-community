<div align="center">

<img src="images/logo.png" alt="DecoyDuck" width="120" />

# DecoyDuck

**Draw your API test scenario as nodes, then run it with one click.**

[Try on the Web](https://decoyduck.rainshelter.net/) · [Microsoft Store](https://apps.microsoft.com/detail/9pnjgzm4c459) · [AI Plugin](#use-with-claude-code-or-codex) · [Feedback](https://github.com/studio-rainshelter/decoyduck-community/issues)

<img src="images/03.단일플로우실행.webp" alt="Running a flow in DecoyDuck" width="720" />

</div>

When doing backend development, there's often a routine like this:

Call the Sign-up API → Log in and copy the token → Paste it into the header → Create content → Read → Update → Delete

You have 6 Postman tabs open, copy the token from the response, and paste it into the header of the next request. Dozens of times a day.

DecoyDuck turns that routine into a flow you build once. Drag nodes onto a canvas, connect them, and the whole scenario — from sign-up and token issuance to resource CRUD — runs with a single button.

This repository is the DecoyDuck community space: report bugs, request features, and install the Claude Code / Codex plugin from here.

## Contents

- [Quick Start](#quick-start)
- [Features](#features)
- [Advanced Usage](#advanced-usage)
- [Use with Claude Code or Codex](#use-with-claude-code-or-codex)
  - [Skills](#skills)
- [How is it different from Postman?](#how-is-it-different-from-postman)
- [Feedback](#feedback)

## Quick Start

Open the [web version](https://decoyduck.rainshelter.net/) — no sign-up or install needed. Your first flow takes about 3 minutes.

### 1. Add nodes

Drag nodes from the **Node Library** in the sidebar onto the canvas. Start, REST API, and End are enough for a first flow.

![Add Nodes](images/01.노드추가.webp)

### 2. Connect edges

Drag from one node's handle to another's. The connections define the execution order.

![Connect Edges](images/02.엣지연결.webp)

### 3. Configure the request

Click the REST API node to open its settings. Enter the URL, method, headers, and body.

![Node Settings](images/05.노드설정.webp)

![Various Node Settings](images/노드설정들.webp)

### 4. Run and check the response

Run the flow. The response appears in the log panel right away.

![Run and Check Response](images/8.rest-api-get-테스트.webp)

## Features

### Requests and body

GET, POST, PUT, PATCH, and DELETE with JSON, form, multipart (file upload), text, XML, or binary bodies. Basic and Bearer auth presets are built in.

![POST Request and Body Settings](images/9.rest-api-post-테스트.webp)

### Variables

Save fields from an API response into variables, then reference them in any later node as `${variable_name}`. No more copying tokens by hand.

![Create Variable](images/10.변수생성.webp)

![Reference Variable](images/11.변수사용.webp)

![Auto-save API Response to Variable](images/12.변수사용-RestAPI노드-결과저장.webp)

### Built-in functions

Generate test data inline with autocomplete: `${$guid()}`, `${$timestamp()}`, `${$randomInt(1,100)}`, `${$randomEmail()}`, and more.

![Built-in Utility Functions](images/14.빌트인함수사용법.webp)

### Copy nodes and flows

Copy nodes or whole flows with `Ctrl+C` / `Ctrl+V` to create variant scenarios quickly.

![Copy Node](images/06.노드복사.webp)

![Copy Flow](images/07.플로우복사.webp)

### Multiple flows per canvas

Keep several scenarios on one canvas. Run one from the **Flows** panel in the sidebar, or all of them with **Run All** in the bottom toolbar.

![Run Single Flow](images/03.단일플로우실행.webp)

![Batch Run All Flows](images/04.모든플로우실행.webp)

### Conditional branching

The If node branches to true/false paths with operators such as `==`, `!=`, `>`, and `<` — for example, to handle a response differently depending on its status code.

![If Node Conditional Branching](images/15.if노드사용법.webp)

## Advanced Usage

### REST + WebSocket in one flow

Get a token from a REST API, then use it to open a WebSocket connection. Connect **WS Connect → WS Request** after the REST API node.

![Mixed WebSocket Flow](images/13.WebSocketNode플로우생성및테스트.webp)

### Loops with Set + If

Increase a counter with a Set node, check it with an If node, and loop back. Useful for repeated calls and retry logic.

![Set and If Loop Structure](images/16.Set과If노드를사용해서루프플로우만드는법.webp)

> [!TIP]
> Flows can also call other flows. Put shared steps such as login into one flow and reuse them from the others.

## Use with Claude Code or Codex

The desktop app includes an MCP server. With the DecoyDuck plugin, Claude Code or Codex can build, run, and debug flows in the app for you — for example, *"Import this OpenAPI spec as flows"* or *"Make a flow that logs in and calls the orders API 10 times."*

> [!IMPORTANT]
> The plugin needs the **desktop app** (Microsoft Store). The web version has no MCP server.

**1. Turn on the MCP server** — in the app, click the **mcp** button in the toolbar and switch it on. The default port is `7275`.

**2. Install the plugin**

Claude Code:

```shell
/plugin marketplace add studio-rainshelter/decoyduck-community
/plugin install decoyduck@decoyduck
```

Codex:

```shell
codex plugin marketplace add studio-rainshelter/decoyduck-community
codex plugin add decoyduck@decoyduck
```

**3. Ask in plain language** — the assistant picks the right skill from your request. See [Skills](#skills) below.

<details>
<summary><b>Using a different port or WSL2</b></summary>

The plugin connects to `http://127.0.0.1:7275/mcp`. If you changed the port in the app, register the server yourself:

```shell
# Claude Code
claude mcp add --transport http decoyduck http://127.0.0.1:<port>/mcp
# Codex
codex mcp add decoyduck --url http://127.0.0.1:<port>/mcp
```

On WSL2, the client can reach the app running on Windows only with `networkingMode=mirrored` in `%USERPROFILE%\.wslconfig`. Run `wsl --shutdown` after changing it.
</details>

### Skills

The plugin adds three skills. You don't need to call them by name: describe what you want, and the assistant loads the matching skill. To call one explicitly in Claude Code, type `/decoyduck:<skill-name>` (for example, `/decoyduck:decoyduck-import-api`).

| Skill | Use it when you want to… |
|---|---|
| [`decoyduck-flows`](#decoyduck-flows) | Build, change, run, or debug a flow — or ask how DecoyDuck works |
| [`decoyduck-import-api`](#decoyduck-import-api) | Turn an existing API definition into flows |
| [`decoyduck-scenarios`](#decoyduck-scenarios) | Build a test that needs login, repetition, polling, or WebSocket checks |

#### `decoyduck-flows`

The base skill. It connects to the app, reads the node schema of your app version, and builds flows the way the app expects. The other two skills use it for their work.

**What it does**

1. Finds the project and canvas you have open in the app, so "this canvas" works.
2. Builds the flow, then adds branches and loops.
3. Checks the canvas for errors (broken edges, missing fields, undefined variables, circular flow calls) and fixes them.
4. Runs the flow and checks each node's result and error logs — not just whether the flow finished.
5. Selects the nodes it built or fixed in the app, so you can see them.

It asks before deleting anything, and can back up the project before large edits. Questions about nodes, variables, templates, and built-in functions are answered from its reference files, so they work even when the app is closed.

**Try asking**

- *"Add a flow on this canvas that creates a user and then fetches it."*
- *"Run the Checkout flow and tell me why it fails."*
- *"What's the difference between a canvas variable and a global variable?"*

#### `decoyduck-import-api`

Converts an API definition into flows: one flow per request, and variables for shared values.

| Supported source | Notes |
|---|---|
| OpenAPI / Swagger (JSON or YAML, file or URL) | Grouped by `tags`; auth from `securitySchemes` |
| Postman collection (v2.x) | Grouped by folder; `{{var}}` and dynamic variables such as `{{$guid}}` are converted. Pre-request and test scripts are not converted. |
| curl commands | Method, headers, body, basic auth, multipart, and `-k` are converted |
| HAR file | Static assets (images, fonts, CSS, JS) are skipped |

**What it does**

1. Reads the source and groups the requests. With more than 20 requests, it shows the groups and asks which ones to import.
2. Creates variables first: `baseUrl`, path parameters such as `id`, and empty `authToken` / `apiKey`. Real tokens or passwords found in the source go into variables, never into the flows, and it tells you which variables to fill in.
3. Creates one flow per request (`GET /users/{id}`, …), with a message node that logs the response.
4. If the API has a login endpoint, it offers to run the login flow before the others and pass the token to them.
5. Checks the canvas, then **asks before running**, because the requests hit real servers.

**Try asking**

- *"Import `openapi.yaml` into a new canvas."*
- *"Convert these curl commands into DecoyDuck flows."*
- *"Make flows from my Postman collection, only the Orders folder."*

#### `decoyduck-scenarios`

Builds tests that need branching, repetition, waiting, or token reuse from ready-made patterns. It asks only for what it can't work out on its own, such as the URL, credentials, or response field names.

| Pattern | What you get |
|---|---|
| Login, then authenticated requests | Login → save token → call APIs with a Bearer token, with a failure path for each request |
| Shared setup flow | A login flow that runs automatically before every flow that needs the token |
| Repeat N times | A counter loop with an optional random delay; built-in functions give each loop fresh test data |
| Poll until ready | Call a status API until a condition is met, or stop after a maximum number of attempts |
| WebSocket check | Connect → send → check the reply → disconnect, with failure paths |

**Try asking**

- *"Log in, then call `/orders` 50 times with a random delay."*
- *"Poll `/jobs/${jobId}` every 2 seconds until status is `done`, give up after 10 tries."*
- *"Connect to the WebSocket, subscribe, and check that the reply says `ok`."*

## How is it different from Postman?

| | **DecoyDuck** | **Postman** |
|---|---|---|
| **Scenario setup** | Connect nodes visually on a canvas | Ordered requests in Collection Runner |
| **Reading the flow** | The whole flow is visible at a glance | Harder to follow as it grows |
| **Passing values** | `${variable_name}` with autocomplete | Environment variables + scripts |
| **REST + WebSocket** | Both in one flow | Separate tabs |
| **Branching / loops** | If and Set nodes | Pre/Post request scripts |
| **Getting started** | Web version, no sign-up | Account required |
| **AI assistant** | Claude Code / Codex plugin (desktop app) | Postman's built-in AI |

Postman is a great tool. But when you need to chain several APIs and test them in order, **drawing the flow** is often more intuitive than writing scripts.

## Feedback

DecoyDuck is a personal side project, so it may have rough edges — feedback is always welcome. Please [open an issue](https://github.com/studio-rainshelter/decoyduck-community/issues) for bugs and feature requests.
