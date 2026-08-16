# Hotel Anomaly Detection POC — Session Context

## What this project is
A proof-of-concept anomaly detection system for a multi-tenant Hotel Management System (HMS), covering expenses, purchases, and stock/inventory — not just raw expense amounts.

## Key decisions made
- **Tenant = property.** One hotel property = one tenant. No separate DBs/schemas for the POC; a shared table with `property_id` scoping is enough.
- **Shared model, tenant-relative features.** One model trained across all tenants, but every feature (price, consumption, etc.) is computed relative to that tenant's own baseline — never compared globally.
- **Two detection layers** (the "proper way," not a shortcut):
  1. **Layer 1 — Deterministic reconciliation.** Check the accounting identity `opening stock + purchased - consumed = closing stock` per property/product/week. Mismatches are hard facts (shrinkage/theft/miscounting), not ML guesses.
  2. **Layer 2 — Statistical/ML layer.** Isolation Forest (or z-score to start) on tenant-relative engineered features, catching price anomalies, consumption anomalies (checked against occupancy, not in a vacuum), and supplier anomalies.
- **Output must be typed and explainable**, not a single opaque score — e.g. `STOCK_DISCREPANCY`, `PRICE_ANOMALY`, `CONSUMPTION_ANOMALY`, `SUPPLIER_ANOMALY`, each with a plain-English reason.
- **Scope for POC**: 4 synthetic properties (2 "Tier1" bigger hotels, 2 "Tier2" smaller ones), 26 weeks of data, 6 products across Food/Beverage/Housekeeping/Maintenance, 5 suppliers.

## User context (important for how to work with them)
- Beginner/learner — wants concepts explained simply, with small concrete examples, **before** code/commands are shown, not after.
- Prefers to run all commands themselves (terminal, Jupyter, git) rather than have Claude execute them — Claude's job is to write code/files and explain; the user runs and reports output back.
- Keep explanations short, plain English, no unexplained jargon.

## Environment setup (already done)
- Python 3.10 (via `py -3.10`), isolated in a `venv` folder (`venv/Scripts/activate` to activate).
- Installed: pandas, numpy, scikit-learn, matplotlib, jupyter.
- Git repo initialized, pushed to `https://github.com/Vatsan-S/Anomaly-Model-POC` (public).
- `venv/` is gitignored — reinstall on any new machine with:
  ```
  py -3.10 -m venv venv
  venv\Scripts\activate
  pip install pandas numpy scikit-learn matplotlib jupyter
  ```

## Build status
- [x] `notebooks/01_generate_synthetic_data.ipynb` — written, **not yet run** by the user. Generates all synthetic tables (`properties`, `occupancy`, `products`, `suppliers`, `purchases`, `stock_ledger`) and a `planted_anomalies_answer_key.csv` with 4 deliberately hidden anomalies (one of each type above), saved to `data/`. The random seed (42) is fixed, so re-running always reproduces identical data — data files don't strictly need to be backed up, just the notebook.
- [ ] Layer 1 — stock reconciliation engine (next notebook, not started)
- [ ] Layer 2 — feature engineering (tenant-relative baselines/deviations)
- [ ] Layer 2 — Isolation Forest anomaly model
- [ ] Merge both layers into one unified, typed, explainable flag list
- [ ] Validate output against the 4 planted anomalies

## Future roadmap (post-POC — not being built yet, noted for later)
These were discussed and deliberately deferred so the POC stays scoped. Revisit only once the core POC (reconciliation + ML layer + validation) proves itself.

- **Feedback loop before re-baselining.** Don't automatically feed a detected anomaly back into "normal" history. A human should confirm it was actually normal first — otherwise abnormal events quietly contaminate future baselines.
- **Minimum sample-size guard on baselines.** Some property+product combos will have very little history (e.g. a rarely-bought item). A baseline computed from 2 data points isn't trustworthy — need a threshold below which we fall back to a broader category-level benchmark instead of a per-product one.
- **Cold-start strategy for brand-new tenants.** A new property has no history at all. Fall back to global/category benchmarks until it accumulates enough of its own data (e.g. 20-30 records), then switch it over to tenant-relative baselines.
- **Per-tenant threshold tuning.** Isolation Forest scores are relative to the population it was trained on — one global cutoff (e.g. 0.70) can over-flag small/quiet tenants and under-flag large/noisy ones. Fine as a single global threshold for the POC; flagged as a known simplification to fix later.
- **Explainability alongside the anomaly score.** Every flagged item should say *why* (which feature deviated most, e.g. "price/night 4x property average"), not just show a raw score — this is what makes the output usable to a non-technical hotel/finance reviewer, not just technically correct.
- **Representation/embedding layer.** Explicitly deferred — expense/inventory data is low-dimensional and tabular, so hand-crafted features + baselines should cover most cases without it. Only worth building if there's a proven cross-tenant pattern that features can't catch, e.g.:
  - A supplier who behaves normally at each individual property but is part of a fraud pattern only visible in aggregate across many tenants.
  - Better cold-start via "borrow a baseline from similar tenants" instead of a generic global fallback.
  Only build this after an A/B comparison shows the simpler model isn't enough — never build it speculatively.
- **Retraining schedule.** As more data accumulates and tenant behavior drifts (seasonality, price inflation, renovations), the model and baselines will need periodic retraining — not designed yet.
- **Productionization.** Real-time inference pipeline (new event → feature extraction → baseline lookup → deviation calc → anomaly model → score → threshold → flag), proper multi-tenant API/dashboard access control (never trust a client-passed tenant ID — derive it server-side from the authenticated session), and monitoring — all out of scope for the POC.

## Next step when resuming
Run `notebooks/01_generate_synthetic_data.ipynb` fully if not already done (Jupyter: `jupyter notebook` from project root with venv active), confirm `data/` folder gets created with 7 CSVs, then move on to building the Layer 1 reconciliation notebook — same teach-first approach (explain the concept and a tiny numeric example before showing code).
