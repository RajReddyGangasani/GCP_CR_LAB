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

## 2026-08-30 — First GitHub push & gh CLI setup

**What we did:**
- Renamed local default branch `master` → `main`.
- Ran the app locally with `uvicorn app.main:app --reload`, confirmed `/` and `/health`
  both respond.
- Installed and authenticated GitHub CLI (`gh auth login`, browser flow).
- Created the GitHub repo with `gh repo create GCP_CR_LAB --public --source=. --remote=origin`
  — this created the repo on GitHub AND added it as the local `origin` remote in one step.
- Pushed with `git push -u origin main` — the `-u` flag set `main` to track `origin/main`
  as its upstream, so future `git push`/`git pull` on this branch no longer need the
  remote/branch spelled out.
- Repo is live: https://github.com/RajReddyGangasani/GCP_CR_LAB

**Concepts covered:**
- `gh repo create` vs plain `git push` — plain `git push` never creates a remote repo;
  it only pushes to one that already exists. Repo creation requires the GitHub
  website, API, or `gh` CLI.
- Upstream tracking branch (`-u` / `--set-upstream`) and how to verify it
  (`git branch -vv`, look for `[origin/main]`).

**Status:** Local dev → Git → GitHub loop is complete for the initial scaffold. Next:
Docker.

---

## 2026-08-30 — Docker fundamentals & first local container

**Context:** Before touching Artifact Registry or Cloud Run, learned Docker itself and
containerized the FastAPI app to prove it runs identically outside the local Python venv.

**Core concept — why Docker:** portability and environment consistency. A container
packages the app, its exact dependency versions, and a minimal OS layer into one unit
that runs identically on a laptop, in Cloud Build, and on Cloud Run — instead of relying
on whatever Python/packages happen to be installed on a given machine.

**The four core building blocks:**
- **Dockerfile** — a set of instructions defining how to build an image.
- **Image** — the built package/blueprint of the app, produced by `docker build`.
- **Container** — a running instance of an image, produced by `docker run`. One image
  can back many running containers simultaneously (this is exactly how Cloud Run
  autoscaling works later: one built image, multiple container instances spun up as
  traffic increases, scaled back to zero when idle).
- **Registry** (Artifact Registry, on GCP) — stores images. A registry holds multiple
  *repositories*; each repository holds multiple *image* builds, each identified by a
  *tag* (version label, e.g. `v1` or a commit hash).

**Layers & caching:** each Dockerfile instruction (`FROM`, `COPY`, `RUN`, ...) creates
one image layer. Docker caches layers, so a rebuild reuses every layer up to the first
one that actually changed. This is why instruction **order** matters — in
[Dockerfile](../Dockerfile), `requirements.txt` is copied and `pip install` run
*before* the app code is copied, so editing Python code alone doesn't force a
dependency reinstall on the next build.

**Dockerfile details specific to Cloud Run:**
- `FROM python:3.12-slim` — minimal official base image with Python already installed.
- `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8080}"]` —
  two things that matter for containers specifically:
  - `--host 0.0.0.0`: must listen on all interfaces, not `localhost` — inside a
    container, `localhost` only refers to the container itself.
  - `--port ${PORT:-8080}`: Cloud Run injects a `PORT` env var at runtime and expects
    the app to listen on it, so the app can't hardcode a port. The shell form
    (`sh -c "..."`) is required here instead of the plain exec-array form, since env
    var expansion (`${PORT:-8080}`) needs a shell to evaluate it.

**`.dockerignore`:** same idea as `.gitignore`, but for the **build context** — the set
of files sent to the Docker daemon on `docker build .`. Excludes `.venv/`,
`__pycache__/`, `docs/`, etc. so the build stays fast and the image stays lean.

**Commands used:**
```bash
docker build -t gcp-cr-lab:v1 .                              # build image from Dockerfile
docker run -d -p 8080:8080 --name gcp-cr-lab-test gcp-cr-lab:v1   # run it as a container
docker ps                                                     # list running containers
docker logs gcp-cr-lab-test                                   # view container's stdout/stderr
curl http://localhost:8080/ ; curl http://localhost:8080/health   # verify it responds
```
`-p 8080:8080` maps `HOST_PORT:CONTAINER_PORT`. `-d` runs detached (background).

**Result:** both `/` and `/health` responded identically to the local (non-Docker) run —
confirming the containerized app behaves the same as it did under the local venv.

**Status:** Image builds and runs locally. Not yet pushed to Artifact Registry, not yet
deployed to Cloud Run. Dockerfile/.dockerignore being committed via a feature branch
(`feature/add-dockerfile`) rather than directly to `main`.

---

<!-- Add new dated entries below this line as we progress through the project. -->
