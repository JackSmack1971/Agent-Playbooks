# Rulesets Index

This index provides a quick reference to all available rulesets in the Agent-Playbooks repository.

## Quick Reference

| Ruleset | Triggers | File Patterns | Description |
|---------|----------|---------------|-------------|
| [FastAPI](fastapi-ruleset.md) | `glob` | `**/*.py`, `**/requirements.txt`, `**/Dockerfile`, `**/docker-compose.yml`, `**/pyproject.toml` | Comprehensive FastAPI development covering setup, security, performance, testing, and deployment |
| [LangGraph](langgraph-ruleset.md) | `glob`, `model_decision` | `*.py`, `main.py`, `workflow.py`, `graph.py`, `agent.py` | Stateful workflow building with memory management, checkpointing, and production patterns |
| [Supabase pgvector](supabase-pgvector-ruleset.md) | `glob`, `model_decision` | `**/*.sql`, `**/*.js`, `**/*.ts`, `**/*.py`, `**/.env*`, `**/supabase/**/*`, `**/package.json` | Database setup, vector operations, authentication, and performance optimization |

## Technology Categories

### Backend Frameworks
- **[FastAPI](fastapi-ruleset.md)**: Python web framework with automatic API documentation
- **[Starlette](starlette-ruleset.md)**: ASGI framework for building async web services

### AI/ML Frameworks
- **[LangGraph](langgraph-ruleset.md)**: Building stateful, persistent workflows with robust memory
- **[Mem0 AI](mem0-ai-ruleset.md)**: Memory management and context retention for AI applications
- **[Pydantic AI](pydantic-ai-ruleset.md)**: Type-safe AI agent development

### Databases & Storage
- **[Supabase pgvector](supabase-pgvector-ruleset.md)**: PostgreSQL with vector operations and authentication
- **[Storage Backends](storage-backends-ruleset.md)**: File storage, object storage, and CDN best practices

### DevOps & Infrastructure
- **[Docker Compose](docker-compose-ruleset.md)**: Multi-container application orchestration
- **[GitHub Workflows](github-workflows-ruleset.md)**: CI/CD pipeline automation and best practices

### Frontend Frameworks
- **[Next.js TypeScript](nextjs-typescript-ruleset.md)**: React framework with TypeScript integration
- **[React 18](react-18-ruleset.md)**: Modern React development patterns and hooks
- **[Vite](vite-ruleset.md)**: Fast build tool and development server

### Languages & Runtimes
- **[JavaScript ES2023](js-es2023-ruleset.md)**: Modern JavaScript features and best practices
- **[TypeScript 5.x](typescript-5x-ruleset.md)**: TypeScript language features and configuration
- **[Python MCP SDK](python-mcp-sdk-ruleset.md)**: Model Context Protocol implementation

### Specialized Tools
- **[Biome Linter](biome-linter-ruleset.md)**: Fast linter and formatter for web projects
- **[Docusaurus](docusaurus-ruleset.md)**: Documentation site generation
- **[Loguru](loguru-ruleset.md)**: Python logging with structured output
- **[OpenTelemetry](opentelemetry-ruleset.md)**: Observability and tracing
- **[Postman](postman-ruleset.md)**: API testing and documentation
- **[SlowAPI Middleware](slowapi-middleware-ruleset.md)**: Rate limiting and API throttling

## Finding the Right Ruleset

### By File Type
- **Python files**: FastAPI, LangGraph, Python MCP SDK, Loguru, Pydantic AI
- **JavaScript/TypeScript**: Next.js, React, Vite, JavaScript ES2023, TypeScript 5.x
- **Configuration files**: Docker Compose, GitHub Workflows, Storage Backends
- **Database files**: Supabase pgvector, Alembic SQLAlchemy
- **Documentation**: Docusaurus, Postman

### By Use Case
- **Web APIs**: FastAPI, Starlette
- **AI Applications**: LangGraph, Mem0 AI, Pydantic AI
- **Vector Databases**: Supabase pgvector
- **DevOps**: Docker Compose, GitHub Workflows
- **Frontend**: Next.js, React, Vite
- **Observability**: OpenTelemetry, Loguru

## Contributing

To add a new ruleset or update existing ones:

1. Follow the [contribution guidelines](CONTRIBUTING.md)
2. Use the [ruleset template](RULESET_TEMPLATE.md) as a starting point
3. Ensure front matter follows the [schema](ruleset-schema.json)
4. Update this index with your new ruleset information

## Validation

All rulesets are validated against:
- **Schema compliance**: Front matter must match `ruleset-schema.json`
- **Naming conventions**: Kebab-case with `-ruleset.md` suffix
- **Content standards**: Consistent structure and normative language

Run validation with:
```bash
python scripts/validate-rulesets.py