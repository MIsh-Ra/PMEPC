# Pre-Merge Evaluation Pipeline for Containerized ML Models

<!-- > **Status: In progress.** Sections marked `TODO` depend on decisions or measurements not yet made. Do not treat any number in this document as a result until it has been measured and recorded in the Results section. -->

## Overview

This project implements an automated gate that runs before a new machine learning model is merged into the production line. A candidate model is packaged in a Docker container, served through a FastAPI endpoint, evaluated against a fixed held-out test set, and compared against the model currently in production. If the candidate fails any defined check (latency, predictive performance, or per-group fairness), the pull request is blocked and a report explains which check failed and by how much. If it passes, the candidate is promoted to production storage.

The project uses a credit card fraud detection model as the reference example. The gate itself is model-agnostic as long as the model is exposed through the same API contract.

## Problem

Model updates can regress in ways that are not visible from a single aggregate metric. A new version may score better overall while being slower, less accurate on the positive (fraud) class, or substantially more likely to flag legitimate transactions for a specific subgroup. Without an automated check, these regressions are caught late or not at all. This project moves those checks into the pull request workflow so they run on every proposed model change.

## Goals

- Block model merges that regress on latency, F1-score, or per-group false positive rate beyond defined thresholds.
- Make every evaluation reproducible: same test set, same metrics, same code path for production and candidate models.
- Produce a human-readable report on each run, including on failure.
- Keep infrastructure cost low by avoiding always-on cloud servers.

## Non-goals

- This is not a live shadow deployment. It does not receive or evaluate real production traffic.
- It does not retrain models. Training happens locally, and the pipeline evaluates the resulting artifact.
- It does not cover model monitoring after release (drift detection, live metrics). That is a separate concern.
- It does not handle data labeling or dataset versioning beyond storing a single fixed test set.

## Architecture

```
Developer                GitHub                          AWS S3
   |                       |                               |
   | push branch + PR      |                               |
   |---------------------->|                               |
   |                       | GitHub Actions workflow starts|
   |                       |                               |
   |                       | 1. Download test_set.csv  ----|-->  /test-data/
   |                       | 2. Download production model -|-->  /production-model/
   |                       | 3. Build Docker image         |
   |                       | 4. Run container (port 8000)  |
   |                       | 5. evaluate.py sends requests |
   |                       |    to localhost:8000/predict  |
   |                       | 6. Compute latency, F1,       |
   |                       |    per-group FPR              |
   |                       | 7. Compare vs production      |
   |                       |                               |
   |                       | PASS: upload candidate -------|-->  /production-model/
   |                       | FAIL: write report, block PR  |
```

### Components

| Component | Role |
|---|---|
| `model/train.py` | Trains a candidate model and saves it as `.joblib` |
| `api/main.py` | FastAPI app exposing `POST /predict`, loads model weights at startup |
| `api/schemas.py` | Pydantic models validating the request and response shape |
| `Dockerfile` | Packages Python version, dependencies, API code, and model weights |
| `evaluate.py` | Sends test rows to the running container, computes metrics, applies gate rules |
| `gate/thresholds.yaml` | Single source of truth for pass/fail thresholds |
| `gate/report.py` | Writes the markdown report for each run |
| `.github/workflows/ml-gate.yml` | Orchestrates the full pipeline on pull requests |
| `tests/` | Unit tests for the API, schema validation, and metric calculations |

## Model and data

- **Dataset:** TODO. Planned: Kaggle "Credit Card Fraud Detection" dataset (labeled, highly imbalanced, roughly 0.17% positive class).
- **Model:** TODO. Planned: XGBoost or Random Forest via scikit-learn.
- **Split:** A fixed held-out test set of approximately 5,000 rows, never used for training. Stored once in S3 under `/test-data/test_set.csv`. Do not regenerate it between runs, or comparisons are invalid.
- **Input schema:** TODO. List each feature name and type here once the dataset is chosen.
- **Output schema:** `{"fraud_probability": float}` (planned).

## API contract

`POST /predict`

Request:
```json
{
  "features": { "TODO_feature_1": 0.0, "TODO_feature_2": 0.0 }
}
```

Response:
```json
{
  "fraud_probability": 0.82
}
```

Behavior to document once implemented:
- Response for malformed input (status code and error body).
- Whether the model's decision threshold is applied inside the API or in the client.

## Evaluation and gate rules

The gate compares a **candidate** model against the **production** model, both evaluated on the same test set.

| Check | Rule | Threshold |
|---|---|---|
| Latency | Average response time per request | TODO (planned: under 50 ms) |
| Latency tail | 95th percentile response time | TODO |
| Predictive performance | F1-score on the positive (fraud) class | TODO (planned: candidate F1 no more than 0.01 below production) |
| Fairness | False positive rate per group | TODO (planned: no group's FPR more than 20% above the overall FPR) |

### Fairness groups

TODO. Define the groups concretely before coding. Candidates:
- Transaction amount bands (e.g., under 10, 10 to 100, over 100), using the dataset's amount field
- Any other grouping feature available in the dataset

Record the exact definition here so the check is reproducible.

### Decision rules

- A run **passes** only if all checks pass.
- TODO: Decide whether any check can be warning-only, and if so, which.
- TODO: Decide how ties (candidate and production scoring identically) are handled.

## Repository layout (planned)

```
ml-gate/
├── README.md
├── model/
│   └── train.py
├── api/
│   ├── main.py
│   └── schemas.py
├── gate/
│   ├── thresholds.yaml
│   └── report.py
├── evaluate.py
├── tests/
├── Dockerfile
├── requirements.txt
└── .github/
    └── workflows/
        └── ml-gate.yml
```

## Setup

### Prerequisites
- Python 3.10
- Docker
- An AWS account with an S3 bucket
- A GitHub repository with Actions enabled

### Local setup
1. Clone the repository and create a virtual environment.
2. Install dependencies: `pip install -r requirements.txt`
3. Train the initial model: TODO (command)
4. Start the API locally: TODO (command)
5. Send a test request with curl or Postman (example request to be added).

### AWS setup
1. Create an S3 bucket with two prefixes: `test-data/` and `production-model/`.
2. Upload `test_set.csv` to `test-data/`.
3. Upload the initial production model to `production-model/`.
4. Create an IAM user with read and write access restricted to this bucket only. Do not attach broad permissions.
5. Record the access key and secret. Do not commit them.

TODO: Document whether OIDC authentication will be used instead of long-lived access keys, and update this section if so.

### GitHub setup
1. Add `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_REGION` to the repository's Actions secrets.
2. Confirm the workflow file is at `.github/workflows/ml-gate.yml`.

## Running the gate

Locally:
```
python evaluate.py --candidate model/candidate.joblib --production model/production.joblib
```
TODO: Confirm exact arguments once implemented.

In CI: opening or updating a pull request triggers the workflow automatically.

## Testing the gate

The gate is only useful if it blocks bad models. Test both outcomes:

1. **Passing case:** Train a candidate with settings comparable to production. Open a PR. Confirm all checks pass and the model is uploaded.
2. **Failing case:** Train a deliberately weaker candidate (for example, a model with very few trees). Open a PR. Confirm the workflow fails, the report names the failed check, and the production artifact in S3 is unchanged.
3. **Latency case:** TODO. Describe how to force a latency failure (for example, adding an artificial delay in a test branch).
4. **Fairness case:** TODO. Describe how to construct a candidate that fails the fairness check.

Record which tests were run and their outcomes in the Results section.

## Results

Not yet measured. Fill in only after running the tests above.

| Metric | Production model | Candidate model | Notes |
|---|---|---|---|
| F1-score (fraud class) | TODO | TODO | Same test set for both |
| Average latency (ms) | TODO | TODO | Specify local or CI runner |
| 95th percentile latency (ms) | TODO | TODO | |
| FPR, amount band 1 | TODO | TODO | |
| FPR, amount band 2 | TODO | TODO | |
| FPR, amount band 3 | TODO | TODO | |
| Gate outcome | TODO | TODO | Pass / fail and reason |

### Gate test log

| Test | Expected | Actual | Notes |
|---|---|---|---|
| Passing candidate | Pass | TODO | |
| Weaker candidate | Fail on F1 | TODO | |
| Latency failure | Fail on latency | TODO | |
| Fairness failure | Fail on FPR | TODO | |

## Known limitations

TODO. Add as they are discovered. Items to consider:
- Latency measured on a laptop and on a GitHub runner will differ. State which is reported.
- The test set is fixed, so the gate only reflects performance on that distribution.
- Fairness groups are based on the chosen grouping only and do not cover all possible subgroups.
- Long-lived AWS access keys are a security tradeoff unless OIDC is used.
- Concurrent pull requests that pass the gate at the same time may race on the S3 upload. TODO: decide how this is handled.

## Design decisions to record

Keep a short entry here for each decision, with the reason, so the reasoning is available later.

- TODO: Why a fixed held-out test set and not cross-validation.
- TODO: Why the pipeline evaluates through the HTTP API rather than calling the model directly.
- TODO: Why uploads to S3 happen only after the gate passes.
- TODO: Why the thresholds were set at their chosen values.

## Future work

- Add OIDC authentication to replace long-lived access keys.
- Add a second fairness grouping.
- Add a manual approval step for promotion.
- Add shadow evaluation against live traffic as a separate project.

## Checklist before finishing

- [ ] Dataset and model chosen and documented
- [ ] API contract documented, including error responses
- [ ] Thresholds written to `gate/thresholds.yaml`
- [ ] Fairness groups defined and recorded
- [ ] Docker image builds and runs locally
- [ ] `evaluate.py` runs locally against both models
- [ ] Workflow passes on a good candidate
- [ ] Workflow fails on a bad candidate, with a readable report
- [ ] Results table filled with measured values
- [ ] Known limitations updated
- [ ] Setup steps tested on a clean machine