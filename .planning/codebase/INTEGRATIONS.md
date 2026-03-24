# External Integrations

**Analysis Date:** 2026-03-24

## APIs & External Services

**Not applicable** - This is a self-contained application with no external API dependencies. All functionality is implemented locally.

## Data Storage

**Databases:**
- Not used - Application uses local CSV files only

**File Storage:**
- Local filesystem only
  - Data directory: `backend/data/` (runtime) and `data/` (project root for pre-loaded datasets)
  - Working data path: `backend/data/traffic.csv` (written during upload/generation)
  - Sample datasets: `data/traffic_final_clean.csv`, `data/sample_traffic.csv`

**Caching:**
- None - In-memory only during session
- Model state: Persisted to disk as PyTorch checkpoint files

## Model Persistence

**Saved Model Location:**
- `backend/saved_models/traffic_model.pth` - PyTorch model weights and configuration
- `backend/saved_models/traffic_model_scaler.pkl` - MinMaxScaler pickle file for feature normalization

**Model Loading (Auto on Startup):**
- `backend/app.py:34-40` - Attempts to load pre-trained model on Flask startup
- Falls back to untrained state if model file doesn't exist

## Authentication & Identity

**Auth Provider:**
- Not used - No authentication system implemented
- CORS enabled via `flask-cors` to allow cross-origin requests
- No API key, token, or user management

**CORS Configuration:**
- `backend/app.py:13-14` - `CORS(app)` enables requests from any origin
- Frontend at `http://localhost:8080` can access backend at `http://localhost:5000`

## Monitoring & Observability

**Error Tracking:**
- None - No external error tracking service

**Logs:**
- Console output only
  - Startup messages: `backend/app.py:295-296`
  - Training progress: `backend/model.py:233-234` (prints every 10 epochs)
  - Model loading status: `backend/app.py:38,40`
  - Warnings for missing model: `backend/app.py:40`

## CI/CD & Deployment

**Hosting:**
- Local/Manual deployment
- No cloud platform integration detected
- Backend: Run `python app.py` in `backend/` directory
- Frontend: Serve `frontend/` directory via `python -m http.server 8080` or any static file server

**CI Pipeline:**
- Not implemented

**Server Configuration:**
- WSGI server: Waitress (production) or Flask development server (fallback)
  - Port: 5000 (configurable in `backend/app.py:300`)
  - Host: 0.0.0.0 (all interfaces)
  - Debug mode: False in production, True optional in development

## Environment Configuration

**Required env vars:**
- None detected - Application is fully self-contained with hardcoded defaults

**Application Configuration (hardcoded):**
- `API_URL = 'http://localhost:5000'` - Frontend API endpoint (`frontend/script.js:2`)
- `FREE_FLOW_SPEED = 70.0` - Greenshields model constant (mph) (`backend/app.py:65`)
- `ROAD_CAPACITY = 7200.0` - Greenshields model constant (vehicles/hr) (`backend/app.py:65`)
- Data directories auto-created: `backend/data/`, `backend/saved_models/` (`backend/app.py:17-18`)

**Secrets location:**
- No secrets management - Application contains no sensitive credentials
- All configuration is public and hardcoded

## Webhooks & Callbacks

**Incoming:**
- None - Application is request-response only

**Outgoing:**
- None - Application makes no external API calls

## REST API Endpoints

**Health/Status:**
- `GET /` - Returns API information and available endpoints
- `GET /status` - Check if data is uploaded and model is trained
- `GET /data-stats` - Get statistics about loaded dataset

**Data Operations:**
- `POST /upload` - Upload CSV traffic data file (supports Kaggle format auto-conversion)
- `POST /generate-sample` - Generate synthetic traffic data for testing
  - Input: `{"hours": integer}` (default: 168 for 1 week)

**Model Operations:**
- `POST /train` - Train LSTM + Transformer model on uploaded data
  - Epochs: 50 (hardcoded in `backend/model.py:172`)
  - Learning rate: 0.001 (hardcoded)
  - Batch size: 512 (hardcoded)

**Prediction Operations:**
- `POST /predict` - Make traffic speed prediction with explanations
  - Input: `{"data": [[speed, volume, hour, day_of_week], ...]}` (12 timesteps required)
  - Output: Prediction, unit, feature importance, counterfactuals, temporal attention

## Data Flow

**Upload → Process → Train → Predict:**
1. Frontend uploads CSV via `POST /upload`
2. Backend validates and optionally converts Kaggle format
3. File saved to `backend/data/traffic.csv`
4. Frontend requests `POST /train` to train model
5. Model saved to `backend/saved_models/traffic_model.pth`
6. Frontend sends `POST /predict` with 12-hour window of data
7. Backend returns prediction with explanations

**Auto-load on Startup:**
- If `backend/saved_models/traffic_model.pth` exists, load it automatically
- If `data/traffic_final_clean.csv` exists, copy to working directory
- Application is ready to serve predictions without retraining

## Format Conversions

**Kaggle Format Detection and Auto-Conversion:**
- Detects columns: `date_time`, `traffic_volume`
- Derives: `hour`, `day_of_week` from timestamp
- Applies Greenshields model to calculate speed from volume
- Adds realistic noise: `np.random.default_rng(42).normal(0, 1.5)` (fixed seed)
- Outputs: `timestamp`, `speed`, `volume`, `hour`, `day_of_week`
- Conversion location: `backend/app.py:56-74`

---

*Integration audit: 2026-03-24*
