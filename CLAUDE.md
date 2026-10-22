# CLAUDE.md

Factory Inventory Management System Demo with GitHub integration - Full-stack application with Vue 3 frontend, Python FastAPI backend, and in-memory mock data (no database).

> ⚠️ **This repository and any fork you create are PUBLIC.** Do not commit credentials, internal hostnames, or private registry URLs. `client/.npmrc` pins the public npm registry and `client/package-lock.json` is gitignored to prevent locally-configured registries from leaking into commits — leave both in place.

## Critical Tool Usage Rules

### Subagents
Use the Task tool with these specialized subagents for appropriate tasks:

- **vue-expert**: Use for Vue 3 frontend features, UI components, styling, and client-side functionality
  - Examples: Creating components, fixing reactivity issues, performance optimization, complex state management
  - **MANDATORY RULE: ANY time you need to create or significantly modify a .vue file, you MUST delegate to vue-expert**
- **code-reviewer**: Use after writing significant code to review quality and best practices
- **Explore**: Use for understanding codebase structure, searching for patterns, or answering questions about how components work
- **general-purpose**: Use for complex multi-step tasks or when other agents don't fit

### Skills
- **backend-api-test** skill: Use when writing or modifying tests in `tests/backend` directory with pytest and FastAPI TestClient

### MCP Tools
- **ALWAYS use GitHub MCP tools** (`mcp__github__*`) for ALL GitHub operations
  - Exception: Local branches only - use `git checkout -b` instead of `mcp__github__create_branch`
- **ALWAYS use Playwright MCP tools** (`mcp__playwright__*`) for browser testing
  - Test against: `http://localhost:3000` (frontend), `http://localhost:8001` (API)

## Stack
- **Frontend**: Vue 3 + Composition API + Vite (port 3000)
- **Backend**: Python FastAPI (port 8001)
- **Data**: JSON files in `server/data/` loaded via `server/mock_data.py`

## Quick Start

```bash
# Backend
cd server
uv run python main.py

# Frontend
cd client
npm install && npm run dev
```

`./scripts/start.sh` / `./scripts/stop.sh` start/stop both servers (also exposed as `/start` and `/stop`).

## Commands

```bash
# Backend tests (run from server/, which owns pyproject.toml and the pytest dev deps;
# running `uv run pytest` from tests/ fails with "Failed to spawn: pytest")
cd server && uv run pytest ../tests/backend -v
cd server && uv run pytest ../tests/backend/test_inventory.py::TestInventoryEndpoints::test_get_all_inventory -v   # single test

# Frontend production build (output: client/dist/)
cd client && npm run build
```

There is no frontend test runner and no linter configured for either side. `client/package-lock.json` is gitignored by design (see warning above).

## Architecture

- **Backend** (`server/main.py`, ~300 lines): single FastAPI file holding Pydantic models, shared filter helpers (`apply_filters`, `filter_by_month`), and all routes. `server/mock_data.py` loads every `server/data/*.json` into module-level lists at import time; nothing persists across restarts. `server/generate_data.py` regenerates the JSON files.
- **Frontend** (`client/src`): routes are registered in `main.js` (Dashboard, Inventory, Orders, Demand, Spending, Reports). `views/Backlog.vue` exists but has no route. All HTTP goes through `api.js`, which hard-codes `http://localhost:8001/api` and drops any filter equal to `'all'` before building the query string.
- **Shared state** lives in module-level refs inside composables, so every importer sees the same state: `useFilters` (the 4 global filters), `useI18n` (en/ja translations from `locales/`, locale persisted in `localStorage`, and currency derived from locale: en→USD, ja→JPY), `useAuth` (mock user whose display fields also switch on locale). New user-facing strings need keys in both `locales/en.js` and `locales/ja.js`.
- **Filter semantics**: the `month` filter matches against each record's `order_date` and accepts `YYYY-MM` or a quarter key; the quarter mapping (`QUARTER_MAP`) is hard-coded to 2025 in `main.py`. Inventory has no time dimension, so it ignores `month`.

## Key Patterns

**Filter System**: 4 filters (Time Period, Warehouse, Category, Order Status) apply to all data via query params
**Data Flow**: Vue filters → `client/src/api.js` → FastAPI → In-memory filtering → Pydantic validation → Computed properties
**Reactivity**: Raw data in refs (`allOrders`, `inventoryItems`), derived data in computed properties

## API Endpoints
- `GET /api/inventory`, `/api/inventory/{id}` - Filters: warehouse, category
- `GET /api/orders`, `/api/orders/{id}` - Filters: warehouse, category, status, month
- `GET /api/dashboard/summary` - All filters
- `GET /api/demand`, `/api/backlog` - No filters
- `GET /api/spending/*` - Summary, monthly, categories, transactions
- `GET /api/reports/quarterly`, `/api/reports/monthly-trends` - Used by the Reports view

## Common Issues
1. Use unique keys in v-for (not `index`) - use `sku`, `month`, etc.
2. Validate dates before `.getMonth()` calls
3. Update Pydantic models when changing JSON data structure
4. Inventory filters don't support month (no time dimension)
5. Revenue goals: $800K/month single, $9.6M YTD all months

## File Locations
- Views: `client/src/views/*.vue`; shared modals/widgets: `client/src/components/`
- Tests: `tests/backend/` (`conftest.py` provides the TestClient fixture); `tests/README.md` lists a `test_orders.py` that does not exist in the repo
- Sub-guides: `client/CLAUDE.md` and `server/CLAUDE.md` hold generic Vue/FastAPI conventions
- API Client: `client/src/api.js`
- Backend: `server/main.py`, `server/mock_data.py`
- Data: `server/data/*.json`
- Styles: `client/src/App.vue`

## Design System
- Colors: Slate/gray (#0f172a, #64748b, #e2e8f0)
- Status: green/blue/yellow/red
- Charts: Custom SVG, CSS Grid for layouts
- No emojis in UI
