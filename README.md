# EduChain Intelligence Platform

A Django 5 / DRF school ranking and analytics platform. Schools submit indicators, the ministry verifies them,
a configurable weighted model ranks every school, and each published ranking is written to a tamper-evident
hash-chain ledger with a verifiable certificate.

## Run it

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python manage.py makemigrations      # migrations are generated, not committed
python manage.py migrate
python manage.py seed_demo           # 40 synthetic schools, 2023-2025, published and on the ledger
python manage.py runserver
```

Open http://127.0.0.1:8000/ (dashboards), /admin/ (management), /api/v1/ (REST).
Demo logins (password `educhain-demo-2026`): `admin`, `ministry`, `schooladmin`, `parent`, `researcher`.
All demo schools and numbers are synthetic. Do not use this password or `seed_demo --reset` on a real database.

With Postgres, Redis and Celery: copy the env vars from `docker-compose.yml`, then `docker compose up`.

## Data hub (CSV upload and download)

Ministry officials and super administrators open **Data hub (CSV)** in the left menu (`/data/`).

- **Templates**: blank schools and indicators files with the exact headers. A column guide on the page lists what each column accepts.
- **Upload**: choose schools or indicators, pick the file, and the whole file is checked first. Every problem is listed with its line and column. Nothing is saved unless every row passes, then you confirm.
- **Indicators**: each row is matched to a school by registration number. Choose *submit for verification* or *verify now* (locks the data and writes it to the ledger). Verified school-years are locked and reported, never overwritten.
- **Download**: all schools, indicators (filter by year and status) and rankings for any year. Exports are safe to open in Excel (formula-injection guarded) and re-importable.
- Every upload, confirmation and download is written to the audit log; upload history is in the admin under *Import jobs*.

## What is implemented

| Area | Where | Notes |
|---|---|---|
| School profiles and ~40 indicators | `schools/` | Draft -> submitted -> verified (locked, hash recorded) |
| Weighted ranking | `rankings/scoring.py`, `rankings/services.py` | Weights editable by super admin via API/admin; normalised to sum 1; change is audited and written to the ledger |
| National / provincial / district ranks | same | Ties share a rank |
| Ledger | `blockchain/` | Hash-chained, HMAC-signed, proof-of-work blocks in the database. Records: methodology, config changes, data validation, every school ranking |
| Certificates | `/verify/<hash>/`, `/api/v1/blockchain/certificate/<id>/` | Checks chain integrity AND that the stored ranking still equals the ledger record |
| ML | `analytics/ml.py` | Random forest factor importance, tier classifier (decision tree vs SVM vs neural net by cross-validation), KMeans clustering, next-year score model |
| Forecasting | `forecasting/engine.py` | Holt smoothing / linear trend for score, pass rate, enrollment, revenue, teachers required |
| Insights | `rankings/insights.py` | Rule-based rank explanations, risk flags, recommendations, executive summary |
| Dashboards | `dashboards/` | Left-rail layout with light and dark themes. Pages: national overview, rankings (table / cards / compact, filters, sort by any category, CSV download), provinces and province detail, leaderboards by category, trends and movers with tier-movement matrix, find-my-school (personal priority sliders and saved schools), compare, ledger explorer, methodology, certificate verification, early warning (staff) |
| API | `api/` | `/schools /rankings /blockchain /analytics /predictions /insights /reports`, JWT, URL versioning (`v1`), throttling |
| Reports | `reports/` | CSV, Excel, PDF; Celery beat schedules monthly / quarterly / yearly |
| Security | `users/` | RBAC (5 roles), TOTP MFA, AES-256-GCM encryption of MFA secrets, audit log, anonymised researcher dataset (HMAC pseudonyms) |
| Deployment | `Dockerfile`, `docker-compose.yml`, `k8s/`, `.github/workflows/ci.yml` | Gunicorn; Nginx / cloud specifics left to your environment |

## Not implemented (be aware before promising these)

- **Apache Spark, Hadoop HDFS, Kafka**: analytics run in-process with pandas/numpy/scikit-learn. `analytics/services.py` functions return plain dicts and are the natural place to move work onto Spark.
- **Hyperledger Fabric / Ethereum**: the ledger is a built-in permissioned hash chain, not a distributed network. A single database administrator could rewrite the whole chain, including signatures if they hold the signing key. For real anti-corruption guarantees, anchor block hashes on an external network or run several independent validator nodes (`LedgerNode` is the placeholder for this).
- **OAuth2 login**: only JWT (+ optional TOTP). Add `django-oauth-toolkit` if you need OAuth2 provider features.
- **Prophet / ARIMA / LSTM / TensorFlow**: with a few annual points per school, Holt smoothing is used instead. XGBoost, LightGBM and CatBoost are used automatically for feature importance if installed.
- **Generative AI**: insights are deterministic templates, not an LLM. Hook an LLM in `rankings/insights.py` if you want free text.
- **Plotly / D3**, Google Cloud / Azure / AWS specifics, Jenkins, bonus features (chatbot, scholarship engine, ESG rating).
- **GDPR**: anonymisation, audit trail and encryption are provided; legal compliance (consent, retention, erasure workflows) is not.

## Tests

```bash
pytest tests/test_core.py        # pure logic: scoring, ranks, ledger tamper detection, forecasting, ML, TOTP, AES, reports
pytest tests/test_csvio.py tests/test_trends.py   # pure: CSV validation, year-on-year logic
pytest tests/test_pipeline.py    # Django end-to-end: seed, publish, tamper detection, API permissions, CSV hub, every page renders
```
