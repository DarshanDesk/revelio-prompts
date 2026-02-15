# 🛠️ Skill: Python-Mock-Injector

<metadata>
  id: PQS-SKILL-002
  goal: Automate isolation using Pythonic mocking patterns.
  standard: SonarQube Maintainability & Reliability.
  frameworks: [pytest-mock, unittest.mock]
</metadata>

## 📖 Description
Identifies external calls (APIs, File Systems, DBs) and injects `mocker` patches. It prioritizes the `pytest-mock` fixture for clean, automatic teardown.

<instruction_set>
1. **Patch Targeting:**
   - Identify where the object is *imported*, not where it is *defined* (the standard Python mocking "where to patch" rule).
2. **Behavior Definition:**
   - Use `return_value` for standard results.
   - Use `side_effect` for simulating exceptions or sequential return values.
3. **Async Mocking:**
   - Ensure `AsyncMock` is used for any patched coroutine to avoid "coroutine was never awaited" errors.
</instruction_set>

<output_rules>
- Prefer the `mocker` fixture over the `@patch` decorator for better readability and scope control.
- Ensure every mock has a defined behavior; avoid "loose" mocks that return default MagicMock objects.
</output_rules>

<example_grounding>
```python
import pytest
from hypothesis import given, strategies as st
from app.logic import DiscountCalculator

# 1. Standard Branch Coverage via Parameterization
@pytest.mark.parametrize("price, discount, expected", [
    (100, 10, 90),   # Happy Path
    (100, 0, 100),   # Boundary: Zero Discount
    (100, 100, 0),   # Boundary: Free item
])
def test_calculate_discount_branches(price, discount, expected):
    calc = DiscountCalculator()
    assert calc.apply(price, discount) == expected

# 2. Stress Testing via Hypothesis (Property-Based)
@given(st.floats(min_value=0, max_value=1e6), st.floats(min_value=0, max_value=100))
def test_discount_properties(price, discount):
    # 🛡️ Hygiene: Universal truth - discounted price never exceeds original
    calc = DiscountCalculator()
    result = calc.apply(price, discount)
    assert result <= price
    assert result >= 0
```