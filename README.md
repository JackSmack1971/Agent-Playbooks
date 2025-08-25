# Agent-Playbooks

Agent-Playbooks is an open collection of coding and framework rulesets designed for agentic development tools like Roo Code and Windsurf. Each playbook provides language-specific conventions, best practices, and framework guidance to help AI coding agents produce reliable, idiomatic code.

## Rulesets

The `RULESETS/` directory contains standardized technical rulesets for various technologies, frameworks, and tools. Each ruleset follows a consistent structure and naming convention for easy discovery and maintenance.

### Quick Start

- **[📋 Rulesets Index](RULESETS/RULESETS_INDEX.md)** - Browse all available rulesets by category
- **[📝 Contribution Guide](RULESETS/CONTRIBUTING.md)** - How to contribute new rulesets
- **[📋 Template](RULESETS/RULESET_TEMPLATE.md)** - Standard template for new rulesets

### Ruleset Structure

All rulesets follow a standardized format:
- **Front Matter**: YAML metadata with trigger types, description, and file patterns
- **Standardized Sections**: Installation, Configuration, Core Concepts, Security, Performance, etc.
- **Kebab-case Naming**: All files use `kebab-case-ruleset.md` naming convention
- **Schema Validation**: All rulesets conform to the defined JSON schema

### Validation

Run the validation script to ensure rulesets conform to standards:
```bash
python scripts/validate-rulesets.py
```

## Technology Categories

### Backend Frameworks
- **[FastAPI](RULESETS/fastapi-ruleset.md)** - Python web framework
- **[Starlette](RULESETS/starlette-ruleset.md)** - ASGI framework

### AI/ML Frameworks
- **[LangGraph](RULESETS/langgraph-ruleset.md)** - Stateful workflow building
- **[Mem0 AI](RULESETS/mem0-ai-ruleset.md)** - Memory management
- **[Pydantic AI](RULESETS/pydantic-ai-ruleset.md)** - Type-safe AI development

### Databases & Storage
- **[Supabase pgvector](RULESETS/supabase-pgvector-ruleset.md)** - PostgreSQL with vector operations
- **[Storage Backends](RULESETS/storage-backends-ruleset.md)** - Database integration patterns

### DevOps & Infrastructure
- **[Docker Compose](RULESETS/docker-compose-ruleset.md)** - Multi-container orchestration
- **[GitHub Workflows](RULESETS/github-workflows-ruleset.md)** - CI/CD automation

### Frontend Frameworks
- **[Next.js TypeScript](RULESETS/nextjs-typescript-ruleset.md)** - React framework
- **[React 18](RULESETS/react-18-ruleset.md)** - Modern React patterns
- **[Vite](RULESETS/vite-ruleset.md)** - Fast build tool

### Languages & Runtimes
- **[JavaScript ES2023](RULESETS/js-es2023-ruleset.md)** - Modern JavaScript
- **[TypeScript 5.x](RULESETS/typescript-5x-ruleset.md)** - TypeScript features
- **[Python MCP SDK](RULESETS/python-mcp-sdk-ruleset.md)** - Model Context Protocol

### Specialized Tools
- **[Biome Linter](RULESETS/biome-linter-ruleset.md)** - Fast linter and formatter
- **[Docusaurus](RULESETS/docusaurus-ruleset.md)** - Documentation site generator
- **[Loguru](RULESETS/loguru-ruleset.md)** - Python logging
- **[OpenTelemetry](RULESETS/opentelemetry-ruleset.md)** - Observability
- **[Socket.IO](RULESETS/socketio-ruleset.md)** - Real-time communication

## Contributing

We welcome contributions! Please see our [contribution guide](RULESETS/CONTRIBUTING.md) for details on:
- Adding new rulesets
- Updating existing rulesets
- Following naming conventions
- Using the standardized template
- Validation requirements

## Recent Standardization

The rulesets have been recently standardized with:
- ✅ Consistent front-matter schema
- ✅ Kebab-case file naming convention
- ✅ Standardized section structure
- ✅ Automated validation script
- ✅ Comprehensive index for discovery
- ✅ Contribution guidelines and templates

This standardization ensures all rulesets follow the same patterns, making them easier to maintain and discover for both human users and automated tools.
