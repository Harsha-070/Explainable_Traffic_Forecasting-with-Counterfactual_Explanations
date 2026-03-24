# Architecture

**Analysis Date:** 2026-03-24

## Pattern Overview

**Overall:** Layered Web Application with Machine Learning Backend

**Key Characteristics:**
- Three-tier architecture: Frontend (HTML/CSS/JS) → Backend API (Flask) → ML Engine (PyTorch)
- Synchronous request-response pattern with single global model instance
- Modular explainability layer alongside prediction pipeline
- Stateful server storing trained models and processed datasets on disk

## Layers

**Presentation (Frontend):**
- Purpose: User interface for data upload, model training, and predictions
- Location: `frontend/`
- Contains: HTML markup, CSS styling, vanilla JavaScript event handlers
- Depends on: Fetch API to communicate with backend, Chart.js for visualization
- Used by: End users via web browser

**API Gateway (Flask):**
- Purpose: HTTP endpoint handler, request validation, response formatting
- Location: `backend/app.py`
- Contains: 8 route handlers (@app.route decorators), CORS setup, file I/O orchestration
- Depends on: TrafficLSTM model, explainer module, pandas for data validation
- Used by: Frontend JavaScript, external API consumers

**Data Processing:**
- Purpose: CSV parsing, format conversion, normalization for training
- Location: `backend/app.py` (routes `/upload`, `/generate-sample`) and `prepare_kaggle_data.py`
- Contains: Greenshields traffic flow model implementation, pandas dataframe transformations
- Depends on: pandas, numpy, sklearn MinMaxScaler
- Used by: API routes when receiving uploads or generating synthetic data

**Model Layer:**
- Purpose: LSTM + Transformer hybrid neural network for time-series prediction
- Location: `backend/model.py` (TrafficLSTM class)
- Contains: PyTorch nn.Module with 4 stages: LSTM → PositionalEncoding → TransformerEncoder → Dense output
- Depends on: torch, numpy, sklearn MinMaxScaler for feature scaling
- Used by: API predict endpoint and explainer module

**Explainability Layer:**
- Purpose: Generate human-readable explanations alongside predictions
- Location: `backend/explainer.py`
- Contains: Feature importance (perturbation-based), counterfactual generation, temporal attention extraction, natural language explanations
- Depends on: Trained model instance, numpy
- Used by: `/predict` endpoint to enrich response

**Storage:**
- Purpose: Persist trained models, preprocessed datasets
- Location: `backend/saved_models/` (model weights), `backend/data/` (CSV), `data/` (source datasets)
- Contains: `.pth` files (PyTorch weights + metadata), `.pkl` (MinMaxScaler state), CSV files
- Depends on: File system
- Used by: Model loading on startup, dataset persistence across requests

## Data Flow

**Training Flow:**

1. User uploads CSV via `/upload` endpoint (frontend)
2. Flask receives file → validates columns (speed, volume, hour, day_of_week)
3. Auto-detects Kaggle format if needed → converts via Greenshields model
4. Saves to `backend/data/traffic.csv`
5. User clicks "Train Model" → calls `/train` endpoint
6. TrafficLSTM.fit() loads data → creates 12-hour sliding windows → normalizes with MinMaxScaler
7. PyTorch DataLoader shuffles batches (size=512) → forward pass → MSELoss → backward → Adam optimizer step (50 epochs)
8. Model saved to `backend/saved_models/traffic_model.pth` + `traffic_model_scaler.pkl`
9. Frontend displays training loss curve

**Prediction Flow:**

1. User enters 12 hourly records (speed, volume, hour, day_of_week) via form
2. Frontend POST to `/predict` with `data: [[...], [...], ...]`
3. Flask loads trained model (from memory or disk)
4. TrafficLSTM.predict():
   - Normalizes input using stored scaler
   - Pads/trims to 12 time steps
   - Forward pass through LSTM → PositionalEncoding → 2 TransformerEncoderLayers → FC layer
   - Takes last output → inverse scales → returns single speed value (mph)
5. explain_prediction() in parallel:
   - Calculates feature importance via perturbation (4 features tested)
   - Generates 5 counterfactual scenarios (volume ±20%, speed +10mph, hour variations)
   - Extracts temporal attention weights from transformer layers
   - Assembles JSON explanation object
6. `/predict` returns JSON with prediction, explanation details, counterfactual results

**Startup Flow:**

1. Flask initializes → creates `data/` and `saved_models/` directories
2. Copies default dataset from `../data/traffic_final_clean.csv` to `backend/data/traffic.csv`
3. If `saved_models/traffic_model.pth` exists → loads model into global `trained_model` variable
4. Server ready to accept requests

**State Management:**

- **Global model state:** Single `trained_model` instance in `app.py` module scope
- **Dataset state:** CSV file persisted at `backend/data/traffic.csv`
- **Scaler state:** MinMaxScaler pickle saved alongside model weights
- **No session management:** Stateless request handlers; all state in files or module globals

## Key Abstractions

**TrafficLSTM (nn.Module):**
- Purpose: Encapsulates hybrid LSTM+Transformer architecture and feature scaling lifecycle
- Examples: `backend/model.py` lines 85-394
- Pattern: PyTorch nn.Module subclass with custom forward(), train_model(), predict(), save/load methods
- Responsibilities: Architecture definition, training loop, inference, attention weight extraction

**PositionalEncoding (nn.Module):**
- Purpose: Add sinusoidal positional information to LSTM outputs before transformer
- Examples: `backend/model.py` lines 14-34
- Pattern: Reusable PyTorch module with learnable/non-learnable parameters
- Enables: Transformer layers to recognize temporal ordering beyond LSTM sequential processing

**TransformerEncoderLayerWithWeights (nn.Module):**
- Purpose: Custom transformer encoder capturing multi-head attention weights for explainability
- Examples: `backend/model.py` lines 37-82
- Pattern: Standard transformer encoder with added `.attn_weights` buffer for inspection
- Enables: Temporal attention visualization in explanations

**explain_prediction() function:**
- Purpose: Orchestrate all explainability sub-modules into single explanation dict
- Examples: `backend/explainer.py` lines 9-66
- Pattern: Functional composition calling calculate_feature_importance(), generate_counterfactuals(), _get_temporal_attention_safe()
- Returns: Rich explanation object with 5 properties (current_prediction, feature_importance, explanation, counterfactual, temporal_attention)

## Entry Points

**API Routes:**
- Location: `backend/app.py`
- `GET /`: Returns endpoint documentation
- `GET /status`: Health check for data/model readiness
- `GET /data-stats`: Returns min/max/mean for each feature in loaded dataset
- `POST /upload`: Accepts CSV file, returns validation result
- `POST /generate-sample`: Creates synthetic traffic data, saves to working dataset
- `POST /train`: Trains model on loaded data, returns training metrics
- `POST /predict`: Makes prediction + explanation, returns rich JSON

**Web UI Entry:**
- Location: `frontend/index.html`
- Triggered by: Browser navigation to file:// or http://localhost:8080
- Initialization: DOMContentLoaded event → checkStatus() → setDefaultValues() → fetchDataStats()

**Script Entry:**
- Location: `prepare_kaggle_data.py`
- Triggered by: Command line: `python prepare_kaggle_data.py <kaggle_csv_path>`
- Purpose: One-time preprocessing of raw Kaggle Metro Interstate Traffic Volume CSV

## Error Handling

**Strategy:** Try-except blocks at endpoint level with JSON error responses; no custom exceptions

**Patterns:**
- Route handlers wrap logic in `try: ... except Exception as e: return jsonify({"error": str(e)}), 500`
- Example: `backend/app.py` lines 77-121 (/upload error cases)
- Data validation errors return 400 with helpful messages (missing columns, invalid file)
- Model load failures caught during startup with warning; graceful fallback to untrained state
- Prediction errors (shape mismatches) return 400 with expected input shape

**Explainer resilience:**
- `_get_temporal_attention_safe()` returns uniform attention if attention extraction fails (`backend/explainer.py` lines 69-105)
- Counterfactual generation continues even if individual scenarios fail
- No error propagation prevents prediction response

## Cross-Cutting Concerns

**Logging:**
- Print statements for startup (model loading) and training progress (every 10 epochs)
- Warnings on stdout if saved model load fails or attention weights unavailable
- No structured logging; output to stderr/stdout

**Validation:**
- CSV file: Required columns checked via set difference (backend/app.py line 103-109)
- Input data: Shape validation (must be 4 features) in `/predict` endpoint (line 179-182)
- DataFrame: MinMaxScaler requires numeric data; numpy converts to float32 (model.py line 188)

**Authentication:**
- None implemented; API assumes trusted network (localhost)
- CORS enabled for all origins via `CORS(app)` (backend/app.py line 14)

**Data Normalization:**
- MinMaxScaler (0-1 range) fitted on training data, persisted with model
- Applied to input before prediction, inverse-transformed to actual speed values
- All model inputs expect scaled [0, 1] ranges (backend/model.py lines 270-271)

**Temporal Processing:**
- All inputs are 12-hour sliding windows (fixed seq_length=12)
- Prediction always targets next hour (y = data[i, 0] for speed at time i+1)
- Padding with first value if fewer than 12 steps provided (model.py lines 260-263)

