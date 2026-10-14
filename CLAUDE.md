# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Training repository for the Sages course "AIOps w praktyce" (3-day course on using AI tools for
diagnostics, configuration and incident response). It is **not** a product codebase under active
feature development — it's an exercise environment:

- `app/` — **Kantyna**, a deliberately instrumented microservices sample app (canteen lunch
  ordering) that runs on a shared AWS EKS cluster. Most "bugs" encountered during labs are
  intentionally injected via runtime flags, not real defects — see Fault injection below.
- `labs/` — lab instructions (`lab01`…`lab07`), each with a goal, steps, and a checkable SUKCES
  (success) condition. Work for a given lab happens by following that lab's README, not by
  independently restructuring the app.
- `docs/` — topic write-ups and prompt templates (`docs/prompts/`) referenced by labs.
- `setup/` — workstation setup and `setup/check.sh` (cluster/environment smoke test).

Content and docs are in Polish; code, comments, and commit conventions in `app/` are in English.

## Commands

All commands below are run from `app/` unless noted.

```bash
# local stack
docker compose up -d --build
docker compose ps
docker compose down -v                      # stop + wipe DB volume

# synthetic traffic (optional profile)
docker compose --profile loadgen up -d loadgen
PEAK_RPS=20 docker compose --profile loadgen up -d loadgen
docker compose --profile loadgen stop loadgen

# tests
make test                                   # everything
make test-orders-api                        # pytest via uv
make test-payments                          # go test, in golang:1.23 container
# equivalent without make:
cd orders-api && uv run pytest -q
cd orders-api && uv run pytest -q tests/test_api.py::test_name   # single test
docker run --rm -v "$PWD/payments:/src" -w /src golang:1.23 go test ./...
docker run --rm -v "$PWD/payments:/src" -w /src golang:1.23 go test ./... -run TestName

# Helm chart
make helm-lint
make helm-template REGISTRY=<registry> VERSION=1.0.0
helm template kantyna deploy/helm/kantyna | kubectl apply --dry-run=client -f -

# build/push images (ECR)
make ecr-login
make build push VERSION=1.0.0 [VARIANT=receipt-v2]   # VARIANT also tags :VERSION-VARIANT
```

Repo-root smoke test against the shared training cluster: `setup/check.sh` (expects
`SUKCES: 11/11`).

## Architecture (Kantyna, `app/`)

```
browser -> web (nginx SPA, :8080) -> /api/* -> orders-api (FastAPI, :8000)
                                                   |-> payments (Go, :8080)    [POST /pay]
                                                   |-> PostgreSQL (:5432)      [orders, menu]
                                                   `-> AWS SQS "orders"        [only if QUEUE_URL set]
                                                           -> worker (:9000 metrics) -> AWS S3 (receipts)
loadgen (:9100) -> orders-api   (synthetic traffic, diurnal profile)
```

Order flow (`orders-api/app/main.py`, `POST /api/orders`):
1. Validate order (`employee_id` as string, `items[]` with `menu_item_id` + `qty`).
2. Price from DB, call `payments` `/pay` — a payment rejection aborts before any DB write.
3. Persist order + items in Postgres in one transaction.
4. Publish event to SQS, only when `QUEUE_URL` is set (local compose has no SQS/S3; worker idles).

`worker` long-polls SQS, "prepares" the order, writes a JSON receipt to S3 at
`receipts/YYYY/MM/DD/<order_id>.json`, deletes the message. Failures requeue; after 5 attempts a
message moves to the DLQ.

Every request carries a `trace_id` that appears in logs and OTLP traces, so you can follow one
request web → orders-api → payments/Postgres/SQS. Services: `orders-api` (Python 3.12 + FastAPI,
`uv`-managed), `payments` (Go, stdlib `net/http` + `prometheus/client_golang`), `worker` (Python +
boto3), `web` (static HTML/JS behind nginx, proxies `/api/`), `loadgen` (asyncio/httpx traffic
generator), `postgres` (17-alpine). Full endpoint table and env var reference: `app/README.md`.

### Fault injection (the point of the app)

Each service reads a `FlagStore` (`orders-api/app/flags.py`, `payments/flags.go`,
`worker/app/flags.py`) that merges environment variables with an optional JSON file
(`FLAGS_FILE`, default `/etc/kantyna/flags/flags.json`; locally `deploy/local/flags.json`; in
cluster the `kantyna-flags` ConfigMap). The file is re-read on an interval (~15s) so flags change
without a restart, and file values win over env values.

These flags are how labs simulate incidents — error rates, latency, CPU burn, a slow memory leak,
a DB connection-pool leak, an N+1 menu query, log noise — see `orders-api/app/runtime.py` for the
hooks and `app/deploy/local/flags.json` for the full flag set per service. When debugging "weird"
behavior in this app, check the active flags before assuming a real code defect.

`APP_VARIANT` (`stable` / `receipt-v2` / `menu-v2`) is a second axis of deliberately different
behavior per build — e.g. `receipt-v2` randomly fails receipt rendering, `menu-v2` forces the N+1
query path (`runtime.menu_uses_n_plus_one`).

### Observability

Prometheus scrapes each service's `/metrics`; Loki collects JSON stdout logs
(`ts,level,service,version,msg,route,status,duration_ms,trace_id,span_id`); Tempo collects OTLP
traces from `orders-api`. `PrometheusRule` alerts (`KantynaHighErrorRate`,
`KantynaHighLatencyP95`, `KantynaPaymentsFailing`, `KantynaPodCrashLooping`,
`KantynaPodOOMKilled`, `KantynaQueueBacklog`, `KantynaDBPoolExhausted`) each link to a runbook in
`app/runbooks/` via `runbook_url`. Deploy artifacts live in `app/deploy/helm/kantyna/templates/`.

## Working in this repo

- `app/.env` holds **fake** secrets for exercises — never treat its contents as real, and never
  send real credentials, passwords, or personal data to the model.
- The EKS cluster is shared across all course participants; work only inside your own namespace
  (`kubectl config set-context aiops --namespace <login>`).
- `.claude/settings.json` already denies reading secrets and running `kubectl delete`/`exec`, and
  requires confirmation for `kubectl set|patch|edit|rollout|apply`. Treat any hypothesis or
  generated command as something to verify before running, especially against the shared cluster.
- Lab instructions in `labs/*/README.md` are the source of truth for what a given exercise expects
  you to do (and often explicitly restrict scope, e.g. "only ask questions, don't change code").

## Dodatki (lab01)

- Wersje narzędzi: `kubectl v1.36.3`, `helm v3.22.0+g144ca65`.
- Komenda do sprawdzania zmian w chart Helm: `helm lint app/deploy/helm/kantyna`.
- Zakaz: nie uruchamiaj `kubectl delete` na współdzielonym klastrze.
