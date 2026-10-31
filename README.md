# wids-backend

Central service of [WNDA System](https://github.com/wnda-system). It receives data from the capture device, runs the detection pipeline, stores results and serves the dashboard API.

> 🚧 **Status:** project scaffolding in progress.

## Stack

| Concern | Choice |
|---|---|
| Language | Go |
| HTTP | Gin + [oapi-codegen](https://github.com/oapi-codegen/oapi-codegen) (strict server) |
| API contract | OpenAPI 3.0 (`api/openapi.yaml`) — the single source of truth |
| Database | PostgreSQL 18, [sqlc](https://sqlc.dev/) + pgx, migrations with [goose](https://github.com/pressly/goose) |
| Background jobs | [River](https://riverqueue.com/) (Postgres-backed queue) |
| Object storage | MinIO (S3-compatible) via presigned URLs |
| Config / logs | Environment variables ([caarlos0/env](https://github.com/caarlos0/env)), structured logs with `log/slog` |
| Errors | [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) Problem Details |

## How it works

```
capture device ──► serve ──► PostgreSQL ◄── worker ──► wids-ml-service
                                │   │           │
                                │   └─ NOTIFY ──┘ (new alert)
dashboard ◄── REST + SSE ── serve
```

- **`serve`** accepts requests and answers quickly. It never waits for the models.
- **`worker`** picks up jobs from the queue, sends feature sequences to the ML service, records predictions and opens or updates alerts.
- New alerts reach the dashboard instantly through Postgres `LISTEN/NOTIFY` and Server-Sent Events.

## Commands

One binary, several subcommands:

| Command | Purpose |
|---|---|
| `wids serve` | Run the HTTP API |
| `wids worker` | Run the job worker |
| `wids migrate up` | Apply database migrations (app + River) |
| `wids user create` | Create a dashboard account (admin or viewer) |
| `wids device create` | Register a capture device and issue an API key |
| `wids mac lookup` | Find the pseudonymised form of a MAC address (admin) |

## Repository layout

```
cmd/wids/            entry point and subcommands
api/openapi.yaml     API contract
internal/
  handler/           HTTP layer (implements the generated server interface)
  service/           business rules
  repository/        SQL queries and sqlc-generated code
  model/             domain types
  worker/            River jobs
  middleware/        auth, request ID, logging
  client/            ML service and object storage clients
  config/
  gen/               generated code — do not edit by hand
migrations/          SQL migrations
deploy/              docker-compose, Caddyfile, MinIO setup
```

Requests flow `handler → service → repository`. API, database and domain types are kept separate and converted at each boundary.

## API overview

All routes are under `/api/v1`. Full reference at `/api/docs` when the service is running.

| Area | Endpoints | Auth |
|---|---|---|
| Ingest | `POST /ingest/features`, `POST /ingest/heartbeats`, `POST /captures`, `POST /captures/{id}/complete` | Device API key (`X-Device-Key`) |
| Auth | `POST /auth/login`, `POST /auth/logout`, `GET /auth/me` | Session cookie |
| Dashboard | `GET /overview`, `GET /alerts`, `GET /alerts/{id}`, `POST /alerts/{id}/acknowledge`, `GET /alerts/stream` (SSE), `GET /devices`, `GET /timeseries`, `GET /models`, `GET /captures` | Session cookie |
| System | `/healthz`, `/readyz`, `/metrics` | — |

## Getting started

Requires Go, Docker and `make` (on Windows, work inside WSL and clone into the Linux filesystem).

```bash
cp .env.example .env
make deps       # start PostgreSQL, MinIO and a stub ML service in Docker
make migrate    # apply migrations
make dev        # run the API with live reload
```

Other targets: `make gen` (regenerate code from the OpenAPI spec and SQL), `make lint`, `make test`, `make up` (full stack in Docker).

## Contributing

- Branch from `dev`, open a pull request back to `dev`. `main` receives releases only, tagged `vX.Y.Z`.
- Pull request titles follow [Conventional Commits](https://www.conventionalcommits.org/). Pull requests are squash-merged.
- Change the API by editing `api/openapi.yaml` first, then run `make gen`. CI fails if generated code is out of date.
- Never edit an applied migration. Add a new one.
- Never commit `.env`, keys or captured data.

## Team

- Naran Wongvuttisaroj
- Boonyapon Boontub
