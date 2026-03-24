# Technology Stack

**Analysis Date:** 2026-03-24

## Languages

**Primary:**
- Python 3.8+ - Backend server, ML model training, data processing
- JavaScript (ES6+) - Frontend interactivity and UI logic
- HTML5 - Frontend markup
- CSS3 - Frontend styling

**Where Used:**
- Backend: `backend/app.py`, `backend/model.py`, `backend/explainer.py`, `prepare_kaggle_data.py`
- Frontend: `frontend/script.js`, `frontend/index.html`, `frontend/style.css`

## Runtime

**Environment:**
- Python 3.8+ (backend execution)
- Browser runtime (frontend - modern browsers with ES6 support)

**Package Manager:**
- pip (Python package management)
- No lockfile detected - `requirements.txt` present
- Frontend uses CDN resources (no npm/yarn)

## Frameworks

**Core:**
- Flask 3.0.0 - REST API framework for backend server
- PyTorch 2.1.2 - Deep learning framework for LSTM and Transformer models
- Pandas 2.2.0 - Data manipulation and CSV processing
- NumPy 1.26.3 - Numerical computing for array operations

**Frontend:**
- Chart.js (CDN) - Data visualization for charts
- Google Fonts API - Typography (Syne, DM Sans)

**Testing:**
- Not detected

**Build/Dev:**
- waitress - WSGI server for production deployment (fallback to Flask dev server if unavailable)

## Key Dependencies

**Critical:**
- torch 2.1.2 - Core ML framework; essential for model inference and training
- flask 3.0.0 - API server framework; required for all backend endpoints
- pandas 2.2.0 - Data loading/validation; critical for CSV parsing and transformation
- scikit-learn 1.4.0 - MinMaxScaler for feature normalization in `backend/model.py`
- numpy 1.26.3 - Array operations for tensor manipulation and numerical computations

**Infrastructure:**
- flask-cors 4.0.0 - CORS middleware; enables cross-origin frontend requests from different ports
- waitress - Production WSGI server for deployment (fallback to Flask debug server)

## Configuration

**Environment:**
- Python virtual environment recommended (`python -m venv venv`)
- No `.env` file configuration detected - all settings hardcoded or passed via CLI
- Default API port: 5000 (configurable via `app.run(port=5000)` in `backend/app.py`)
- Default Flask API URL: `http://localhost:5000` (hardcoded in `frontend/script.js` line 2)

**Build:**
- No build configuration files detected
- No package.json, Dockerfile, or similar
- Manual Python environment setup required

## Platform Requirements

**Development:**
- Python 3.8+ with pip
- Modern browser (Chrome, Firefox, Safari, Edge) with ES6 JavaScript support
- Text editor or IDE
- No special OS requirements (Windows, macOS, Linux compatible)

**Production:**
- Python 3.8+ runtime environment
- Web server: Waitress WSGI server (via `waitress.serve()` in `backend/app.py:300`) or Flask built-in server
- Static file serving: Python HTTP server (`python -m http.server 8080` in `frontend/`)
- Disk space for: trained model files (`backend/saved_models/traffic_model.pth`, `*_scaler.pkl`), CSV data files (`backend/data/traffic.csv`)

## File Locations

**Backend Entry Point:**
- `backend/app.py` - Flask application with REST API routes

**Model Files:**
- `backend/model.py` - TrafficLSTM class: LSTM + Transformer hybrid architecture
- `backend/explainer.py` - Explanation generation: feature importance, counterfactuals, temporal attention
- `backend/saved_models/` - Trained model storage (`.pth` files and `_scaler.pkl`)

**Frontend Entry Point:**
- `frontend/index.html` - Main UI
- `frontend/script.js` - API client and UI event handlers
- `frontend/style.css` - Styling

**Data Preprocessing:**
- `prepare_kaggle_data.py` - Converts Kaggle Metro Interstate Traffic Volume CSV to project format
- `backend/data/` - Runtime data storage directory
- `data/` - Optional pre-loaded datasets (`traffic_final_clean.csv`, `sample_traffic.csv`)

## Key API Dependencies

**External APIs:**
- None detected - application is self-contained

**CDN Resources:**
- Chart.js via CDN: `https://cdn.jsdelivr.net/npm/chart.js`
- Google Fonts API: `https://fonts.googleapis.com/css2?family=Syne:...&family=DM+Sans:...`

## Data Format Support

**Input:**
- CSV format with columns: `timestamp`, `speed`, `volume`, `hour`, `day_of_week`
- Auto-detection of Kaggle Metro Interstate Traffic Volume format (columns: `date_time`, `traffic_volume`)
- Automatic conversion via Greenshields traffic flow model (`backend/app.py:56-74`)

**Output:**
- JSON responses from Flask API
- CSV export for processed data

---

*Stack analysis: 2026-03-24*
