# Codebase Concerns

**Analysis Date:** 2026-03-24

## Tech Debt

**Broad Exception Handling:**
- Issue: Blanket `except Exception as e` catches used throughout, suppressing specific error details
- Files: `backend/app.py` (lines 39, 120, 152, 197, 238, 290), `backend/explainer.py` (line 92), `frontend/script.js` (lines 41, 101, 155, 179, 224, 280)
- Impact: Makes debugging difficult, masks actual error sources, poor visibility into API failures
- Fix approach: Catch specific exceptions (ValueError, FileNotFoundError, KeyError) and provide contextual error messages; add detailed logging for unexpected errors

**Silent Error Suppression in Frontend:**
- Issue: `fetchDataStats()` in `frontend/script.js` (line 41-43) silently ignores all fetch errors with `catch (error) { // Silently ignore - defaults are fine }`
- Files: `frontend/script.js` (lines 34-43)
- Impact: Failed API calls go unlogged, users never know data stats couldn't load, hard to diagnose network issues
- Fix approach: Log errors to console, implement retry logic or user notification for critical operations

**Bare `weights_only=False` in Model Loading:**
- Issue: `torch.load()` in `backend/model.py` (line 374) uses `weights_only=False` which bypasses PyTorch security warnings
- Files: `backend/model.py` (line 374)
- Impact: Potential for arbitrary code execution if model files are compromised; deprecated behavior
- Fix approach: Validate model checksums, implement safer loading with weights validation, add warning comments explaining the risk

## Known Bugs

**Scaler Mismatch on Cold Start:**
- Symptom: Prediction returns incorrect values or crashes if model is loaded before scaler is fitted
- Files: `backend/model.py` (lines 269-271, 309-310), `backend/app.py` (lines 164-167)
- Trigger: Load pre-trained model, then immediately call `/predict` without running `/train` first on data
- Current behavior: `scaler.transform()` called on unfitted scaler, returns NaN or error
- Workaround: Always call `/train` after loading, or ensure scaler is loaded alongside model

**Temporal Attention Fallback Mismatch:**
- Symptom: Temporal attention displays "not available" even though model theoretically has attention weights
- Files: `backend/explainer.py` (lines 69-105), `frontend/script.js` (lines 335-338)
- Trigger: If `model.get_attention_weights()` fails silently, fallback to uniform weights (lines 96-104)
- Impact: Users see uniform attention when actual attention pattern exists but extraction failed
- Workaround: Check model version, ensure transformer layers exist

**Progress Bar Mismatch with Actual Training:**
- Symptom: Frontend progress bar reaches 100% before training actually completes
- Files: `frontend/script.js` (lines 195-202, 209-211)
- Trigger: Progress interval clears at fixed 90% (line 199) then jumps to 100%, but actual training may still be running
- Impact: User sees "complete" while backend is still processing, confusing UX
- Fix approach: Use server-sent events or polling for actual training progress instead of client-side simulation

## Security Considerations

**CORS Enabled Without Origin Validation:**
- Risk: `CORS(app)` in `backend/app.py` (line 14) allows requests from ANY origin
- Files: `backend/app.py` (line 14)
- Current mitigation: None—all origins accepted
- Recommendations: Restrict to specific frontend domain(s), use `CORS(app, origins=['http://localhost:8080', 'https://yourdomain.com'])`

**Hardcoded API Endpoint in Frontend:**
- Risk: API_URL hardcoded to `http://localhost:5000` in `frontend/script.js` (line 2)
- Files: `frontend/script.js` (line 2)
- Impact: Requires manual code change for production deployment; no environment configuration
- Recommendations: Use environment variables or config file, serve config.json from backend, detect API from window.location

**Insufficient Input Validation on File Upload:**
- Risk: File upload accepts any CSV without schema validation
- Files: `backend/app.py` (lines 77-121)
- Current mitigation: Column checks after parsing (lines 100-109), but no file size limits or MIME type checks
- Recommendations: Add `MAX_FILE_SIZE = 100MB` check, validate MIME type, reject files with >1M rows, add file scanning for malicious content

**CSV Injection Vulnerability (Minor):**
- Risk: User-uploaded CSV with formula strings (e.g., `=1+1`) could execute if opened in Excel
- Files: `backend/app.py` (lines 93-98), `frontend/script.js` (lines 128-158)
- Current mitigation: Data is processed numerically, but malicious formulas in `timestamp` column could persist
- Recommendations: Sanitize timestamp column, prefix suspicious formulas with single quote when re-exporting

**Model File Path Traversal Not Mitigated:**
- Risk: `filepath` parameter in model save/load could theoretically be exploited
- Files: `backend/model.py` (lines 348-393)
- Current mitigation: Path constructed from hardcoded directory (`saved_models/`)
- Recommendations: Validate filepath doesn't contain `..`, use `os.path.abspath()` and assert it's within allowed directory

## Performance Bottlenecks

**Synchronous Flask Server for Long-Running Training:**
- Problem: Training runs in main Flask thread; blocks all requests during model.train_model() (50 epochs)
- Files: `backend/app.py` (lines 124-153), `backend/model.py` (lines 172-236)
- Cause: No async/background task support; synchronous training loop takes 30-120 seconds
- Impact: Frontend progress bar stuck, any other API calls timeout, bad UX during training
- Improvement path: Use Celery + Redis for background tasks, or implement streaming progress with webhooks

**Client-Side Historical Data Generation on Every Prediction:**
- Problem: JavaScript generates 12 hours of fake historical data with random perturbations for every prediction
- Files: `frontend/script.js` (lines 254-264)
- Cause: Model expects 12-step sequences but API only accepts single input set
- Impact: Reduces prediction quality (synthetic history isn't real), adds latency (random number generation + transformation)
- Improvement path: Store actual historical window on backend, send only current features to `/predict`, return confidence interval

**MinMaxScaler Refitting on Every Prediction:**
- Problem: Scaler is fitted once during training, but if data distribution changes, predictions degrade
- Files: `backend/model.py` (lines 144-145, 191-192)
- Cause: Scaler is never retrained or validated
- Impact: Predictions degrade over time if new traffic patterns emerge; no retraining mechanism
- Improvement path: Implement online learning with periodic retraining, track data distribution shift

**Feature Importance Calculation Uses Perturbation (Expensive):**
- Problem: Each feature importance call reruns predictions 4+ times with modified inputs
- Files: `backend/explainer.py` (lines 138-165)
- Cause: Perturbation-based importance requires multiple forward passes
- Impact: `/predict` endpoint latency ~200-500ms for explanations alone
- Improvement path: Cache SHAP values, use gradient-based importance (faster), or run explanations asynchronously

**No Caching of Predictions:**
- Problem: Identical input always reruns full forward pass through LSTM + Transformer
- Files: `backend/app.py` (lines 156-198)
- Cause: No caching layer between API and model
- Impact: 100 identical requests = 100 redundant model executions
- Improvement path: Redis cache with 5-10 minute TTL, or local in-memory LRU cache for common inputs

## Fragile Areas

**Tight Coupling Between Frontend Progress and Backend Training:**
- Files: `frontend/script.js` (lines 186-234), `backend/app.py` (lines 124-153)
- Why fragile: Frontend simulates progress independently; if backend takes longer than expected, progress bar shows 100% before training completes
- Safe modification: Implement server-sent events (SSE) or WebSocket for real-time progress, remove client-side simulation
- Test coverage: No integration tests for training pipeline

**Model Architecture Assumptions in Explainer:**
- Files: `backend/explainer.py` (lines 69-105), `backend/model.py` (lines 286-336)
- Why fragile: Explainer assumes model has `get_attention_weights()` method; if architecture changes, falls back to uniform weights silently
- Safe modification: Check model type/version before calling attention extraction, raise errors instead of silent fallback
- Test coverage: No tests for attention extraction failure modes

**Hardcoded Greenshields Model Parameters:**
- Files: `backend/app.py` (lines 64-69), `prepare_kaggle_data.py` (lines 92-95)
- Why fragile: FREE_FLOW_SPEED=70, ROAD_CAPACITY=7200 hardcoded; doesn't work for different roads/datasets
- Safe modification: Move to config file, add validation that derived speeds are physically plausible (10-80 mph range)
- Test coverage: No tests validating Greenshields model correctness

**Timestamp Parsing Without Timezone Handling:**
- Files: `backend/app.py` (lines 59-71), `prepare_kaggle_data.py` (lines 77-85)
- Why fragile: All timestamps parsed as naive datetime; no timezone info preserved
- Safe modification: Store timezone explicitly, use `pd.to_datetime(..., utc=True)`, document expected timezone
- Test coverage: No tests for timezone edge cases

**No Validation of Sequence Length Consistency:**
- Files: `backend/model.py` (lines 259-266)
- Why fragile: If input has <12 steps, pads with first value; if >12, truncates to last 12; unclear which approach is correct
- Safe modification: Reject inputs that don't match seq_length=12, document why padding/truncation is needed
- Test coverage: No tests for short/long sequences

## Scaling Limits

**In-Memory Model Storage:**
- Current capacity: One PyTorch model ~100MB in memory; scaler pickled alongside (~1KB)
- Limit: 10+ simultaneous predictions can cause memory issues, no multi-model support
- Scaling path: Load models on-demand with caching, use distributed inference (Ray/TensorFlow Serving), implement model versioning

**CSV Processing Without Streaming:**
- Current capacity: Tested with 48k rows (Kaggle dataset)
- Limit: Loading entire CSV into pandas DataFrame; 1M+ row files cause memory spikes
- Scaling path: Use Dask or SQLite for chunked processing, stream processing for large uploads

**No Request Rate Limiting:**
- Current capacity: Flask default (single thread) handles 1-5 req/sec
- Limit: No rate limiting; malicious user can DOS with prediction spam
- Scaling path: Add `Flask-Limiter`, implement request queuing, use Gunicorn + multiple workers

**No Database—File System as Storage:**
- Current capacity: CSV files in `data/` and `saved_models/`
- Limit: No concurrent access control; multiple training jobs can overwrite same model file
- Scaling path: Move to PostgreSQL for metadata, store models in object storage (S3), implement locking

## Dependencies at Risk

**PyTorch 2.1.2 — Potential API Deprecation:**
- Risk: `weights_only=False` parameter in `torch.load()` is deprecated and will be removed
- Files: `backend/model.py` (line 374)
- Impact: Code breaks on PyTorch 3.0+
- Migration plan: Update to `weights_only=True` with proper model checkpoint structure, validate state dicts before loading

**Flask 3.0.0 — Minor Version Lock:**
- Risk: Version pinned to 3.0.0; security patches in 3.0.1+ not pulled
- Files: `backend/requirements.txt` (line 1)
- Impact: Potential Flask security vulnerabilities
- Migration plan: Use `Flask>=3.0.0,<4.0.0` for patch updates, test against latest 3.x versions

**scikit-learn 1.4.0 — MinMaxScaler Potential Changes:**
- Risk: `MinMaxScaler` API could change in 2.0+
- Files: `backend/model.py` (line 9), `backend/requirements.txt` (line 6)
- Impact: Scaler pickle incompatibility, retraining required
- Migration plan: Add version checks for scaler pickle loading, implement fallback unpickler

**Chart.js from CDN (No SRI):**
- Risk: CDN-served library with no Subresource Integrity check
- Files: `frontend/index.html` (line 11)
- Impact: Potential supply chain attack if CDN is compromised
- Recommendations: Add `integrity` attribute with SRI hash, or vendor locally

## Missing Critical Features

**No Model Versioning:**
- Problem: Only one model file (`traffic_model.pth`); no A/B testing, rollback, or comparison
- Blocks: Can't safely deploy new models without losing old one, no experiment tracking
- Impact: If new training is worse, no easy way to revert
- Priority: Medium—necessary for production use

**No Input Feature Validation Beyond Column Checks:**
- Problem: Accepts any numeric values; no bounds checking on speed/volume/hour/day
- Blocks: Model can receive nonsensical inputs (speed=-100, hour=25) without warning
- Impact: Predictions unreliable, no graceful degradation for bad data
- Priority: High—simple to add validation

**No Model Confidence/Uncertainty Quantification:**
- Problem: Returns single point prediction, no confidence interval or error estimate
- Blocks: User doesn't know if prediction is reliable
- Impact: False confidence in edge cases
- Priority: Medium—requires Bayesian or ensemble approach

**No Retraining Pipeline:**
- Problem: Model trained once; no online learning, active learning, or scheduled retraining
- Blocks: Can't improve over time with new data, performance degrades as distribution shifts
- Impact: Stale model, degrading accuracy over months
- Priority: High—critical for production systems

**No Explainability for Failure Cases:**
- Problem: Feature importance and counterfactuals calculated same way regardless of prediction quality
- Blocks: Can't distinguish high-confidence vs low-confidence predictions
- Impact: Users trust bad predictions equally
- Priority: Medium—add confidence threshold

**No Data Quality Monitoring:**
- Problem: No checks for missing values, outliers, or data distribution shift
- Blocks: Bad data silently accepted and trained on
- Impact: Model degrades silently when data quality drops
- Priority: Medium—add data validation and alerting

## Test Coverage Gaps

**Untested Prediction Edge Cases:**
- What's not tested: Predictions with <12 historical steps, mismatched dimensions, extreme values (speed=1, volume=100000)
- Files: `backend/model.py` (lines 242-284), `backend/app.py` (lines 156-198)
- Risk: Silent failures, NaN outputs, cryptic errors
- Priority: High—simple edge case tests

**No Integration Tests for Full Pipeline:**
- What's not tested: Upload → Train → Predict workflow end-to-end
- Files: `backend/app.py` (all endpoints), `frontend/script.js` (all functions)
- Risk: Breaking changes between layers go undetected
- Priority: High—would catch progress bar mismatch, scaler issues

**Explainer Function Robustness Not Tested:**
- What's not tested: `explain_prediction()` with malformed input, missing attention weights, extreme feature importance values
- Files: `backend/explainer.py` (all functions)
- Risk: Explanation generation crashes on edge cases
- Priority: Medium—add failure mode tests

**Frontend API Error Handling Not Tested:**
- What's not tested: API returning 500, timeout, missing fields in response
- Files: `frontend/script.js` (lines 128-286)
- Risk: UI breaks or displays corrupted data on API errors
- Priority: Medium—add mock server tests

**Scaler Persistence Not Tested:**
- What's not tested: Saving and loading scaler from disk, version compatibility
- Files: `backend/model.py` (lines 348-393)
- Risk: Scaler corruption, unpickling failures on different Python versions
- Priority: Low—rare issue but high impact

---

*Concerns audit: 2026-03-24*
