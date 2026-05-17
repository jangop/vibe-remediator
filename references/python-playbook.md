# Python Remediation Playbook

Python's dynamic nature is a prototyping superpower and a maintenance nightmare. The fix is to add the static guarantees the LLM didn't.

## 1. Ruff — fast automatic cleanup

Run Ruff over the whole codebase before doing anything else. It autofixes hundreds of AI-generated anti-patterns: unused imports, dangerous mutable defaults (`def f(x=[]):` ), bare `except:`, redundant casts, missing `__init__.py`.

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
select = ["E", "F", "I", "B", "UP", "SIM", "RUF", "S", "ASYNC", "C4", "PERF"]
ignore = ["E501"]  # let formatter handle line length
```

Key rule families:
- `B` (bugbear): catches mutable defaults, broad except, etc.
- `ASYNC`: detects sync calls in async contexts (huge for FastAPI codebases)
- `S` (bandit): security issues — `eval`, hardcoded passwords, unsafe yaml.load
- `SIM`: simplifications
- `PERF`: performance footguns

## 2. Pydantic at every boundary

If the codebase has a FastAPI app, Pydantic is your fastest win.

**Before** (vibe):
```python
@app.post("/orders")
async def create_order(payload: dict):
    user_id = payload.get("user_id")
    items = payload["items"]
    # ... operates on Any
```

**After:**
```python
class OrderItem(BaseModel):
    sku: str
    quantity: int = Field(gt=0)

class CreateOrderRequest(BaseModel):
    user_id: UUID
    items: list[OrderItem] = Field(min_length=1)

class OrderResponse(BaseModel):
    id: UUID
    status: Literal["pending", "confirmed", "shipped"]
    total_cents: int

@app.post("/orders", response_model=OrderResponse)
async def create_order(payload: CreateOrderRequest) -> OrderResponse:
    ...
```

Now the inside of `create_order` operates on typed, validated data; FastAPI handles 422s automatically; the OpenAPI spec is generated for free.

For non-FastAPI code, use `BaseModel.model_validate(raw)` at boundaries.

## 3. Type checking in strict mode

Pick one of `mypy --strict` or `pyright --strict`. Pyright is faster and integrates with VS Code; mypy has broader ecosystem support.

Strict mode flags:
- Untyped function defs
- Missing return types
- `Any` in signatures
- Implicit re-exports
- Unreachable code

AI-generated code routinely omits return types and hallucinates attributes that don't exist. Strict mode catches both immediately.

**Migration tip:** introduce strict mode per-module via `[[tool.mypy.overrides]]` blocks. Start with the modules at the boundary (API routes, schema definitions) and work inward.

## 4. Async hazards (FastAPI especially)

This is one of the most damaging vibe-coded patterns because the symptom — slow response times under load — only appears in production.

**Patterns to find:**

| Bad | Why | Fix |
|---|---|---|
| `requests.get(url)` in `async def` | Blocks the event loop for the whole request duration | Use `httpx.AsyncClient` |
| Sync SQLAlchemy in `async def` | Same — blocks the loop | Use `AsyncSession` (or wrap in `run_in_threadpool`) |
| `time.sleep(n)` | Blocks the loop | `await asyncio.sleep(n)` |
| `open(path).read()` for large files | Blocks I/O | `aiofiles` or `run_in_threadpool` |
| CPU-bound work (image processing, parsing) | Blocks the loop | `run_in_threadpool` or `asyncio.to_thread` |

**Detection:**

```bash
ruff check --select ASYNC .
```

Or grep:

```bash
grep -rn "requests\." --include="*.py" .
grep -rn "time\.sleep" --include="*.py" .
```

## 5. Database layer (DAL / Repository)

AI-generated code routinely puts SQL inside HTTP handlers, inside loops, inside template renderers. The fix is to force every query through a single layer.

**Repository pattern:**

```python
# repositories/orders.py
class OrderRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_user(self, user_id: UUID) -> list[Order]:
        # All queries for the Order entity live here.
        # N+1 is now visible — and fixable in one place.
        stmt = (
            select(Order)
            .where(Order.user_id == user_id)
            .options(selectinload(Order.items))  # eager load to kill N+1
        )
        return list((await self.session.scalars(stmt)).all())
```

Once routes only call `repo.get_by_user(user_id)`, N+1 issues stop being scattered across the codebase.

## 6. Silent catch-alls

Search for and rewrite:

```bash
grep -rn "except Exception" --include="*.py" .
grep -rn "except:" --include="*.py" .
```

Three replacement strategies, pick per case:

1. **Narrow the exception type** — catch what you actually expect (`httpx.TimeoutException`, `ValidationError`).
2. **Re-raise as a domain error** — `except DBError as e: raise OrderNotFoundError() from e`.
3. **Log and re-raise** — at the top-level handler only, with structured logging.

Almost never: catch-and-swallow.

## 7. Testing the safety net

Use `pytest` + `httpx.AsyncClient` for FastAPI:

```python
@pytest.fixture
async def client(app):
    async with httpx.AsyncClient(app=app, base_url="http://test") as c:
        yield c

async def test_create_order_happy_path(client):
    r = await client.post("/orders", json={"user_id": "...", "items": [...]})
    assert r.status_code == 200
    assert r.json()["status"] == "pending"
```

Use a real test database (sqlite in-memory or a Postgres test container) rather than mocking the DAL. Mocked tests freeze the structure you're trying to change.

## 8. Common bloated dependencies to audit

Check whether these are pulling weight or could be removed:
- `pandas` for trivial CSV reading → stdlib `csv`
- `requests` in async code → `httpx`
- `python-dateutil` for ISO parsing → `datetime.fromisoformat`
- `attrs` + `pydantic` both present → standardize on one
- Multiple HTTP libraries (requests + httpx + aiohttp) — pick one
