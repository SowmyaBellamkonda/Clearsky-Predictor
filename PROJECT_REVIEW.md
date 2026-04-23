# ClearSky Project Review

Date: 2026-04-06
Reviewer: GitHub Copilot (GPT-5.3-Codex)

## Executive Summary

The project has a strong product direction and solid feature depth (real-time AQI aggregation, ML forecast integration, eco-scoring, and a conversational assistant). The main risks are currently in security hygiene and production readiness.

Overall assessment:
- Product and architecture direction: Good
- Security posture: High risk (must fix before sharing/deploying)
- Reliability and maintainability: Moderate risk
- Deployment readiness: Not ready for production

## Key Findings (Ordered by Severity)

### 1. Critical: Service account private key committed to repository
- Evidence: [ml-service/gee-key.json](ml-service/gee-key.json#L1)
- Why this matters: A real private key in source control can be used by unauthorized parties to access Google Cloud/Earth Engine resources, incur billing, and exfiltrate data.
- Recommendation:
1. Immediately revoke/rotate this key in Google Cloud IAM.
2. Remove the file from git history and keep only a template (for example, `gee-key.example.json`).
3. Load credentials via secure runtime secret injection (environment variable or secret manager).

### 2. High: Sensitive/runtime artifacts are not ignored, and a virtual environment appears tracked
- Evidence: [.gitignore](.gitignore#L1) (missing ignores for Python venvs and service credentials), and dependency files under [ml-service/venv/Lib/site-packages/fastapi/applications.py](ml-service/venv/Lib/site-packages/fastapi/applications.py#L1)
- Why this matters: Repository bloat, accidental secret leakage, noisy diffs, slower CI, and harder collaboration.
- Recommendation:
1. Add ignores for `ml-service/venv/`, `.venv/`, `__pycache__/`, `.env`, `ml-service/gee-key.json`, and model binaries where appropriate.
2. Untrack already committed artifacts with `git rm -r --cached` and recommit cleanly.

### 3. High: Frontend and backend URLs are hardcoded to localhost
- Evidence:
1. [src/services/aqiService.js](src/services/aqiService.js#L4)
2. [src/context/AQIContext.jsx](src/context/AQIContext.jsx#L48)
3. [src/components/AssistantPanel.jsx](src/components/AssistantPanel.jsx#L45)
4. [src/components/MapOverlay.jsx](src/components/MapOverlay.jsx#L11)
- Why this matters: Breaks staging/production deployments and prevents environment-specific routing.
- Recommendation:
1. Use `import.meta.env.VITE_API_BASE_URL` in frontend with a single service-level source of truth.
2. Keep backend service URLs in backend env (already partly done with `ML_SERVICE_URL`).

### 4. High: Inconsistent Python dependency specification
- Evidence:
1. [ml-service/requirements.txt](ml-service/requirements.txt#L1) does not include required runtime packages used in code.
2. [ml-service/app.py](ml-service/app.py#L40) imports TensorFlow dynamically.
3. [ml-service/eco_service.py](ml-service/eco_service.py#L10) uses `google.oauth2` and `requests`.
4. [README.md](README.md#L55) requires manual extra `pip install` steps.
- Why this matters: Non-reproducible environments and onboarding failures.
- Recommendation:
1. Make `requirements.txt` complete and authoritative.
2. Remove ad-hoc installation instructions that bypass lockstep dependency management.

### 5. Medium: Location history lookup likely misses data due to exact float matching
- Evidence: [backend/routes/api.js](backend/routes/api.js#L129) matches Mongo records by exact latitude/longitude equality.
- Why this matters: Slight coordinate jitter from GPS means historical sequences may not match, reducing ML context quality.
- Recommendation:
1. Normalize coordinates before storing/querying (for example, 3-4 decimal precision).
2. Or query by geospatial radius (`$near`) plus time window.

### 6. Medium: Current map location state initialization can become stale
- Evidence: [src/components/MapOverlay.jsx](src/components/MapOverlay.jsx#L142) initializes `mapLocations` from initial context values only once.
- Why this matters: Initial coordinates default to `0,0` in context and may persist in map state after geolocation updates, causing confusing UX.
- Recommendation:
1. Add a sync effect that updates the `isCurrent` location entry when context `coords`, `locationName`, or `aqiValue` changes.

### 7. Medium: Open CORS policy with no environment guard
- Evidence: [backend/server.js](backend/server.js#L17) uses `app.use(cors())` with default permissive behavior.
- Why this matters: Unrestricted cross-origin access increases misuse risk in production.
- Recommendation:
1. Restrict origins via env-configured allowlist.
2. Use stricter methods/headers defaults and per-route tightening where possible.

### 8. Medium: Error responses lose actionable context
- Evidence: [backend/routes/api.js](backend/routes/api.js#L411) and [backend/routes/api.js](backend/routes/api.js#L570) return generic failures while logging only brief messages.
- Why this matters: Harder diagnosis in production and poor client troubleshooting.
- Recommendation:
1. Add structured error logging with source service, status code, and request correlation id.
2. Return safe but useful error envelopes to clients.

### 9. Low: Documentation/setup mismatch for frontend working directory
- Evidence: [README.md](README.md#L69) instructs `cd src` then `npm install`, but root-level [package.json](package.json#L1) defines the frontend project.
- Why this matters: New contributors may fail setup or install dependencies in the wrong location.
- Recommendation:
1. Update setup instructions to run frontend commands from project root.

## Positive Observations

- Good endpoint segmentation and layering between frontend, Node backend, and FastAPI ML service.
- Backend rate limiting is in place for API routes: [backend/server.js](backend/server.js#L24).
- AQI schema includes a practical compound index for temporal/spatial queries: [backend/models/AQIData.js](backend/models/AQIData.js#L31).
- The forecast response model combining OWM short-term intervals and LSTM +24h forecast is product-friendly: [backend/routes/api.js](backend/routes/api.js#L373).

## Priority Fix Plan

### Immediate (today)
1. Rotate/revoke exposed GEE key and remove from repository.
2. Harden `.gitignore` and untrack `venv` and sensitive files.
3. Externalize all frontend API URLs into environment variables.

### Short term (this week)
1. Normalize geolocation storage/query strategy.
2. Complete `requirements.txt` and verify clean environment bootstrapping.
3. Tighten CORS policy for non-local environments.

### Follow-up (next sprint)
1. Add automated tests for `/predict`, `/eco-score`, and `/chat` happy/error paths.
2. Add health-check and readiness checks for both backend and ML service.
3. Add structured logging and basic observability (request ids, latency, downstream failures).

## Testing/Validation Notes

- Static review performed across core frontend, backend, and ML service code paths.
- IDE-reported problems check returned no active diagnostics at review time.
- No end-to-end runtime execution was performed during this review.
