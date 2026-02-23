# Spec: Remove Order Execution Capability

## Goal

Remove all repo-managed paths that can place or queue live orders.

After this change, the system remains useful for:

- position sync
- contract sync and lookup
- read-only order/history visibility (optional, see scope)

## Why

- Reduce operational and financial risk from accidental or unintended execution.
- Keep this repo focused on observability, portfolio monitoring, and research workflows.

## Non-Goals

- Rebuilding execution in a safer subsystem.
- Dropping historical `orders` / `order_events` data in this phase.
- Removing IBKR connectivity required for non-execution workflows (positions/contracts).

## Current Execution Surfaces

- CLI live execution: `scripts/execute_cl_buy_or_sell_continous_market.py`
- Worker-based broker submit path: `scripts/work_order_queue.py` + `task worker:orders`
- API order mutation:
  - `POST /api/v1/orders`
  - `POST /api/v1/orders/{order_id}/cancel`
- Tradebot execution tool path in `src/services/tradebot_agent.py`:
  - `preview_order`
  - `check_pretrade_job`
  - `submit_order`
- Pre-trade margin checks used for execution:
  - `src/services/pretrade_checks.py`
  - `pretrade.check` handler in `scripts/work_jobs.py`

## Target State

- No code path calls broker order submit APIs (`ib.placeOrder`).
- No first-party UI/API/chat path can create, queue, submit, or cancel orders.
- Order history remains queryable (read-only) for audit/debug.
- Jobs worker continues for non-execution jobs.

## Scope Decisions

1. Keep read-only order visibility.

- Keep:
  - `GET /api/v1/orders`
  - `GET /api/v1/orders/{order_id}`
  - `GET /api/v1/orders/{order_id}/events`
- Remove or hard-disable all order mutation endpoints.

2. Keep schema tables for now.

- Keep `orders` and `order_events` tables to preserve history.
- Do not add destructive migrations in this phase.

3. Remove execution workers and tools completely.

- Delete execution worker/runtime surfaces instead of only hiding them in UI.

## Implementation Plan

### Phase 1: Immediate Execution Kill

1. Remove direct execution CLI.

- Delete `scripts/execute_cl_buy_or_sell_continous_market.py`.
- Remove references in docs and README.

2. Disable worker-based execution.

- Remove `worker:orders` task from `Taskfile.yaml`.
- Remove `scripts/work_order_queue.py` entrypoint (delete file).

3. Remove order mutation API.

- In `src/api/routers/orders.py`, remove:
  - `create_order`
  - `cancel_order`
- Keep read-only list/detail/events endpoints.

4. Remove tradebot execution tools.

- In `src/services/tradebot_agent.py`:
  - remove `preview_order`, `check_pretrade_job`, `submit_order` tool specs
  - remove tool handlers and mappings
  - update `_SYSTEM_PROMPT` to disallow execution language and behavior

### Phase 2: Execution Dependency Cleanup

1. Remove pretrade job path.

- Remove `JOB_TYPE_PRETRADE_CHECK` from `src/services/jobs.py`.
- Remove `handle_pretrade_check` and routing from `scripts/work_jobs.py`.
- Remove `src/services/pretrade_checks.py` if no longer referenced.

2. Remove now-unused execution helpers/imports.

- Clean up `src/services/order_queue.py` usage if only used by removed write paths.
- Remove stale imports and constants in API/services.

3. Keep read models stable.

- Keep `src/models.py` order entities until a later archival/drop decision.

### Phase 3: UI + Docs Hardening

1. Frontend cleanup.

- Remove cancel actions from:
  - `frontend/src/components/OrdersTable.tsx`
  - `frontend/src/components/OrdersSideTable.tsx`
- Update `TradebotChat` copy to remove “queue orders” language.
- Update worker status lights if `orders` worker is removed from expected list.

2. Documentation updates.

- Update:
  - `README.md` goals/workflow/disclaimer wording that implies live execution
  - `docs/tradebot-chatbot.md`
  - `docs/tradebot-workers.md`
  - delete or archive `docs/execute-future-cl-order-script.md`

## Acceptance Criteria

1. No executable order submit path remains.

- `rg "ib\\.placeOrder"` returns no repo-owned execution path.

2. API cannot mutate orders.

- No `POST /api/v1/orders` route.
- No `POST /api/v1/orders/{order_id}/cancel` route.

3. Tradebot cannot request order placement.

- No `submit_order`, `preview_order`, or `check_pretrade_job` in tool specs/map.
- System prompt has no instruction to queue/submit orders.

4. Operator workflow has no order worker.

- `Taskfile.yaml` contains no `worker:orders`.
- `scripts/work_order_queue.py` is removed.

5. Static checks pass.

- `uv run --with ruff ruff check`
- `uv run --with pyright pyright`

## Rollout / Risk

- Rollout order:
  1. remove execution runtime paths first (Phase 1),
  2. then cleanup dependencies (Phase 2),
  3. then UI/docs polish (Phase 3).
- Primary risk: hidden internal references to removed tools/job types.
- Mitigation: fail-fast on startup/import errors via Ruff/Pyright and endpoint smoke checks.

## Open Questions

1. Should `orders` read APIs remain long-term, or be moved to an archived/audit-only surface?
2. Should we keep `Order`/`OrderEvent` schema indefinitely or schedule a later archive + drop migration?
3. Do we want a feature flag (`ENABLE_ORDER_EXECUTION=false`) as a temporary safety backstop during transition, or proceed with full hard-removal only?
