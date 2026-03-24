# Testing Patterns

**Analysis Date:** 2026-03-24

## Test Framework

**Current State:** No automated testing framework detected
- No test files present (no `.py` files with `test_` prefix or `_test.py` suffix)
- No test configuration files: `pytest.ini`, `pyproject.toml`, `setup.cfg`, or `tox.ini`
- No JavaScript test files or Jest/Vitest configuration
- **Testing approach: Manual/Integration only**

**Recommendation for Implementation:**
- Backend: Use `pytest` for Python unit/integration tests
- Frontend: Use `pytest` with Selenium or similar for browser automation (if E2E needed)
- No test runner commands currently available

## Test File Organization

**Location Pattern (if implemented):**
- Python: Co-located pattern recommended
  - Test files: `backend/test_app.py`, `backend/test_model.py`, `backend/test_explainer.py`
  - Alternative: `backend/tests/` directory with `test_*.py` files

**Naming Convention (if implemented):**
- Python test files: `test_*.py` prefix (pytest standard)
- Test functions: `test_*()` naming
- Test classes: `Test*` camelcase
- JavaScript: `*.spec.js` or `*.test.js` suffix

**Current Directory Structure (no tests):**
```
backend/
├── app.py
├── model.py
├── explainer.py
├── requirements.txt
└── data/
frontend/
├── script.js
└── index.html
```

## Test Structure Patterns

**Recommended Python Structure (for future implementation):**

```python
import pytest
from app import app, convert_kaggle_format
from model import TrafficLSTM
import pandas as pd
import numpy as np

class TestUploadData:
    """Test data upload and validation"""

    def setup_method(self):
        """Setup test client"""
        self.app = app
        self.client = self.app.test_client()

    def test_upload_missing_file(self):
        """Test upload with no file"""
        response = self.client.post('/upload')
        assert response.status_code == 400
        assert "No file provided" in response.json["error"]

    def test_kaggle_format_conversion(self):
        """Test Kaggle format auto-detection"""
        df = pd.DataFrame({
            'date_time': pd.date_range('2024-01-01', periods=100, freq='h'),
            'traffic_volume': [4000 + np.random.randint(-500, 500) for _ in range(100)]
        })

        converted_df, was_converted = convert_kaggle_format(df)

        assert was_converted is True
        assert 'speed' in converted_df.columns
        assert 'volume' in converted_df.columns
        assert len(converted_df) == 100
```

**Recommended JavaScript Structure (for future implementation):**

```javascript
describe('Traffic Speed Predictor UI', () => {

    beforeEach(() => {
        // Setup DOM elements
        document.body.innerHTML = `
            <input id="speed" type="number" value="58">
            <input id="volume" type="number" value="20">
            <button id="predictBtn">Predict</button>
        `;
    });

    test('predict button is disabled during prediction', async () => {
        const btn = document.getElementById('predictBtn');
        btn.disabled = false;

        // Mock fetch
        global.fetch = jest.fn(() =>
            Promise.resolve({
                ok: true,
                json: () => Promise.resolve({prediction: 45.5})
            })
        );

        await predict();

        expect(btn.disabled).toBe(false); // Re-enabled after response
    });
});
```

## Mocking

**Framework:** No mocking library currently used
- All external calls (HTTP, file I/O) are direct

**Current Mock Patterns Observed:**

**Flask Test Client Approach (if implemented):**
```python
# Use Flask's built-in test client - no external mocks needed
@app.route('/predict', methods=['POST'])
def predict():
    # In tests, call with test_client().post('/predict', json={...})
    # No need to mock HTTP - Flask handles it
```

**JavaScript Mock Pattern (if implementing):**
```javascript
// Current code makes direct fetch() calls
// For testing, would need to mock:
global.fetch = jest.fn((url) => {
    if (url.includes('/predict')) {
        return Promise.resolve({
            ok: true,
            json: () => Promise.resolve({
                prediction: 45.5,
                explanation: {...}
            })
        });
    }
});
```

**What to Mock:**
- External API calls (though in this project, backend is local)
- File I/O operations (CSV upload/save)
- Model prediction calls (in integration tests, can be real; in unit tests, mock model.predict())

**What NOT to Mock:**
- Internal Flask route logic (test with test_client)
- Data processing steps (test with real small datasets)
- Chart visualization (skip in tests or use snapshot testing)

## Fixtures and Factories

**Test Data Pattern (recommended):**

```python
# backend/fixtures.py
import pandas as pd
import numpy as np

def sample_traffic_data(hours=100):
    """Create sample traffic data for testing"""
    timestamps = pd.date_range(start='2024-01-01', periods=hours, freq='h')
    np.random.seed(42)

    data = []
    for ts in timestamps:
        hour = ts.hour
        data.append({
            'timestamp': ts.strftime('%Y-%m-%d %H:%M:%S'),
            'speed': np.random.randint(30, 70),
            'volume': np.random.randint(500, 2000),
            'hour': hour,
            'day_of_week': ts.dayofweek
        })

    return pd.DataFrame(data)

def sample_prediction_input(seq_length=12):
    """Create sample input for model prediction"""
    return np.random.randn(seq_length, 4) * 20 + [50, 1000, 12, 3]
```

**Fixture Location:**
- Python: `backend/conftest.py` (pytest standard) or `backend/fixtures.py`
- JavaScript: `frontend/__tests__/fixtures.js` or inline in test files

**Current Test Data (manual approach):**
- `/generate-sample` endpoint creates sample data for manual testing
- `prepare_kaggle_data.py` handles production data preparation

## Coverage

**Requirements:** No coverage requirements enforced
- No `.coveragerc` or coverage configuration
- No CI/CD pipeline to enforce minimums

**Recommendation:** If implementing tests, target:
- Backend models: 80%+ coverage
- API routes: 70%+ coverage
- Frontend components: 60%+ coverage (visualization hard to test)

**View Coverage (if implemented):**
```bash
pytest --cov=backend --cov-report=html
# Opens htmlcov/index.html for visual report
```

## Test Types

**Unit Tests (recommended):**
- Model methods: `TrafficLSTM.predict()`, `TrafficLSTM._create_sequences()`
- Explainer functions: `calculate_feature_importance()`, `generate_counterfactuals()`
- Data conversion: `convert_kaggle_format()`
- Scope: Single function in isolation, mocked dependencies
- Approach: Fast, deterministic tests with fixed random seeds

**Integration Tests (recommended):**
- Flask API routes: `/upload`, `/train`, `/predict`
- Full data pipeline: upload → train → predict
- Model persistence: save() and load() roundtrips
- Scope: Multiple functions together, real small datasets

**E2E Tests (optional for future):**
- Not currently used
- Would require Selenium or Playwright
- Test full user flow: upload data → train → make prediction
- Validate UI updates, chart rendering

## Common Patterns

**Async Testing (Python with async routes):**
Not applicable — Flask routes are synchronous
If async routes were added:
```python
@pytest.mark.asyncio
async def test_async_predict():
    response = await client.get('/predict')
    assert response.status_code == 200
```

**Async Testing (JavaScript):**
Current test structure for async functions:
```javascript
describe('async functions', () => {
    test('predict awaits fetch', async () => {
        global.fetch = jest.fn(() =>
            Promise.resolve({
                ok: true,
                json: () => Promise.resolve({prediction: 45})
            })
        );

        await predict();

        expect(global.fetch).toHaveBeenCalledWith(
            'http://localhost:5000/predict',
            expect.any(Object)
        );
    });
});
```

**Error Testing:**
```python
def test_predict_invalid_input_shape():
    """Test model rejects wrong input shape"""
    model = TrafficLSTM()

    # Wrong number of features
    with pytest.raises(Exception):
        model.predict(np.zeros((12, 3)))  # Should be 4 features

def test_api_handles_missing_model():
    """Test API gracefully fails when model not trained"""
    client = app.test_client()

    # First, delete saved model if it exists
    # Then make prediction request
    response = client.post('/predict', json={'data': [...] })

    assert response.status_code == 400
    assert "not trained" in response.json["error"]
```

## Data Validation in Tests

**DataFrame Validation Pattern:**
```python
def test_data_validation():
    """Verify uploaded data has required columns"""
    df = pd.DataFrame({'speed': [40, 45], 'volume': [100, 150]})
    # Missing 'hour' and 'day_of_week'

    response = client.post('/upload', data=...)
    assert response.status_code == 400
    assert 'Missing required columns' in response.json['error']
```

**Numerical Bounds Testing:**
```python
def test_speed_prediction_bounds():
    """Verify predictions are within realistic range"""
    model = TrafficLSTM()
    model.load('saved_models/traffic_model.pth')

    # Various inputs
    for _ in range(100):
        data = np.random.randn(12, 4)
        pred = model.predict(data)

        assert 10 <= pred <= 80, f"Prediction {pred} out of bounds"
```

## Test Execution Roadmap

**If implementing pytest (Python):**
```bash
# Run all tests
pytest backend/

# Run with coverage
pytest --cov=backend backend/

# Run specific test file
pytest backend/test_model.py

# Run specific test
pytest backend/test_model.py::TestTrafficLSTM::test_predict

# Watch mode (requires pytest-watch)
ptw backend/
```

**Current Manual Testing Approach:**
1. Start Flask server: `python backend/app.py`
2. Open frontend: `open frontend/index.html` in browser
3. Manual tests via UI:
   - Generate sample data → `/generate-sample`
   - Upload CSV → `/upload`
   - Train model → `/train`
   - Make prediction → `/predict`
4. Verify results visually

---

*Testing analysis: 2026-03-24*
