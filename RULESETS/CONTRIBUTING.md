# Ruleset Contribution Guidelines

This document outlines the standards and processes for contributing to the Agent-Playbooks rulesets.

## File Naming Conventions

- **Use kebab-case**: All ruleset files must use lowercase kebab-case naming
- **Consistent suffix**: All files must end with `-ruleset.md`
- **Examples**:
  - ✅ `fastapi-ruleset.md`
  - ✅ `nextjs-typescript-ruleset.md`
  - ❌ `FastAPI-ruleset.md`
  - ❌ `nextjs_typescript_ruleset.md`

## Front Matter Schema

All rulesets must include valid YAML front matter following this schema:

```yaml
---
trigger: ["glob"]                    # Array of trigger types
description: "Brief description"     # One sentence, 10-200 chars
globs: ["**/*.ext"]                  # Array of file patterns
version: "1.0.0"                     # Semantic version (optional)
last_updated: "2025-01-01"           # ISO date (optional)
---
```

### Trigger Types

- `glob`: File pattern matching
- `model_decision`: AI model decision-based triggering
- `file_extension`: Specific file extension matching
- `content_pattern`: Content-based pattern matching

## Section Structure

Rulesets must follow this standardized section order:

1. **Installation & Setup**
2. **Configuration & Initialization**
3. **Core Concepts / API Usage**
4. **Security & Permissions**
5. **Performance & Scalability**
6. **Error Handling & Troubleshooting**
7. **Testing**
8. **Deployment & Production Patterns**
9. **Known Issues & Mitigations**
10. **Version Compatibility Notes**
11. **References**

## Writing Standards

### Normative Language

Use consistent normative language:
- **MUST**: Requirements that are mandatory
- **ALWAYS**: Actions that are always required
- **SHOULD**: Recommended best practices
- **NEVER**: Prohibited actions

### Code Examples

- **Include language identifiers**: Use proper syntax highlighting
- **Show correct AND incorrect patterns**: Demonstrate both good and bad practices
- **Keep examples concise**: Focus on the specific pattern being illustrated
- **Test all code examples**: Ensure examples are syntactically correct

### Section Organization

- Use `##` for main sections, `###` for subsections
- Include descriptive headers that clearly indicate content scope
- Group related concepts under appropriate main sections
- Use bullet points for rules and recommendations

## Validation

Before submitting, validate your ruleset:

1. **Schema validation**: Front matter must conform to `ruleset-schema.json`
2. **File naming**: Must follow kebab-case with `-ruleset.md` suffix
3. **Section completeness**: All required sections should be present
4. **Content quality**: Clear, actionable guidance with examples

## Pull Request Process

1. **Fork and branch**: Create a feature branch for your changes
2. **Follow naming**: Use descriptive branch names (e.g., `add-react-ruleset`)
3. **Single responsibility**: Each PR should address one ruleset or improvement
4. **Testing**: Run validation scripts and test examples
5. **Documentation**: Update this guide if adding new patterns

## Maintenance

- **Regular updates**: Review and update rulesets for new versions
- **Issue tracking**: Document known issues with mitigation strategies
- **Community feedback**: Incorporate user feedback and real-world experience
- **Deprecation**: Clearly mark deprecated patterns and provide migration guidance

## Tools and Automation

- Use `scripts/validate-rulesets.py` for automated validation
- Run schema validation before commits
- Consider adding CI/CD checks for ruleset compliance
- Use the provided template as a starting point for new rulesets