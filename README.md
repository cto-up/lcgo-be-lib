# lcgo (`ctoup.com/lcgo`)

A Go library for **LLM prompt management and text generation**, built on
[langchaingo](https://github.com/tmc/langchaingo). It powers the prompt features of the CtoUp platform
and is consumed by an application that wires its handlers into a Gin server.

## Features

- **Prompt store + REST API** — multi-tenant CRUD over prompts (`core_prompts`), plus endpoints to
  *format* a prompt (template substitution) and *execute* it against an LLM, with optional
  Server-Sent-Events streaming.
- **`gochains`** — a builder-style wrapper around langchaingo `LLMChain` supporting plain-text and
  structured (JSON) output, with response-schema validation.
- **Multi-provider LLM factory** — OpenAI, GoogleAI (Gemini), Mistral, Anthropic, and Ollama, selected
  at runtime; API keys are read from the environment.
- **pgvector vector store** — a custom langchaingo `VectorStore` for PostgreSQL + pgvector with
  similarity search filtered by tenant, ACL roles, and tags.

## Layout

| Path | Purpose |
|------|---------|
| `pkg/core/api/` | `PromptHandler` (REST handlers) + OpenAPI specs in `openapi/` |
| `pkg/core/service/` | Generation and prompt-execution services |
| `pkg/core/service/gochains/` | Chain abstraction (`BaseChain`, `ChainBuilder`) and output parsers |
| `pkg/core/db/` | SQLC-generated repository, Goose migrations, queries |
| `pkg/shared/llmmodels/` | LLM provider factory |
| `pkg/shared/pgvector/` | pgvector vector store |
| `api/openapi/core/` | Generated Go server/types (do not edit) |
| `cmd/prompt/` | Example CLI that runs the sample chains |
| `internal/example/` | Sample chains used by the CLI |

## Dependencies

This module uses `ctoup.com/coreapp` (auth, multi-tenancy, request middleware, progress events, DB helpers)
via a `replace` directive pointing at the sibling repo `../core-be-lib`. The frontend TypeScript client is
generated into `../lcgo-fe-lib`.

## Getting started

```bash
# Configure environment (DB connection, LLM API keys, LOG_FOLDER, BACKEND_PORT)
cp .env .env.local        # then edit .env.local

# Regenerate code after editing specs or SQL
make openapi
make sqlc

# Build the example CLI
make build-prompt

# Run tests (unit tests run offline; integration tests need LLM API keys / Postgres)
go test ./...
```

See [CLAUDE.md](./CLAUDE.md) for architecture and development workflows, and
[CONVENTIONS.md](./CONVENTIONS.md) for conventions.

## Release Management

Automated versioning is handled via the Makefile:

```bash
make release VERSION=v0.0.1 NOTES="Description of changes"
```

## License

Distributed under the **MIT License**.
