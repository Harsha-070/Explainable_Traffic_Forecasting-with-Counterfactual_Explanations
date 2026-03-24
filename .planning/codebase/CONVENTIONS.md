# Coding Conventions

**Analysis Date:** 2026-03-24

## Naming Patterns

**Files:**
- Python backend files: `snake_case` (e.g., `app.py`, `model.py`, `explainer.py`)
- JavaScript frontend files: `snake_case` (e.g., `script.js`)
- Data directories: `lowercase` (e.g., `data`, `saved_models`)

**Functions/Methods:**
- Python: `snake_case` for all functions and methods
  - Examples: `train_model()`, `predict()`, `explain_prediction()`, `calculate_feature_importance()`, `generate_counterfactuals()`
  - Private/helper functions prefixed with underscore: `_create_sequences()`, `_get_temporal_attention_safe()`
- JavaScript: `camelCase` for functions
  - Examples: `checkStatus()`, `uploadFile()`, `trainModel()`, `displayResults()`, `drawFeatureChart()`

**Variables:**
- Python: `snake_case` consistently
  - Examples: `base_prediction`, `data_scaled`, `epoch_loss`, `transformed_data`, `training_samples`
- JavaScript: `camelCase`
  - Examples: `featureChart`, `attentionChart`, `predictBtn`, `uploadArea`, `fileInput`
- Constants: `UPPER_CASE` in Python (e.g., `FREE_FLOW_SPEED`, `ROAD_CAPACITY`, `API_URL`)

**Types/Classes:**
- Python classes: `PascalCase`
  - Examples: `TrafficLSTM`, `PositionalEncoding`, `TransformerEncoderLayerWithWeights`
- Constructor parameters often match instance attribute names

## Code Style

**Formatting:**
- No automated formatter (Prettier, Black, or autopep8) configured
- Python: 4-space indentation (PEP 8 standard)
- JavaScript: 2-space indentation observed in Chart.js configuration
- Line length: Generally kept reasonable (no strict limit enforced)

**Linting:**
- No ESLint or Pylint configuration files detected
- Style enforcement is implicit through code review

## Import Organization

**Python Order (observed pattern):**
1. Standard library imports: `import os`, `import math`, `import sys`
2. Third-party imports: `import torch`, `import pandas as pd`, `import numpy as np`
3. Local imports: `from model import TrafficLSTM`, `from explainer import explain_prediction`

**JavaScript Order (observed pattern):**
1. Configuration constants: `const API_URL = '...'`
2. State variables: `let featureChart = null`, `let attentionChart = null`
3. Library setup: `Chart.defaults.color = ...`
4. Event listeners and function definitions follow

**Path Aliases:**
- No path aliases detected in this codebase
- Imports use relative paths for modules in same directory
- No import alias configuration (tsconfig.json, jsconfig.json) present

## Error Handling

**Python Pattern:**
- Try-catch blocks wrap major operations (API routes, model loading/prediction)
- Broad `except Exception as e` pattern used throughout
  - Example: `except Exception as e: return jsonify({"error": str(e)}), 500`
- HTTP errors returned as JSON with descriptive messages
- Validation errors provide context: missing columns, required fields shown in response
- Silent fallbacks in non-critical paths: `except ImportError:` for optional WSGI server
- Warnings printed to stdout (e.g., "Warning: Could not load saved model")

**JavaScript Pattern:**
- Try-catch blocks in async functions
- Broad error handling: `catch (error) { ... }`
- User-facing errors shown via `alert()` or status messages
- Network failures caught and displayed as toast notifications
- Silent fallbacks for optional features: `catch (error) { // Silently ignore - defaults are fine }`

**No custom exception classes used** — generic Exception with messages is standard throughout

## Logging

**Framework:** Console-based logging only

**Python Patterns:**
- `print()` statements for informational messages
- Used for: model status (`print("Model loaded..."`), training progress (`print(f'Epoch [{epoch+1}/{epochs}]...`)`)
- Warnings: `print(f"Warning: Could not load saved model: {e}")`

**JavaScript Patterns:**
- `console.log()` for debug/info messages
- Used for: API unavailability (`console.log('API not available yet')`)
- Limited logging in production code

## Comments

**When to Comment:**
- Module-level docstrings required for all files (module purpose)
- Class docstrings required (e.g., for `TrafficLSTM`, `PositionalEncoding`)
- Function docstrings used extensively for public APIs
- Inline comments explain complex logic or non-obvious decisions
  - Example: `# Clamp volume so speed never goes below MIN_SPEED`
  - Example: `# Greenshields: speed = Vf * (1 - volume / capacity)`

**JSDoc/TSDoc:**
- Python uses docstrings with standard format:
  ```python
  def function_name(arg1, arg2):
      """
      Short description.

      Args:
          arg1: Description
          arg2: Description

      Returns:
          Description of return value
      """
  ```
- Example in `explainer.py`:
  ```python
  def explain_prediction(model, data: np.ndarray) -> Dict[str, Any]:
      """
      Generate explanation for a traffic speed prediction

      Args:
          model: Trained TrafficLSTM model
          data: Input data array of shape (seq_length, 4)
                Features: [speed, volume, hour, day_of_week]

      Returns:
          Dictionary containing:
          - current_prediction: The predicted speed
          - feature_importance: Impact of each feature
          ...
      """
  ```

## Function Design

**Size:** Functions are generally focused and reasonably sized
- Small utility functions: 5-20 lines
- Main logic functions: 20-50 lines
- Complex functions like `train_model()`: 50+ lines with clear sections

**Parameters:**
- Python uses type hints: `def calculate_feature_importance(model, data: np.ndarray, feature_names: List[str]) -> Dict[str, float]:`
- JavaScript functions receive arguments without type annotations
- Use of `**kwargs` for flexible parameter passing (e.g., `fit(df, **kwargs)`)

**Return Values:**
- Python: Explicit return statements with type hints
  - Functions return dictionaries, numpy arrays, floats, or DataFrames
  - No implicit None returns — either returns value or raises exception
- JavaScript: Direct return of values or data structures
  - Async functions return promises
  - Chart updates return nothing (side effects only)

## Module Design

**Exports (Python):**
- Flask app module: `app = Flask(__name__)` — main export
- Model module: `TrafficLSTM` class exported (used as `from model import TrafficLSTM`)
- Explainer module: `explain_prediction()` function exported
- Utility script: `preprocess_kaggle_data()` function as entry point

**Exports (JavaScript):**
- Functions defined in global scope (no module pattern used)
- All functions callable from HTML event handlers and console

**Barrel Files:**
- Not used in this codebase
- Each module has single primary export (class or main function)

## Data Types & Structures

**Python:**
- NumPy arrays for numerical data and model inputs
- Pandas DataFrames for tabular data (training data, loaded CSVs)
- Python dictionaries for API responses and explanations
  - Example: `{"status": "success", "message": "...", "rows": count}`
- Type hints used selectively (more in explainer.py than app.py)

**JavaScript:**
- Plain objects for data structures
- Arrays for lists (e.g., historical data sequences)
- Chart.js library for visualization state management

## Standard Patterns

**Model State Management (Python):**
- Global variable pattern: `trained_model = None` in app.py
- Lazy loading on first use: check if model exists, load if needed
- Singleton-like pattern: single global instance per Flask process

**API Response Format (Flask):**
```python
{
    "status": "success" | error response,
    "message": "Human-readable message",
    ... additional data fields
}
```

**Async/Await (JavaScript):**
- All API calls use `async function` with `await fetch()`
- Error handling with try-catch-finally blocks
- State UI updates in finally block to restore button states

---

*Convention analysis: 2026-03-24*
