# Rama Llama — OPIM 5509 final project: what changed and how to ship it

## The model
An ensemble of 5 GRU encoder-decoder networks averaged inside one `model.keras` (Keras only, as the rules require),
giving the 10th / 50th / 90th percentile of Connecticut load for each of the next 24 hours.

| Change vs. FP1/FP2 | Why |
| :-- | :-- |
| Trained on **HRRR forecasts as issued** (the run that really existed ~7 h after it started, plus the run before) | live, the model only ever sees a forecast |
| **Sunlight** (`solar_wm2`, from HRRR) as a 6th weather column | rooftop solar hides midday load (~3 MW CRPS on 2025) |
| **Encoder-decoder**: Conv1D + GRU read the past week, a GRU walks the next 24 hours | one row per forecast hour with its own weather |
| **Load anchors** per future hour: same hour yesterday, same hour last week, newest hour | lets it copy yesterday's shape on calm days |
| **5 networks averaged**, final fit on every hour through Oct 2026 | ~5 MW CRPS from averaging; newest data carries 2026's load shape |

Numbers (validation 2025 with HRRR as issued, trained 2023-24): see `notebooks/FP_Final_Rama_Llama.ipynb` and the table
in the chat summary.

## Files
* `notebooks/FP2_Probabilistic_LSTM_Rama_Llama.ipynb` — **submit this as the Part 2 notebook**: FP2's Parts 0-9 with
  my model; loads my three trained ensembles (`models/eval/`, `models/live/`) and recomputes every score and figure
  (Figure 1 loss curve ... Figure 5 permutation importance, biggest-misses table). `RETRAIN = True` retrains them.
* `notebooks/FP_Final_Rama_Llama.ipynb` — the shorter train-from-scratch version (Colab, T4 GPU, ~15 min): data, the
  `make_inputs` contract + check, network, 2025 calibration, 2026 test, final fit, ship.
* `models/live/model.keras`, `models/live/model_card.json` — the trained model ready to ship.
* `cloud_function/forecast-24h-dl/main.py` — `make_inputs_rama_v1` (word-for-word copy of the notebook's
  `make_inputs`), chosen when the card says `"feature_set": "rama_v1"`; the old FP1/FP2 path still works. HRRR
  `solar_wm2` is read from the class file; if that file is missing the NWS forecast is used and sunlight is estimated
  from its cloud cover. Keras/TensorFlow are pinned to the versions the model was saved with.
* `.github/workflows/rama-ops.yml` — ships `models/live/` on push to main; snapshots live data to the `live-data`
  branch every hour.
* `.github/workflows/deploy-forecast-24h-dl.yml` — now also redeploys when the function code changes on main.

## Ship it (in this order)
1. **Deploy the new function** — Actions → "Deploy forecast-24h-dl" → Run workflow (branch `main`). Wait for green.
   (It must be deployed before the new model is uploaded: the old function can't build the new inputs.)
2. **Ship the model**, either way:
   * **A (one-time permission, then automatic):** in Cloud Shell, once:
     ```
     gcloud storage buckets add-iam-policy-binding gs://energy-raa20002-f26-515102 --member=serviceAccount:cf-deployer@energy-raa20002-f26-515102.iam.gserviceaccount.com --role=roles/storage.objectAdmin
     ```
     then Actions → "Rama ops" → Run workflow. From then on any push to `models/live/` ships automatically and live
     data lands on the `live-data` branch every hour.
   * **B (Colab):** open the notebook, set `UPLOAD = True` in Part 8 and run all (it retrains and ships).
3. Check the leaderboard's Final project tab ~2 hours later; aim for ~80% inside. To change the range width, only
   `range_widen` in `model_card.json` changes (no retraining).

## Rollback
Every shipped model is copied to `energy/models/history/`. Copy an older `model_*.keras` + `model_card_*.json` back to
`energy/models/model.keras` / `model_card.json`. The function handles both old (FP1) and new cards.
