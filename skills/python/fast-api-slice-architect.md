# 🛠️ Skill: FastAPI-Slice-Architect

<metadata>
  id: PQS-SKILL-001
  goal: Define the narrowest possible test isolation for FastAPI.
  standard: Component Isolation & Performance.
  frameworks: [FastAPI, httpx, pytest-asyncio]
</metadata>

## 📖 Description
Determines if a test should be a pure unit test (mocking all dependencies) or a slice test using `app.dependency_overrides`. It ensures that tests do not trigger actual database migrations or external network calls.

<instruction_set>
1. **Isolation Strategy:**
   - If testing a pure logic utility: Use standard `pytest` without FastAPI overhead.
   - If testing a Router/Endpoint: Use `httpx.AsyncClient` with `app`.
2. **Dependency Overriding:**
   - Identify `Depends()` injections in the target route.
   - Generate `app.dependency_overrides` stubs for DB sessions or external clients.
3. **Fixture Scoping:**
   - Define fixtures in `conftest.py` with the narrowest scope (`function` by default) to prevent state leakage between tests.
</instruction_set>

<output_rules>
- NEVER use a real database; always provide a mock or an in-memory override.
- ALWAYS use `pytest.mark.asyncio` for any endpoint utilizing `async def`.
</output_rules>

<example_grounding>
```python
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app, get_db

# 🛡️ Hygiene: Override real DB with a mock/test session
async def override_get_db():
    yield "mock_db_session"

@pytest.mark.asyncio
async def test_read_items_slice():
    # Given: Set dependency override
    app.dependency_overrides[get_db] = override_get_db
    
    # When: Using AsyncClient to simulate the event loop
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as ac:
        response = await ac.get("/items/")
    
    # Then: Verify isolation and output
    assert response.status_code == 200
    assert response.json() == {"status": "success", "data": "mock_db_session"}
    
    # 🧹 Cleanup: Reset overrides to prevent state leakage
    app.dependency_overrides = {}
```
 </example_grounding>