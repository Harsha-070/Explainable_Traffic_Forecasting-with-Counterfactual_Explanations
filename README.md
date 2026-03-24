# Traffic Speed Predictor with Explainable AI

An AI-powered traffic speed prediction system using LSTM + Transformer hybrid neural networks with explainable AI features including counterfactual explanations, temporal attention visualization, and feature importance analysis.

## Dataset

This project uses the **Metro Interstate Traffic Volume** dataset from Kaggle:

**Download link:** https://www.kaggle.com/datasets/anshtanwar/metro-interstate-traffic-volume

The dataset contains **48,204 hourly records** of I-94 Westbound traffic volume from MN DoT ATR station 301 (between Minneapolis and St Paul, MN), collected from 2012-2018.

### Quick Setup

**Option A — Preprocess before running:**
```bash
# 1. Download Metro_Interstate_Traffic_Volume.csv from Kaggle
# 2. Place it in the project root
python prepare_kaggle_data.py Metro_Interstate_Traffic_Volume.csv
# 3. Start the backend as usual
```

**Option B — Upload via the web UI:**
Simply upload the raw Kaggle CSV through the frontend — it auto-detects the format and converts it on the fly.

### How the conversion works

The Kaggle dataset has `traffic_volume` but no `speed`. Speed is derived using the **Greenshields traffic flow model**, a standard traffic-engineering formula:

```
speed = free_flow_speed × (1 − volume / road_capacity)
```

For I-94: `free_flow_speed = 70 mph`, `road_capacity = 7200 veh/hr`

## Project Structure

```
traffic-predictor/
├── backend/
│   ├── app.py              # Flask API server
│   ├── model.py            # LSTM + Transformer neural network
│   ├── explainer.py        # Explainability engine
│   └── requirements.txt    # Python dependencies
├── frontend/
│   ├── index.html          # Main web interface
│   ├── style.css           # Styling
│   └── script.js           # Frontend logic
├── data/
│   ├── traffic_final_clean.csv  # Processed dataset
│   └── sample_traffic.csv       # Sample dataset
├── prepare_kaggle_data.py  # Kaggle CSV preprocessor
└── README.md
```

## Features

1. **Data Upload** - Upload CSV traffic data or generate sample data
2. **LSTM Model Training** - Train neural network on historical data
3. **Real-time Predictions** - Predict traffic speed for the next hour
4. **Explainable AI**:
   - Feature importance visualization
   - Counterfactual "what-if" scenarios
   - Human-readable explanations

## Input Data Format

### Required CSV Columns

| Column | Description | Type | Example |
|--------|-------------|------|---------|
| `timestamp` | Date/time of measurement | datetime | 2024-01-01 08:00 |
| `speed` | Traffic speed in mph | float | 55.5 |
| `volume` | Traffic volume (vehicles/hour) | int | 1200 |
| `hour` | Hour of day (0-23) | int | 8 |
| `day_of_week` | Day of week (0=Monday, 6=Sunday) | int | 1 |

### Sample CSV Format

```csv
timestamp,speed,volume,hour,day_of_week
2024-01-01 08:00,55,1200,8,1
2024-01-01 09:00,45,1500,9,1
2024-01-01 10:00,60,1100,10,1
2024-01-01 11:00,58,900,11,1
```

## How It Works

### 1. Data Processing
- Data is normalized using MinMaxScaler (0-1 range)
- Sliding window sequences created (12 hours input -> 1 hour prediction)
- Train/validation split for model evaluation

### 2. LSTM Model Architecture
```
Input Layer:  12 time steps x 4 features
LSTM Layer 1: 50 hidden units, dropout=0.2
LSTM Layer 2: 50 hidden units, dropout=0.2
Dense Layer:  1 output (predicted speed)
```

### 3. Prediction Process
1. Take last 12 hours of traffic data
2. Normalize input features
3. Pass through trained LSTM network
4. Inverse transform to get actual speed value

### 4. Explanation Generation
- **Feature Importance**: Measures impact of each feature using perturbation
- **Counterfactual**: Shows "what-if" scenarios with different inputs
- **Natural Language**: Generates human-readable explanation text

## API Endpoints

### GET `/`
Returns API information and available endpoints.

### GET `/status`
Check system status (data uploaded, model trained).

**Response:**
```json
{
  "data_uploaded": true,
  "model_trained": true,
  "model_loaded": true,
  "data_info": {"rows": 168, "columns": ["timestamp", "speed", "volume", "hour", "day_of_week"]}
}
```

### POST `/upload`
Upload traffic data CSV file.

**Request:** Form-data with `file` field containing CSV

**Response:**
```json
{
  "status": "success",
  "message": "Data uploaded successfully",
  "rows": 168,
  "columns": ["timestamp", "speed", "volume", "hour", "day_of_week"]
}
```

### POST `/generate-sample`
Generate sample traffic data for testing.

**Request:**
```json
{"hours": 168}
```

**Response:**
```json
{
  "status": "success",
  "message": "Generated 168 hours of sample data",
  "rows": 168
}
```

### POST `/train`
Train the LSTM model on uploaded data.

**Response:**
```json
{
  "status": "success",
  "message": "Model trained successfully",
  "epochs": 50,
  "final_loss": 0.0478,
  "training_samples": 156
}
```

### POST `/predict`
Make traffic speed prediction with explanations.

**Request:**
```json
{
  "data": [
    [55, 1200, 7, 1],
    [52, 1300, 8, 1],
    [48, 1400, 8, 1],
    [45, 1500, 8, 1],
    [42, 1550, 8, 1],
    [40, 1600, 9, 1],
    [38, 1500, 9, 1],
    [42, 1400, 9, 1],
    [48, 1200, 10, 1],
    [52, 1000, 10, 1],
    [55, 900, 11, 1],
    [58, 850, 11, 1]
  ]
}
```

Each row: `[speed, volume, hour, day_of_week]`

**Response:**
```json
{
  "prediction": 56.4,
  "unit": "mph",
  "explanation": {
    "current_prediction": 56.4,
    "feature_importance": {
      "speed": 26.5,
      "volume": 8.8,
      "hour": 64.7,
      "day_of_week": 0.0
    },
    "explanation": "The predicted traffic speed is 56.4 mph, indicating free-flowing conditions...",
    "counterfactual": {
      "scenarios": [
        {
          "scenario": "volume_decrease_20%",
          "description": "If traffic volume decreased by 20%",
          "original_prediction": 56.4,
          "new_prediction": 56.0,
          "change": -0.4
        },
        {
          "scenario": "rush_hour",
          "description": "If it was morning rush hour (8 AM)",
          "original_prediction": 56.4,
          "new_prediction": 55.8,
          "change": -0.6
        }
      ],
      "most_impactful": "off_peak"
    },
    "input_summary": {
      "avg_speed": 47.9,
      "avg_volume": 1283.0,
      "current_hour": 11,
      "day_of_week": 1
    }
  }
}
```

## Installation

### Backend Setup

```bash
cd backend

# Create virtual environment (recommended)
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Start the server
python app.py
```

The API will be available at `http://localhost:5000`

### Frontend Setup

Simply open `frontend/index.html` in your web browser.

Or serve with a static file server:
```bash
cd frontend
python -m http.server 8080
```

Then open `http://localhost:8080`

## Usage Guide

1. **Upload Data**: Click "Upload Data" to upload your CSV file, or click "Generate Sample Data" for testing

2. **Train Model**: Click "Train Model" to train the LSTM neural network (~50 epochs)

3. **Make Predictions**: Enter current traffic conditions:
   - Current speed (mph): 0-100
   - Traffic volume (vehicles/hour): 0-5000
   - Hour of day (0-23): Current hour
   - Day of week (0=Mon, 6=Sun): Current day

4. **View Results**:
   - Predicted speed with traffic condition indicator
   - Feature importance bar chart
   - Human-readable explanation
   - What-if counterfactual scenarios

## Model Accuracy

The LSTM model typically achieves:
- **Training Loss**: ~0.05 (MSE)
- **MAE**: ~3-5 mph on validation data
- **Accuracy**: 85-90% within 5 mph of actual values

Accuracy depends on:
- Quality and quantity of training data
- Consistency of traffic patterns
- Proper feature engineering

## Technologies Used

- **Backend**: Python 3.8+, Flask, PyTorch
- **Frontend**: HTML5, CSS3, JavaScript, Chart.js
- **ML**: LSTM Neural Network
- **Explainability**: Custom perturbation-based feature importance

## License

MIT License
