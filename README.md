# GCP Cloud Run Lab

A small FastAPI service used to learn Git, GitHub, Docker, and the Google Cloud Platform
production lifecycle (Cloud Build, Artifact Registry, Cloud Run, Secret Manager, BigQuery,
and observability).

## Endpoints

- `GET /` — basic hello
- `GET /health` — health check

## Local development

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```
