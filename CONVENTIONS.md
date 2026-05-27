## Project Overview

`github.com/cto-up/lcgo` — a Go library for LLM prompt management and text generation.

- **backend**: A Go API library (prompt store, generation services, vector store) consumed by a Gin server.

## Key Technologies

- **Language**: Go
- **HTTP**: Gin
- **Database**: PostgreSQL (with pgvector)
- **LLM**: langchaingo (OpenAI, GoogleAI/Gemini, Mistral, Anthropic, Ollama)
- **Codegen**: oapi-codegen (OpenAPI) and SQLC (SQL)

## Development Workflow & Common Commands

This project uses `make` as the primary task runner in the repository root.

- **Run tests**: `go test ./...`
- **Run linter**: `golangci-lint run` (if installed)
- **Build the example CLI**: `make build-prompt` (or `go build -o prompt ./cmd/prompt`)
- **Regenerate API code**: `make openapi`
- **Regenerate DB code**: `make sqlc`

## Architectural Conventions

Contract-first, layered architecture.

- All API changes must first be defined in the OpenAPI specification located at
  `pkg/core/api/openapi/`.
- Once defined, the API code is generated with `make openapi` (Go server into `api/openapi/core/`,
  TypeScript client into `../lcgo-fe-lib/lib/openapi/core`).
- New database tables are created with Goose migrations in `pkg/core/db/migration/`, and the
  corresponding queries are added to `pkg/core/db/query/`.
- After adding migrations and queries, run `make sqlc` to generate the type-safe database code.
- Handlers live in `pkg/core/api/`.
- Services live in `pkg/core/service/`.
- Database access lives in `pkg/core/db/` (always via SQLC-generated functions; never raw `database/sql`).
