# Cross-Environment TPC-H Runtime Prediction (CETP)

CETP is an applied machine learning project for predicting how a TPC-H query will perform on a different machine than the one where the query plan was captured. The system uses PostgreSQL `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` output from a development environment, combines it with static hardware signatures, and predicts whether a query is likely to violate an SLA when deployed elsewhere.

The core idea is to replace expensive cross-machine benchmarking with a model-based runtime estimate. Instead of running a full benchmark suite on every candidate production machine, CETP extracts workload features, classifies the likely bottleneck, estimates runtime scaling factors, and defers uncertain predictions when the target hardware is outside the training distribution.

## What this project includes

- Query-plan feature extraction from PostgreSQL execution plans
- Bottleneck classification to estimate whether the workload is bandwidth-, compute-, I/O-, or mixed-bound
- Hardware signature modeling using memory bandwidth and multi-core compute throughput
- Quantile regression for predicting p50, p95, and p99 scaling behavior across machines
- Confidence gating to avoid unsafe predictions on unfamiliar hardware
- A command-line gate that returns PASS, WARN, or BLOCK verdicts
- A demo pipeline and CI-style validation flow for example queries

## Why it matters

Database query performance does not transfer cleanly across hardware. A query that runs well on a development machine can become slow or unstable on production infrastructure. CETP addresses that gap by giving a fast, model-based estimate before deployment, helping teams decide whether a query should be tested further or blocked.

## High-level approach

CETP decomposes the problem into two parts:

1. Workload cost from the development machine
   - planner cost
   - shared read and hit blocks
   - row counts
   - buffer and I/O behavior

2. Hardware signature from the target machine
   - memory bandwidth
   - multi-core compute throughput

A Random Forest classifier estimates the bottleneck class, and gradient-boosted quantile regressors predict runtime scaling factors across machines. A hardware-distance confidence check avoids overconfident predictions when the target machine is far outside the training distribution.

## Project structure

```text
CETP_v1-main/
├── README.md
├── RUNNING.md
├── gate/
│   ├── cetp_gate.py
│   ├── cetp_gate_demo.py
│   ├── cetp.yml
│   └── examples/
├── dashboard/
│   ├── index.html
│   ├── data.js
│   └── assets/
├── models/
│   ├── bottleneck_classifier.pkl
│   ├── scaling_quantile_models.pkl
│   ├── hardware_signature.pkl
│   └── *.json
├── models_online/
│   └── v2/
├── data/
│   ├── raw/
│   └── corrected/
├── scripts/
│   ├── collection/
│   ├── analysis/
│   └── ...
├── tests/
│   └── test_e2e.py
├── results/
│   ├── phase7_report.md
│   ├── online_predictor_report.md
│   ├── ...
│   └── figures and summary outputs
├── .github/
│   └── workflows/
└── ...
```

## Demonstration

The project includes a runnable demo gate that exercises multiple verdict paths:

```bash
python gate/cetp_gate.py --demo
```

This produces a local verdict report for scenarios such as:

- PASS
- BLOCK
- WARN for borderline SLA risk
- WARN for low-confidence hardware predictions

Example output from the verified built-in scenarios:

```text
q19: c5a -> c7i, SLA 200 ms
CETP Gate verdict: PASS
Predicted runtime: p50=59.1 ms, p95=79.9 ms, p99=129.6 ms

q1: r5n -> z1d, SLA 3000 ms
CETP Gate verdict: BLOCK
Predicted runtime: p50=6104.4 ms, p95=10539.3 ms, p99=42616.8 ms

q18: c7i -> r5n, SLA 8000 ms
CETP Gate verdict: WARN
Predicted runtime: p50=6157.2 ms, p95=8010.4 ms, p99=12830.3 ms
```

`PASS` means predicted p99 is within the SLA. `BLOCK` means predicted p50
already exceeds the SLA. `WARN` means the result needs a canary or additional
measurement, either because the predicted tail exceeds the SLA or because the
target hardware is outside the training range.

You can also run the gate against a real EXPLAIN JSON plan:

```bash
python gate/cetp_gate.py --explain-json path/to/explain.json --config gate/cetp.yml --query-id q1
```

### Run your own SQL query

The gate consumes PostgreSQL's `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)`
output. First run the query on a local PostgreSQL database and save its plan.
On Windows, use the full `psql.exe` path if PostgreSQL is not on `PATH`:

```powershell
cd "C:\clg\My Projects\Cross-Environment-TPC-H-Runtime-Prediction-master"

& "C:\Program Files\PostgreSQL\16\bin\psql.exe" `
  -U postgres -d tpch -X -t -A `
  -c "EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT COUNT(*), AVG(price) FROM lineitem WHERE quantity > 50;" `
  > query_explain.json
```

Then choose the development profile, target profile, and SLA threshold:

```powershell
& "C:\Users\sarth\AppData\Local\Programs\Python\Python314\python.exe" `
  gate\cetp_gate.py `
  --explain-json query_explain.json `
  --dev-machine c7i `
  --prod-machine c5a `
  --sla-ms 3000 `
  --query-id custom_query
```

The supported trained profiles are `c5a`, `m5a`, `r5n`, `z1d`, and `c7i`.
The query is executed on the development PostgreSQL database; the production
machine is used for prediction. The bundled `c5a` profile is not a live
server, so this command does not connect to an actual cloud `c5a` machine.

For trained machines, you can select hardware by name instead of entering
bandwidth and compute values:

```bash
python gate/cetp_gate.py \
  --dev-features gate/examples/gate_queries/q19_c5a_dev_features.json \
  --dev-machine c5a --prod-machine c7i --sla-ms 200 --query-id q19
```

Available trained machines are `c5a`, `m5a`, `r5n`, `z1d`, and `c7i`.

To run the online self-calibration loop:

```bash
python scripts/analysis/addition7_online_loop.py
```

To view the dashboard locally:

```bash
python3 -m http.server 8000 --directory dashboard
```

Then open:

```text
http://localhost:8000
```

The dashboard is a read-only results viewer. It displays the trained machine
specifications, evaluation charts, and saved prediction results; it does not
execute SQL, connect to PostgreSQL, connect to a live `c5a` machine, or
generate new PASS/WARN/BLOCK decisions. Run SQL and the CETP gate from the
command line as described above, then use the dashboard to inspect the
available specifications and results.

To run the validation suite:

```bash
pytest tests/test_e2e.py
```

## Results and validation

The repository includes evaluation reports and output artifacts under `results/`, including analysis of:

- bottleneck classification
- hardware signal quality
- confidence and abstention logic
- online learning / retraining behavior
- regression quality across machines and TPC-H queries

The project is designed to be reproducible and evidence-driven, rather than a purely illustrative demo.

## Limitations

This is a research-oriented prototype rather than a full production system. The repository explicitly documents limitations such as:

- the model is trained on a specific set of hardware configurations
- confidence handling is conservative when the target machine is unfamiliar
- some signal choices are weaker than others under certain workload types
- the project focuses on local benchmark and model validation rather than live production deployment

These are documented in the analysis reports under `results/`.

## Status

Independent project focused on query-performance prediction, SLA risk estimation, and deployment gating for cross-machine SQL performance.

## Notes

The repository also includes a `RUNNING.md` file with step-by-step commands for running demos and validation flows. For the most complete technical discussion of methodology and evaluation, see the reports in `results/`.
