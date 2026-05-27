# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

`github.com/cto-up/lcgo-lib` is a Go **library** for LLM prompt management and text generation, built on
[`github.com/tmc/langchaingo`](https://github.com/tmc/langchaingo). It provides:

- A multi-tenant **prompt store + REST API** (CRUD, template formatting, and execution with optional SSE streaming).
- **`gochains`** — a thin, builder-style abstraction over langchaingo `LLMChain` for plain-text and structured (JSON) output.
- A **multi-provider LLM factory** (OpenAI, GoogleAI/Gemini, Mistral, Anthropic, Ollama).
- A custom **pgvector vector store** for Postgres with tenant/role(ACL)/tag filtering on similarity search.

It is consumed by an application that wires the handlers into a Gin server. There is **no standalone REST
server `main` in this repo** — `cmd/prompt` is a small CLI that exercises the example chains.

## Before you start any task

1. This module depends on `ctoup.com/coreapp` via a `replace` directive pointing at `../core-be-lib`.
   Auth, tenancy, request middleware, progress events (`event.ProgressEvent`), DB connection helpers, and
   `util`/`helpers` all come from coreapp. If a change is needed there, edit it in `../core-be-lib`.
2. The frontend TypeScript client is generated into the sibling repo `../lcgo-fe-lib` (see `make openapi`).
3. Never modify the `vendor/` folder directly.

## Build & Development Commands

```bash
# Code generation (run after changing specs/queries)
make openapi             # Generate Go server + TS axios client from OpenAPI specs
make sqlc                # Generate type-safe Go code from SQL queries (runs in pkg/core/db)

# Build
make build-prompt                  # Build the example CLI -> ./prompt
go build -o prompt ./cmd/prompt    # Equivalent

# Infrastructure (expects docker/docker-compose-postgresql.yml, not checked in here)
make postgresup          # Start PostgreSQL + pgvector via Docker Compose
make postgresdown        # Stop PostgreSQL

# Tests
go test ./...                              # Run all tests
go test -v ./pkg/core/service/...          # Run tests for a specific package
go test -v -run TestFunctionName ./...     # Run a single test

# Release
make release VERSION=v1.0.0 NOTES="Description"   # gh release create
```

**Toolchain prerequisites:**

- `brew install sqlc` (or see sqlc.dev) — for `make sqlc`
- `npm install -g openapi-typescript-codegen` — provides the `openapi` CLI used by `make openapi`
- `oapi-codegen` (`go install github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@latest`) — Go server/types generation

**Logging:** `cmd/prompt` writes rotating logs and requires the `LOG_FOLDER` env var to be set (and the
directory to exist). Configuration is read from `.env` / `.env.local` via `godotenv`.

**Tests:** Tests load env from `.env` / `.env.local`. Pure unit tests (e.g. `TestExtractJSONFromResponse`)
run offline; the chain/generation tests are integration tests that call real LLM providers and therefore
need the relevant API keys (`OPENAI_API_KEY`, `GOOGLEAI_API_KEY`, `ANTHROPIC_API_KEY`, `MISTRAL_API_KEY`,
`OLLAMA_SERVER_URL`) and, for prompt-store tests, a reachable Postgres. There is **no Testcontainers setup**
in this repo.

## Architecture

### Layered Structure (contract-first)

1. **OpenAPI spec** (`pkg/core/api/openapi/`) — define endpoints here first; `core-api.yaml` + `core-schema.yaml` reference modular `parts/`
2. **Generated code** (`api/openapi/core/`) — `oapi-codegen` output (`core-service.go`, `core-schema.go`), DO NOT EDIT
3. **Handlers** (`pkg/core/api/*_handler.go`) — parse request, enforce tenant/role, call service, return response
4. **Services** (`pkg/core/service/`) — business logic, no HTTP context
5. **Repository** (`pkg/core/db/repository/`) — SQLC-generated, DO NOT EDIT
6. **SQL queries** (`pkg/core/db/query/`) — SQLC source files (`sqlc.yaml` in `pkg/core/db/`)
7. **Migrations** (`pkg/core/db/migration/`) — Goose format with `-- +goose Up` / `-- +goose Down`, embedded via `embed.go`

### Key Packages (in this repo)

- `pkg/core/api/` — `PromptHandler`: CRUD on prompts plus `/format` (template substitution only) and `/execute` (run a prompt through an LLM, with SSE streaming when `Accept: text/event-stream`)
- `pkg/core/service/` — `generation_service.go` (`GenerateTextAnswer`, `GenerateStructuredAnswer`, JSON extraction/validation); `prompt_execution_service.go` (`ExecutePrompt` — template formatting with required-param validation)
- `pkg/core/service/gochains/` — `BaseChain` + fluent `ChainBuilder` over langchaingo; `ChainTypeDefault` vs `ChainTypeStructured`; output parsers (`BaseOutputParser`, `RawStringParser`)
- `pkg/core/db/` — `Store` (wraps SQLC `Queries` + pgx pool), migrations, queries
- `pkg/shared/llmmodels/` — `Provider` enum + `NewLLM(provider, model, json)` for OpenAI, GoogleAI/Gemini, Mistral, Anthropic, Ollama (API keys from env)
- `pkg/shared/pgvector/` — custom langchaingo `vectorstores.VectorStore` for Postgres + pgvector, filtering similarity search by `tenant_id`, ACL `roles`, and `tags`
- `internal/example/` — sample chains (`GenerateSimpleAnswer`, `GenerateSkillsAnalysis`) used by `cmd/prompt`

### Provided by coreapp (`../core-be-lib`, imported — not defined here)

- `pkg/shared/auth` — auth provider + context keys (`auth.AUTH_TENANT_ID_KEY`, `auth.AUTH_USER_ID`, …) and role helpers (`auth.IsAdmin`, `auth.IsSuperAdmin`, `auth.IsCustomerAdmin`)
- `pkg/shared/event` — `ProgressEvent` used for streaming
- request/tenant/auth middleware, `helpers` (paging, error responses), `util`, and the Postgres connector

Role hierarchy (defined in coreapp): `SUPER_ADMIN` (global) > `CUSTOMER_ADMIN` (tenant) > `ADMIN` (tenant) > `USER` (tenant).

## Development Conventions

### Workflow for New Endpoints

1. Edit the OpenAPI spec in `pkg/core/api/openapi/` (add a path file under `parts/`, wire it in `core-api.yaml`)
2. Run `make openapi`
3. Implement the generated `ServerInterface` method in a `*_handler.go` file
4. Add service logic in `pkg/core/service/`

### Workflow for Database Changes

1. Create a migration in `pkg/core/db/migration/` (format: `YYYYMMDDHHMMSS_description.sql`, Goose Up/Down)
2. Add/adjust SQL in `pkg/core/db/query/`
3. Run `make sqlc`
4. Never write raw DB code — always use SQLC-generated functions

### Code Patterns

- Import order: stdlib → external → internal (alphabetical)
- Error wrapping: `fmt.Errorf("service.DoThing: %w", err)`
- `context.Context` as first param in public functions
- Tenant isolation: every prompt query filters by the tenant ID read from context
- Role checks (when required) belong in handlers, not services; return 403 on failure
- DB driver: `pgx/v5` only, never `database/sql`

## Repository Layout Note

For full-stack code generation, this repo (`lcgo-be-lib`), the coreapp backend (`core-be-lib`), and the
frontend (`lcgo-fe-lib`) should be sibling directories under the same parent. `make openapi` writes the
generated TypeScript axios client to `../lcgo-fe-lib/lib/openapi/core`.
