---
trigger: ["glob"]
description: Brief one-sentence description of the technology and its ruleset coverage
globs: ["**/*.ext", "**/pattern/**/*"]
version: "1.0.0"
last_updated: "2025-01-01"
---

# Technology Name Rules

## Installation & Setup

- **MUST** use proper installation method with specific version pinning
- **ALWAYS** configure development environment with isolated dependencies
- **SHOULD** use official documentation for initial setup

```bash
# Installation example
pip install technology==1.0.0
```

```python
# Configuration example
config = {
    "key": "value"
}
```

## Configuration & Initialization

- **MUST** validate configuration parameters on startup
- **ALWAYS** use environment variables for sensitive data
- **SHOULD** provide sensible defaults for optional settings

## Core Concepts / API Usage

- **MUST** understand fundamental concepts and patterns
- **ALWAYS** follow established conventions and best practices
- **SHOULD** leverage built-in features before custom implementations

## Security & Permissions

- **MUST** implement proper authentication and authorization
- **ALWAYS** validate and sanitize user inputs
- **SHOULD** follow principle of least privilege

## Performance & Scalability

- **MUST** monitor and optimize resource usage
- **ALWAYS** implement caching for expensive operations
- **SHOULD** design for horizontal scaling when applicable

## Error Handling & Troubleshooting

- **MUST** implement comprehensive error handling
- **ALWAYS** log errors with appropriate context
- **SHOULD** provide meaningful error messages to users

## Testing

- **MUST** write automated tests for critical functionality
- **ALWAYS** test both success and failure scenarios
- **SHOULD** achieve high test coverage for production code

## Deployment & Production Patterns

- **MUST** use configuration management for different environments
- **ALWAYS** implement health checks and monitoring
- **SHOULD** follow infrastructure as code principles

## Known Issues & Mitigations

- **MUST** document workarounds for known limitations
- **ALWAYS** provide mitigation strategies for common problems
- **SHOULD** reference relevant issue trackers or documentation

## Version Compatibility Notes

- **MUST** specify tested versions and compatibility requirements
- **ALWAYS** document breaking changes and migration paths
- **SHOULD** provide version-specific configuration guidance

## References

- [Official Documentation](https://example.com)
- [API Reference](https://api.example.com)
- [Community Resources](https://community.example.com)