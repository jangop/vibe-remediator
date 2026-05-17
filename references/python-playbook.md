# Python Remediation Playbook

## 1. Ruff — fast automatic cleanup

Run Ruff over the whole codebase before doing anything else. It autofixes hundreds of AI-generated anti-patterns: unused imports, mutable defaults, bare `except:`, redundant casts, missing `__init__.py`.

```bash
ruff check --fix .
ruff format .
```

A good starting `pyproject.toml`:

```toml
[tool.ruff]
target-version = "py312"
line-length = 100

[tool.ruff.lint]
# B=bugbear, ASYNC=sync-in-async, S=bandit security, SIM=simplifications, PERF=perf footguns
select = ["E", "F", "I", "B", "UP", "SIM", "RUF", "S", "ASYNC", "C4", "PERF"]
ignore = ["E501"]  # let formatter handle line length
```

## 2. Pydantic at every boundary

If the codebase has a FastAPI app, Pydantic is your fastest win.

**Before:**
```python
@app.post("/orders")
async def create_order(payload: dict):
    user_id = payload.get("user_id")
    items = payload["items"]
    # ... operates on Any
```

**After:**
```python
class CreateOrderRequest(BaseModel):
    user_id: UUID
    items: list[OrderItem] = Field(min_length=1)

@app.post("/orders", response_model=OrderResponse)
async def create_order(payload: CreateOrderRequest) -> OrderResponse:
    ...
```

The handler now operates on typed, validated data; FastAPI returns 422s automatically; the OpenAPI spec is generated for free. Apply the same pattern to `response_model` on outbound payloads.

For non-FastAPI code, use `BaseModel.model_validate(raw)` at every boundary.

## 3. Type checking in strict mode

Pick one of `mypy --strict` or `pyright --strict`. Pyright is faster and integrates with VS Code; mypy has broader ecosystem support.

Strict mode flags untyped defs, missing return types, `Any` in signatures, implicit re-exports, and unreachable code — exactly the gaps AI-generated code routinely leaves.

**Migration tip:** introduce strict mode per-module via `[[tool.mypy.overrides]]` blocks. Start at the boundary (API routes, schema definitions) and work inward.

## 4. Async hazards (FastAPI especially)

One of the most damaging vibe-coded patterns because the symptom — slow response times under load — only appears in production.

| Bad | Why | Fix |
|---|---|---|
| `requests.get(url)` in `async def` | Blocks the event loop for the whole request duration | Use `httpx.AsyncClient` |
| Sync SQLAlchemy in `async def` | Same — blocks the loop | Use `AsyncSession` (or wrap in `run_in_threadpool`) |
| `time.sleep(n)` | Blocks the loop | `await asyncio.sleep(n)` |
| `open(path).read()` for large files | Blocks I/O | `aiofiles` or `run_in_threadpool` |
| CPU-bound work (image processing, parsing) | Blocks the loop | `run_in_threadpool` or `asyncio.to_thread` |

**Detection:** `ruff check --select ASYNC .` covers the common cases. If Ruff isn't configured yet, fall back to targeted greps like `grep -rnE '\brequests\.(get|post|put|delete|patch)\b' --include="*.py"`.

## 5. Database layer (DAL / Repository)

AI-generated code routinely puts SQL inside HTTP handlers, inside loops, inside template renderers. The fix is to force every query through a single layer.

```python
# repositories/orders.py
class OrderRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_user(self, user_id: UUID) -> list[Order]:
        stmt = (
            select(Order)
            .where(Order.user_id == user_id)
            .options(selectinload(Order.items))  # eager load to kill N+1
        )
        return list((await self.session.scalars(stmt)).all())
```

Once routes only call `repo.get_by_user(user_id)`, N+1 issues stop being scattered across the codebase.

## 6. Silent catch-alls

Detection:

```bash
grep -rnE 'except\s+(Exception)?:' --include="*.py" .
```

Replacement strategies are listed in `SKILL.md` Phase 5 — narrow the type, re-raise as a domain error, or log and re-raise at the top. Apply per case.

## 7. Testing the safety net

Use `pytest` + `httpx.AsyncClient` with `ASGITransport` for FastAPI:

```python
@pytest.fixture
async def client(app):
    transport = httpx.ASGITransport(app=app)
    async with httpx.AsyncClient(transport=transport, base_url="http://test") as c:
        yield c
```

Use a real test database (sqlite in-memory or a Postgres test container) rather than mocking the DAL. Mocked tests freeze the structure you're trying to change.

## 8. Auditing dependencies

Ask each dependency to earn its place. Common candidates worth questioning:

- `pandas` used only for `read_csv` — does stdlib `csv` cover it?
- `python-dateutil` used only for ISO parsing — `datetime.fromisoformat` handles it.
- `requests` in an async codebase — should be `httpx`.
- Multiple HTTP libraries (`requests` + `httpx` + `aiohttp`) or multiple modeling libraries (`attrs` + `pydantic`) — consolidate on whichever already has wider coverage.

Don't strip a dependency because it's "old-fashioned" — only because the project would be simpler without it.
