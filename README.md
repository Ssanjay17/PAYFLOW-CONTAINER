# PayFlow — Containerized Architecture (Podman)

Original monolithic FastAPI backend + static frontend split into **7 separate
containers**, each with its own `Containerfile`, orchestrated by
`compose.yaml`.

```
                         ┌─────────────────────┐
   Browser  ───────────► │  gateway (nginx)     │  :8080  (only exposed port)
                         └─────────┬────────────┘
                    ┌──────────────┼───────────────┬─────────────┐
                    ▼              ▼                ▼             ▼
           frontend-index   frontend-dashboard  frontend-admin  /api /ml
             (login/nav)      (customer UI)     (admin + AI/ML     │
              :8081              :8082          dashboard) :8083   │
                                                                    ▼
                                                          backend-api :8000
                                                          (auth, accounts,
                                                        payments, tx, kyc...)
                                                            │         │
                                                 ┌──────────┘         └────────┐
                                                 ▼                             ▼
                                          db (MySQL) :3306          ml-service :8100
                                          + redis :6379          (fraud RandomForest +
                                                                   IsolationForest)
```

## Services / Containers

| Container            | Folder                 | Purpose                                              |
|-----------------------|------------------------|-------------------------------------------------------|
| `gateway`             | `gateway/`             | nginx reverse proxy — the **only** port exposed to the browser (8080) |
| `frontend-index`      | `frontend-index/`      | Login/landing page + nav-bar; also serves shared `css/style.css`, `js/config.js`, `js/api.js` |
| `frontend-dashboard`  | `frontend-dashboard/`  | Customer dashboard UI (`dashboard.html` + `js/app.js`) |
| `frontend-admin`       | `frontend-admin/`      | Admin panel, including the **AI/ML fraud dashboard tab** (`admin.html` + `js/admin.js` + Chart.js vendor lib) |
| `backend-api`         | `backend/`             | Core FastAPI service: auth, users, accounts, beneficiaries, payments, transactions, notifications, KYC, admin |
| `ml-service`          | `ml-service/`          | Standalone AI/ML microservice — RandomForest + IsolationForest fraud scoring, exposed via `/predict` |
| `db`                  | `db/`                  | MySQL 8 database |
| `redis` (image only)  | —                       | Cache/session store (official `redis:7-alpine` image) |

**Why this split:** `ml-service` used to be a `joblib`-loaded module living
inside the backend process. It's now its own container with its own
`Containerfile`, its own `requirements.txt` (sklearn/pandas/joblib live only
here now — the backend image is lighter), and its own REST API
(`POST /predict`). The backend still owns the DB queries needed to build the
fraud feature vector (txn history, averages) since that's core-service data,
then calls `ml-service` over HTTP — that's the correct microservice boundary.

The UI is split the same way you split ShopEase: one container per page
(`index`, `dashboard`, `admin`), each with its own nav/sidebar and its own
Containerfile, navigated between via plain `<a href="…">` / `location.href`
(already how the original frontend worked) and stitched together by the
gateway.

## Build & run — via `make`

A `Makefile` at the project root wraps every `podman-compose` command you
need. Defaults to `podman-compose`; pass `COMPOSE=docker-compose` to use
plain Docker instead.

```bash
cd payflow-containerized

make help        # list every available target with a description
make up           # build + start all 7 containers, foreground, logs streaming
```

Then open **http://localhost:8080** — that's the single entry point.
- `/` → login page
- `/dashboard.html` → customer dashboard (after login)
- `/admin.html` → admin panel + AI/ML fraud dashboard (after admin login)
- `/api/...` → backend REST API (also see `/docs` for Swagger UI)
- `/ml/predict` → direct access to the fraud model, if you want to test it standalone

### Whole-stack targets

| Command | What it does |
|---|---|
| `make build` | Build all 7 containers, don't start them |
| `make up` | Build (if needed) + start all containers, **foreground** |
| `make up-d` | Build + start all containers, **detached** |
| `make down` | Stop and remove all containers |
| `make restart` | `down` then `up-d` |
| `make logs` | Tail logs from every container |
| `make ps` | Show status of all containers |
| `make rebuild` | Full rebuild with `--no-cache`, then start detached |
| `make clean` | Stop everything **and** wipe volumes (drops MySQL data) |
| `make prune-volumes` | `down -v` — remove named volumes only |

### Per-container targets

Swap `<svc>` for `gateway`, `backend`, `ml`, `index`, `dashboard`, `admin`, or `db`.

| Command | What it does |
|---|---|
| `make build-<svc>` | Build just that one container |
| `make up-<svc>` | Rebuild + restart just that one container, detached |
| `make logs-<svc>` | Tail logs for just that one container |

Examples:
```bash
make build-ml        # rebuild just the AI/ML fraud service image
make up-backend       # rebuild + restart just backend-api
make logs-admin       # tail frontend-admin logs
```

### Using Docker instead of Podman

```bash
make COMPOSE=docker-compose up
```

## First-time setup: create an admin account

The register endpoint always creates `role=customer` accounts by design.
To get into `/admin.html` you need to bootstrap an admin/operator account
directly against the running `backend-api` container:

```bash
make admin
```

This runs `create_admin.py` inside the `backend-api` container with the
default credentials `admin@payflow.com` / `StrongPass123`. Edit the `admin:`
target in the `Makefile` (or run the underlying `podman exec` command
manually) to use your own email/password/name:

```bash
podman exec -it payflow-backend-api python create_admin.py create \
  --email admin@payflow.com --password "StrongPass123" --name "Ops Admin"
```

## Notes

- `ml-service` trains its fraud model **at image build time** (baked into
  the image), so there's no first-request cold-start delay — it's ready to
  serve `/predict` the moment the container starts.
- If `ml-service` is ever unreachable, `backend-api` fails safe: it flags
  the transaction for manual `review` rather than silently letting it
  through un-scored (see `backend/app/services/fraud_service.py`).
- All 7 services communicate over the internal `payflow-net` bridge network
  defined in `compose.yaml`; only `gateway` (8080) and `db` (3306, for your
  own DB tooling) are published to the host.
- Rebuild a single service after editing it: e.g.
  `podman-compose -f compose.yaml build ml-service && podman-compose -f compose.yaml up -d ml-service`
