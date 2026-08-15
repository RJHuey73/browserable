# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this repository.

## What This Is

**Browserable** (`browserable/browserable`) is an open-source, self-hostable
browser automation platform for AI agents: given a natural-language task, it
drives a real browser (local or remote) to navigate sites, fill forms, click
buttons, and extract information. It's not a single npm package — it's a set
of independently-versioned services meant to run together via Docker Compose,
plus a CLI that bootstraps that Compose stack and a JS SDK for talking to the
running API.

## Layout

This is **not** an npm/yarn workspace — there is no root `package.json`. Each
directory below is its own independently installed Node project.

| Path | Purpose |
|------|---------|
| `tasks/` | Core backend: Express API + background job processing (Bull/Redis queues). This is where the agents live. |
| `tasks/agents/` | Agent implementations — `base.js` (shared action contract), `jarvis.js` (orchestrator), `browserable.js` (the browser-driving agent), `deepresearch.js`, `generative.js`. See Architecture below. |
| `tasks/logic/` | Business logic called by routes/agents (flows, data tables, users, accounts, vectors, integrations, OTP, logs). |
| `tasks/routes/` | Express route handlers (`jarvis`, `flow`, `user`, `account`, `api`, `otp`, `integrations/`). |
| `tasks/services/` | Infra clients: Mongo, Postgres (`db.js`), Redis queues, S3, LLM calls, email, alerts. |
| `tasks/prompts/` | System/agent prompts, one subfolder per agent under `prompts/agents/`. |
| `browser/` | Standalone local browser service — a small Express + Playwright API server (`api-server.js`, `browser-manager.js`) that `tasks/` talks to for browser sessions. Runs on port 9221. |
| `ui/` | Admin/dashboard frontend. **Note:** this package started life as the `gilbarbara/react-redux-saga-boilerplate` template (see `ui/README.md`) — React + Redux-Saga + styled-components + webpack, not a modern Vite/Next app; keep that in mind before assuming conventions from newer stacks. |
| `cli/` | The `browserable` npm CLI (`npx browserable`) — clones the repo, checks Docker/Docker Compose/Node, brings up the dev Compose stack, and starts the local browser service. See `cli/index.js`. |
| `sdk/browserable-js/` | Published `browserable-js` npm package — TypeScript SDK (axios-based) for the REST API. |
| `sdk/examples/` | Example usage of the JS SDK. |
| `deployment/` | `docker-compose.dev.yml` (the dev stack: tasks, ui, docs, redis, mongodb, mongo-express, minio, pgadmin, Postgres/Supabase db), `browserable.sql`, `supabase-docker/`. |
| `docs/` | Mintlify docs source (`docs.json` nav, `.mdx`/`.md` pages) — served locally at `:2002` by the `docs` Compose service. `docs/development/` mirrors much of this file for end users. |

## Commands

There's no top-level build/test/lint command — run these per-package.

### Bring up the full stack (recommended entry point)
```bash
npx browserable              # clone + docker compose up + start local browser service (same as `browserable start`)
npx browserable down         # docker compose down (containers removed, volumes kept)
```
Equivalent manual form (from repo root):
```bash
docker compose -f deployment/docker-compose.dev.yml up
cd browser && npm install && npx playwright install && npm start   # local browser service, port 9221
```

### `tasks/` (backend API + agents)
```bash
cd tasks
npm install
npm start            # node ./bin/www — normally run inside the `tasks` Docker container instead
```
No test script is defined here (`tasks/package.json` has none) and no test
files exist in this package — treat backend changes as needing manual/API
verification against the running stack, not `npm test`.

### `browser/` (local Playwright browser service)
```bash
cd browser
npm install
npx playwright install     # required once, downloads Playwright browsers
npm start                  # api-server.js, port 9221
```

### `ui/` (admin dashboard)
```bash
cd ui
npm install
npm start                  # webpack dev server (tools/start)
npm run build               # production build (tools/build)
npm run lint                 # eslint over src/config/test/tools
npm run lint:styles          # stylelint
npm run typecheck            # tsc --noEmit -p test/tsconfig.json
npm test                     # jest (test:coverage in CI, test:watch locally, via is-ci)
npm run test:e2e             # cypress, against a served build
npm run validate             # typecheck + lint + lint:styles + test:coverage + build + size-limit — closest thing to a full CI gate
```

### `cli/` (the `browserable` npm package)
```bash
cd cli
npm install
node index.js start   # exercise the CLI locally without publishing
```

### `sdk/browserable-js/` (published SDK)
```bash
cd sdk/browserable-js
npm install
npm run build   # tsc -> dist/
npm test        # jest
```

## Architecture: the agent system (`tasks/agents/`)

All agents extend `BaseAgent` (`tasks/agents/base.js`), which defines two
universal actions every agent supports:
- **`end`** — agent is done; takes `reasoning` + `output`, logs them via
  `jarvis.updateNodeUserLog`, and calls `jarvis.endNode(...)`.
- **`error`** — irrecoverable failure; logs the error and calls
  `jarvis.errorAtNode(...)`.

A concrete agent (`generative.js`, `deepresearch.js`, `browserable.js`)
follows this shape:
- `CODE` — a unique string identifier (e.g. `"DEEPRESEARCH_AGENT"`).
- `DETAILS` — `{ description, input: {parameters, required, types}, output: {...} }`.
  The `description` is written *for the orchestrating LLM* — it's read at
  runtime to decide whether/when to invoke this agent, so it's deliberately
  verbose about what the agent does, when to use it, and what it's not for.
- `getActions()` — starts from `super.getBaseActions()` (deep-cloned) and adds
  agent-specific actions, each described the same
  `{description, input, output}` way — this whole structure is effectively a
  hand-rolled function-calling schema fed to the LLM.
- `getActionFns()` — maps each action name to an async handler
  `({ jarvis, aiData, runId, nodeId, threadId, ... }) => ...`, where `jarvis`
  is the orchestrator instance passed in, giving the action access to
  `updateNodeStatus`, `updateNodeUserLog`, `updateNodeDebugLog`, `endNode`,
  `errorAtNode`, etc.

**`jarvis.js`** is the orchestrator: it imports `GenerativeAgent`,
`BrowserableAgent`, and `DeepResearchAgent` into an `agentMap` keyed by
`CODE`, drives the LLM tool-calling loop against each agent's `getActions()`
schema, dispatches to `getActionFns()` handlers, and enforces call limits
(`LLM_CALL_LIMIT_PER_THREAD` / `_PER_NODE` / `_PER_RUN` / `_PER_FLOW_PER_DAY`).
It also owns flow/run/node status updates (`tasks/logic/flow.js`), data table
access, vector search, and email/Discord alerting.

**`browserable.js`** is the largest agent (the actual browser-driving one):
it wraps Playwright page control (click, type, scroll — see
`scrollOnPage`/`typeOnPage`), injects a webview JS helper into pages, talks to
`tasks/services/browser.js` for session management (local vs. remote
provider), and screenshots/uploads to S3 via `sharp` + `uploadFileToS3`.

When adding a new agent: extend `BaseAgent`, define `CODE`/`DETAILS`, extend
`getActions()`/`getActionFns()` off the base ones (don't reimplement `end`/
`error`), register it in `jarvis.js`'s `agentMap`, and add its prompts under
`tasks/prompts/agents/<agent-name>/`.

## Conventions

- **Agent descriptions are LLM-facing prose, not code comments.** When
  editing `DETAILS.description` or action `description` fields, write for an
  LLM deciding whether to call this — be explicit and repetitive about intent,
  not concise like a docstring.
- **Backend services communicate over Redis-backed Bull queues**
  (`tasks/services/queue.js`: `baseQueue`, `agentQueue`, `integrationsQueue`,
  `flowQueue`, `browserQueue`, `vectorQueue`) — long-running agent/browser work
  is dispatched through these, not handled inline in request handlers.
  `tasks/app.js` requires `logic/integrations/base` and
  `logic/integrations/browser` purely for their queue-processor side effects.
- **Two databases, different roles**: Postgres (`tasks/services/db.js`,
  `TASKS_DATABASE_URL`) is the primary relational store (users, accounts,
  flows); MongoDB (`tasks/services/mongodb.js`) holds task/run/node logs and
  larger documents. Don't assume one is authoritative for everything.
- **The local browser service is a separate process from `tasks/`.** `tasks/`
  calls out to it (or to a remote provider like Hyperbrowser/Steel/
  Browserbase) rather than driving Playwright in-process for every case —
  check `tasks/services/browser.js` and `tasks/logic/integrations/browser.js`
  before assuming direct Playwright access.
- **`ui/` conventions predate this project.** It's still shaped by the
  `react-redux-saga-boilerplate` it was forked from (styled-components,
  redux-saga, webpack via `tools/`, Travis/CodeClimate badges that no longer
  apply). Follow existing patterns in `ui/src/` rather than introducing a
  different stack.

## Testing

- `ui/` is the only package with a real test setup: Jest + React Testing
  Library for unit tests (`npm test` / `npm run test:coverage`), Cypress for
  e2e (`npm run test:e2e`, requires a served build on `:2001`).
- `sdk/browserable-js/` has Jest unit tests (`npm test`).
- `tasks/`, `browser/`, and `cli/` have no automated tests — `tasks` and
  `browser`'s `test` scripts are placeholders (`browser`'s literally exits 1),
  and `cli` has none either. Verify changes to these by running the stack
  (`npx browserable` or the manual Compose command) and exercising the
  relevant HTTP endpoints or CLI command by hand.

## Gotchas

- **Always use `localhost`, not `127.0.0.1`**, for every service URL — some
  containers/cookies are bound specifically to `localhost` and will silently
  fail with the loopback IP (see `docs/development/troubleshooting.mdx`).
- **Ports**: UI `2001`, Docs `2002`, Tasks API `2003` (Bull board at
  `/admin/queues`), MongoDB `27017`, MongoDB Express `3300`, MinIO API `9000`
  / Console `9001`, Postgres/db-studio `8000`, local browser service `9221`.
  A stuck "initial setup" screen almost always means the Tasks API isn't
  reachable — check `docker ps` and `http://localhost:2003/health`.
- **The CLI (`cli/index.js`) is directory-sensitive**: `validateDirectory()`
  expects to be run either inside the cloned `browserable` repo or in a
  parent directory containing a `browserable/` folder; commands like `down`
  refuse to run outside that.
- **Default secrets are placeholders, not just examples** — `SECRET`,
  `S3_KEY`/`S3_SECRET`, MinIO/Mongo-Express credentials in
  `docs/development/environment-variables.md` and `deployment/.env` are
  committed dev defaults (`please_update_this_secret`, `secret1234`, etc.).
  Don't reuse them outside local development.
- **API keys are set at runtime, not just via env vars** — the primary path
  is the Admin UI (`http://localhost:2001/dash/@admin/settings`) for LLM
  provider keys (OpenAI/Claude/Gemini/Qwen/DeepSeek) and remote browser
  provider keys (Hyperbrowser/Steel/Browserbase); env vars in
  `docker-compose.dev.yml` are the fallback/self-hosting path.
- **`browser/`'s `npm start` runs Playwright browsers you must install
  first** via `npx playwright install` (from inside `browser/`) — the CLI's
  `setupBrowserService()` does this automatically, but a manual `cd browser
  && npm start` without it will fail.
