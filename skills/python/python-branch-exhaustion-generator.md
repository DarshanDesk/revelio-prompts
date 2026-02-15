# 🛠️ Skill: Python-Branch-Exhaustion-Generator

<metadata>
  id: PQS-SKILL-003
  goal: Mathematically reach 90% branch coverage.
  standard: High Hygiene / 90% Coverage Gate.
  frameworks: [pytest, Hypothesis]
</metadata>

## 📖 Description
Analyzes Python control flow (`if/elif/else`, `try/except`, `match/case`) and generates a test matrix that hits every logical path.

<instruction_set>
1. **Path Mapping:**
   - Map every conditional branch in the target function.
   - For Pydantic models, identify optional fields and boundary constraints.
2. **Test Generation:**
   - Use `@pytest.mark.parametrize` for standard multi-input validation.
   - For complex algorithmic logic, generate a `Hypothesis` strategy to find edge-case failures.
3. **Error Pathing:**
   - Explicitly write tests using `with pytest.raises(...)` for every handled exception in the source code.
</instruction_set>

<output_rules>
- Target 90% branch coverage as the absolute minimum.
- Ensure all Pydantic `ValidationError` states are tested for inbound request schemas.
</output_rules>

<example_grounding>
```python
import pytest
from app.services import WeatherService

def test_fetch_weather_isolation(mocker):
    # 🛡️ Hygiene: Patch where it is IMPORTED, not defined
    # We use mocker.Mock() to define behavior and AsyncMock for coroutines
    mock_api = mocker.patch("app.services.ExternalAPIClient.get_data", new_callable=mocker.AsyncMock)
    mock_api.return_value = {"temp": 22, "condition": "Sunny"}

    service = WeatherService()
    result = await service.get_forecast("London")

    # Then: Assert behavior without hitting real network
    assert result == "It is 22 degrees and Sunny"
    mock_api.assert_called_once_with("London")
```
</example_grounding>