# Planned & Completed Improvements

This document tracks future enhancements and recently completed improvements for the ClearSky project.

## Future Enhancements
* **Intelligent Historical Data Fetching (Cold Start Solution)**
  * **Current State:** The backend currently uses a "naive backfill" with ±5% random noise to simulate past 7 days' data when a user searches for a new location with no database history. While this prevents the LSTM model from throwing an error, it essentially creates a persistence model and lowers initial prediction accuracy for new locations.
  * **Proposed Improvement:** Instead of faking the history, dynamically fetch the actual past 7 days of historical weather and pollution data retroactively.
  * **Implementation Strategy:** Use the **Open-Meteo API** (which provides free historical weather and air quality without API keys or paywalls) to gather the full 14-feature set for the last 7 days in a single, fast call. To prevent user latency on the first load, this could be triggered asynchronously, or the Open-Meteo call is fast enough to perform synchronously before calling the ML service.
  * **Expected Impact:** Massive leap in the accuracy of the 24-hour LSTM forecast for newly queried locations, avoiding the 6-day "accumulation" period required before predictions become highly precise under the current model.

## Completed Improvements
*(List completed structural, performance, or feature improvements here as they are implemented)*
