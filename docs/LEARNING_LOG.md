# GCP Production Learning Log

A chronological record of this project: what we did, why, the commands used, and the
concepts covered. Goal is to end up with a trail I can read back to explain how I built,
deployed, and operated this system.

---

## 2026-08-29 — Environment check & local project setup

**Context:** Starting from an empty directory. Goal for today: get a local FastAPI app
running and ready for its first git commit. No GCP resources touched yet.

**What we did:**
- Verified local tooling: Python 3.12.7, git 2.52.0 installed; GitHub CLI (`gh`) not
  installed (deferred — will create the GitHub repo via the website, or install `gh`
  later if preferred).
- Created a Python virtual environment (`python3 -m venv .venv`, `source .venv/bin/activate`)
  to isolate this project's dependencies from the system Python.
- Created the initial project structure:
  ```
  GCP_CR_LAB/
  ├── app/
  │   ├── __init__.py
  │   └── main.py
  ├── requirements.txt
  ├── .gitignore
  ├── README.md
  └── docs/LEARNING_LOG.md
  ```
- Wrote a minimal FastAPI app (`app/main.py`) with two routes:
  - `GET /` — basic hello response.
  - `GET /health` — health check, returns `{"status": "ok"}`. This is the kind of
    lightweight endpoint Cloud Run and load balancers poll to confirm a container is
    alive and ready to receive traffic.
- Pinned dependencies in `requirements.txt`: `fastapi`, `uvicorn[standard]`.
- Set up `.gitignore` to exclude `.venv/`, `__pycache__/`, `*.pyc`, and `.env` — none of
  these should ever be committed (regenerable, or reserved for secrets).

**Concepts covered:**
- **Virtual environment** — a self-contained Python interpreter + package set per
  project, so dependencies don't leak across projects or conflict with system Python.
- **`.gitignore` purpose** — keep the repo limited to source of truth (code, config
  templates), not generated artifacts or secrets.
- **Health check endpoint** — why production platforms like Cloud Run expect one.

**Status:** App not yet run/tested locally, not yet committed to git, no GitHub repo
created yet.

---

## 2026-08-29 — GCP foundations: resource hierarchy, IAM, APIs, gcloud CLI

**Context:** Before touching any GCP resources, covered the foundational concepts
needed to understand everything that follows (Cloud Build, Artifact Registry, Cloud Run,
Secret Manager, BigQuery all sit inside this hierarchy and IAM model).

**Concepts covered:**
- **Resource hierarchy:** `Organization (optional) → Folder (optional) → Project →
  Resources`. IAM policies inherit downward. A personal Google account with no
  Workspace/org typically creates projects with "No organization" as the parent —
  relevant for this project, since org/folder-level IAM won't come up here.
- **Project identity:** Project ID (human-chosen, globally unique, immutable) vs
  Project Number (auto-generated, immutable) vs Project Name (display label only,
  mutable, not unique) — three distinct things.
- **Correction:** a project does not have "a region." Region/zone is chosen per
  resource at creation time, not once for the whole project (App Engine is the one
  exception, not used here).
- **IAM model:** identity (user account or service account) + role (bundle of
  permissions — basic/predefined/custom) + resource = a policy binding. "Predefined vs
  custom" describes roles, not policies. IAM policy (identity-based access) is a
  different mechanism from Organization Policy (governance constraints on resource
  configuration, independent of identity) — not used in this project.
- **Service accounts:** non-human identity for workloads. Auto-created default service
  accounts (e.g. Compute Engine default SA) tend to be over-privileged (historically
  `Editor`) — we'll create a dedicated, narrowly-scoped service account for the Cloud
  Run app instead. Inside GCP, workloads authenticate via Application Default
  Credentials (ADC) through the metadata server — no key files to manage.
- **APIs:** every GCP service must be explicitly enabled per project
  (`gcloud services enable <api>.googleapis.com`) before use — a common source of
  "permission denied" errors when a needed API simply isn't turned on yet.
- **gcloud CLI:** `gcloud config list` / `gcloud config set project` for active context;
  `gcloud auth login` (authenticates the CLI) vs `gcloud auth application-default login`
  (sets credentials used by client libraries like Python's `google-cloud-*` packages) —
  two distinct credential types, both needed eventually.

**Where permissions will live in this project:** project-level or narrower
(resource-level, e.g. one secret or one bucket), matching least-privilege — org/folder
level bindings are for broad cross-project roles (security/platform admin) that don't
apply to a single-developer learning project.

**Status:** No GCP resources created yet. Local app still pending its first local run
and first git commit.

---

<!-- Add new dated entries below this line as we progress through the project. -->
