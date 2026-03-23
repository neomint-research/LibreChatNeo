# LibreChatNeo Fork Changes and Update Guide

This document summarizes the intentional differences between this fork
(`neomint-research/LibreChatNeo`) and upstream LibreChat
(`danny-avila/LibreChat`), plus how to keep the fork updated. It is meant
for someone new to the codebase who needs context and a maintenance checklist.

For the full, raw diff between the fork and upstream, run:
`git diff --stat upstream/main..main`

## Baseline and intent

- Upstream remote: `upstream` -> `https://github.com/danny-avila/LibreChat.git`
- Fork remote: `origin` -> `https://github.com/neomint-research/LibreChatNeo.git`
- Goal of this fork: a deployable, single-command stack with local web search,
  GHCR images, and improved MCP/tooling UX.

## Change map (by area)

### Deployment, images, and CI

- **Custom GHCR images**:
  - API: `ghcr.io/neomint-research/librechatneo:latest`
  - ws-local: `ghcr.io/neomint-research/librechatneo-ws-local:latest`
  - SearxNG: `ghcr.io/neomint-research/librechatneo-searxng:latest`
  - See `docker-compose.yml` and `deploy-compose.yml` for defaults.
- **New publish workflow**: `.github/workflows/neo-publish-ghcr.yml` builds and
  pushes the images above. Most upstream CI workflows were removed to keep the
  fork lightweight.
- **Docker entrypoint**: `docker/entrypoint.sh` adds:
  - automatic `OLLAMA_BASE_URL` detection,
  - embedded MongoDB startup when `MONGO_URI` is missing,
  - directory setup + permissions for mounted volumes.
- **Compose defaults**:
  - `docker-compose.yml` is tuned for named volumes and an optional
    `mintapp-librechat` network.
  - `deploy-compose.yml` is a minimal "pull and run" stack for GHCR images.

### Local web search (no API keys)

- **New services**:
  - `searxng` container with custom config in `docker/searxng/settings.yml`
  - `ws-local` Node + Playwright service in `ws-local/`
- **Backend pipeline**:
  - `api/server/services/WebSearch/` implements search, fetch, rerank, and safety
    filtering for local browsing.
  - `api/app/clients/tools/structured/WebSearch.js` exposes the tool to models.
  - `api/server/controllers/WebSearchController.js` + `api/server/routes/web.js`
    expose API endpoints and SSE status updates.
- **Client UI**:
  - `client/src/components/Chat/Input/WebSearchStatusBar.tsx` and
    `client/src/components/Chat/Messages/Content/WebResults.tsx` display status
    and results with citations.
- **Config surface**:
  - Env vars in compose: `WS_LOCAL_BASE_URL`, `SEARXNG_BASE_URL`,
    `WEB_SEARCH_PROVIDER`, `WEB_SCRAPER`, `WEB_RERANK`, `WEB_SEARCH_MAX_URLS`,
    `WEB_SEARCH_TIMEOUT_MS`, `WEB_SEARCH_AUTO`, etc.
  - Deeper details live in `docs/local-web-search.md`.

### MCP (Model Context Protocol) improvements

- **Registry redesign**:
  - `packages/api/src/mcp/MCPServersRegistry.ts` gathers server metadata,
    tool definitions, and instructions at startup with timeouts.
- **Tool list + error UX**:
  - Server-side changes in `api/server/controllers/mcp.js`,
    `api/server/routes/mcp.js`, and `api/server/services/MCP.js` improve tool
    discovery and user-facing error messages.
- **Client UX**:
  - `client/src/components/SidePanel/MCP/MCPPanel.tsx` and
    `client/src/components/Chat/Input/MCPSelect.tsx` add tool refresh and more
    reliable selection/visibility.
- **Default server config**:
  - `librechat.example.yaml` includes a `mcpServers` entry for
    `mintapp-gateway`.

### UI/UX updates in chat and settings

- **Tools and endpoint selection**:
  - More robust model/endpoint filtering and auto-selection in
    `client/src/hooks/Endpoint/*` and chat input components.
- **Thinking / reasoning display**:
  - New `client/src/components/Artifacts/Thinking.tsx` and related content
    parts for show/hide reasoning output.
- **Speech-to-text mic**:
  - The mic button is disabled in `client/src/components/Chat/Input/AudioRecorder.tsx`
    due to routing/state issues, while the underlying STT hooks remain.

### Auth and single-user mode

- **Single-user defaults**:
  - `api/server/utils/singleUser.js` provides a stable user object when auth is
    disabled.
  - `api/server/middleware/optionalJwtAuth.js` and
    `api/server/middleware/requireJwtAuth.js` apply the single-user shortcut
    when `DISABLE_AUTH` is enabled.

### Codebase cleanup

- **Cluster and cache pruning**:
  - Legacy leader election and related tests removed from `packages/api/src/cluster`.
- **CI + i18n workflow cleanup**:
  - Most upstream workflows removed (a11y, locize sync, unused packages, etc.)
    in favor of the GHCR publish workflow.

## Key files for maintaining fork-specific behavior

- `deploy-compose.yml` — deployed stack using GHCR images
- `docker-compose.yml` — local/dev stack with web search containers
- `docker/entrypoint.sh` — auto-detection and embedded Mongo behavior
- `ws-local/` — local search/scraper service
- `docker/searxng/settings.yml` — SearxNG engine configuration
- `api/server/services/WebSearch/*` — backend local search pipeline
- `api/app/clients/tools/structured/WebSearch.js` — tool schema for models
- `client/src/components/Chat/Input/*` — tools, MCP, and web search UI
- `packages/api/src/mcp/*` — MCP registry and connection logic
- `librechat.example.yaml` — default config, including MCP server entry
- `docs/local-web-search.md` — deep dive on local web search stack

## How to keep LibreChatNeo updated

### 1) Update a deployed instance (images)

Use the deploy compose file and pull new images:

```bash
docker compose -f deploy-compose.yml pull
docker compose -f deploy-compose.yml up -d
```

Or run the helper script:

```bash
npm run update:deployed
```

### 2) Sync with upstream LibreChat (code)

```bash
git fetch upstream
git checkout main
git merge upstream/main
```

When resolving conflicts, pay extra attention to these hot spots:

- `docker-compose.yml` and `deploy-compose.yml`
- `docker/entrypoint.sh`
- `api/server/services/WebSearch/*`
- `api/app/clients/tools/structured/WebSearch.js`
- `packages/api/src/mcp/*`
- `client/src/components/Chat/*` (MCP + web search UI)
- `librechat.example.yaml`

### 3) Rebuild/publish images when fork code changes

For local builds:

```bash
docker compose build api ws-local searxng
```

For GHCR builds, trigger `.github/workflows/neo-publish-ghcr.yml`
or build/push manually with the same image names used in compose.

### 4) Validate after updates

- `docker compose ps` and `docker compose logs -f api ws-local searxng`
- Run `npm run lint` and the relevant tests for changed areas
- In the UI, verify:
  - chat works,
  - MCP tools list loads and refreshes,
  - local web search returns results with citations.

## Quick mental model for new maintainers

- This fork is a Docker-first, GHCR-published LibreChat build with local web
  search baked in.
- Most code changes land in three places: web search pipeline, MCP/Tools, and
  Docker/compose.
- Keep `docs/local-web-search.md` and this file updated whenever those areas
  change.
