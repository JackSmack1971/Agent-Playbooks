---
trigger: glob
description: Comprehensive rules for Biome v2+ linting configuration, covering setup, rule management, troubleshooting, and best practices for JavaScript/TypeScript projects
globs: ["**/biome.json", "**/biome.jsonc", "**/.biomerc.json"]
---

# Biome Linter Rules

## Configuration File Structure

- **ALWAYS** use `biome.json` or `biome.jsonc` as the primary configuration file
- **INCLUDE** the JSON schema reference for IDE autocomplete: `"$schema": "https://biomejs.dev/schemas/1.8.1/schema.json"`
- **STRUCTURE** configuration with top-level `linter`, `formatter`, `javascript`, `typescript`, `json`, `css` sections
- **ENABLE** the linter explicitly: `"linter": { "enabled": true }`

```json
{
  "$schema": "https://biomejs.dev/schemas/1.8.1/schema.json",
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true
    }
  }
}
```

## Rule Categories and Severity Management

- **USE** rule severity levels: `"error"`, `"warn"`, `"info"`, `"off"`
- **UNDERSTAND** that v2 changed default severities - style rules no longer emit errors by default
- **CONFIGURE** rule groups: `"all"`, `"recommended"`, or individual categories
- **AVAILABLE** categories: `a11y`, `complexity`, `correctness`, `nursery`, `performance`, `security`, `style`, `suspicious`

```json
{
  "linter": {
    "rules": {
      "recommended": true,
      "correctness": {
        "noUnusedVariables": "error",
        "noUndeclaredVariables": "error"
      },
      "style": {
        "noParameterAssign": "warn",
        "useNamingConvention": "error"
      },
      "suspicious": {
        "noDebugger": "error",
        "noConsole": "warn"
      }
    }
  }
}
```

## Rule-Specific Configuration

- **CONFIGURE** individual rules with options when needed
- **USE** `level` and `options` for complex rule configurations
- **HANDLE** fix safety with `"fix": "safe" | "unsafe" | "none"`

```json
{
  "linter": {
    "rules": {
      "correctness": {
        "noUnusedImports": {
          "level": "error",
          "fix": "safe"
        }
      },
      "style": {
        "useNamingConvention": {
          "level": "error",
          "options": {
            "conventions": [
              {
                "selector": {
                  "kind": "classMember",
                  "modifiers": ["private"]
                },
                "match": "_(.*)",
                "formats": ["camelCase"]
              }
            ]
          }
        }
      },
      "a11y": {
        "noBlankTarget": {
          "level": "error",
          "options": {
            "allowDomains": ["example.com", "trusted-site.org"]
          }
        }
      }
    }
  }
}
```

## File Inclusion and Exclusion Patterns

- **CONFIGURE** file patterns carefully after v1 to v2 migration
- **USE** `files.include` and `files.ignore` arrays for pattern matching
- **SUPPORT** glob patterns: `**/*.{js,ts,jsx,tsx}`
- **EXCLUDE** common directories: `["**/node_modules/**", "**/dist/**", "**/build/**"]`

```json
{
  "files": {
    "include": ["src/**/*.{js,ts,jsx,tsx}", "tests/**/*.{js,ts}"],
    "ignore": [
      "**/node_modules/**",
      "**/dist/**",
      "**/build/**",
      "**/*.config.js",
      "**/coverage/**"
    ]
  }
}
```

## Language-Specific Configuration

- **ENABLE** linting per language when needed
- **CONFIGURE** JavaScript, TypeScript, JSON, and CSS linting separately
- **HANDLE** different parser options for each language

```json
{
  "javascript": {
    "linter": {
      "enabled": true
    },
    "globals": ["window", "document", "console"]
  },
  "typescript": {
    "linter": {
      "enabled": true
    }
  },
  "json": {
    "linter": {
      "enabled": true
    }
  },
  "css": {
    "linter": {
      "enabled": true
    }
  }
}
```

## Overrides for Specific File Patterns

- **USE** overrides to apply different rules to specific files
- **COMMON** patterns: test files, configuration files, build scripts
- **SPECIFY** include patterns and rule modifications

```json
{
  "overrides": [
    {
      "include": ["**/*.test.{js,ts}", "**/*.spec.{js,ts}"],
      "linter": {
        "rules": {
          "suspicious": {
            "noConsole": "off"
          },
          "complexity": {
            "noExcessiveComplexity": "off"
          }
        }
      }
    },
    {
      "include": ["**/scripts/**"],
      "linter": {
        "rules": {
          "suspicious": {
            "noConsole": "off"
          }
        }
      }
    }
  ]
}
```

## CLI Usage and Integration

- **RUN** linting: `biome lint [files]`
- **FIX** automatically: `biome lint --write`
- **TARGET** specific rules: `biome lint --only=style/useNamingConvention`
- **SKIP** rules: `biome lint --skip=style --skip=suspicious/noExplicitAny`
- **CI** integration: Use `--error-on-warnings` to fail CI on warnings

```bash
# Basic linting
biome lint src/

# Auto-fix issues
biome lint --write src/

# Run specific rule groups
biome lint --only=correctness --only=security

# Skip problematic rules
biome lint --skip=style/noParameterAssign

# CI-friendly output
biome lint --reporter=github src/
```

## Migration from ESLint

- **MIGRATE** configurations: `biome migrate eslint --write`
- **INCLUDE** inspired rules: `biome migrate eslint --include-inspired --include-nursery`
- **REVIEW** migration output as it's best-effort and may not be perfect
- **MANUAL** adjustments needed for complex ESLint configurations

```bash
# Migrate from ESLint
biome migrate eslint --write

# Include additional rule mappings
biome migrate eslint --include-inspired --include-nursery --write
```

## v1 to v2 Migration

- **RUN** migration command: `biome migrate --write`
- **REVIEW** configuration changes, especially file patterns and rule severities
- **UPDATE** style rule severities if you want them to emit errors
- **CHECK** that critical rules haven't been downgraded to warnings

```json
// v2 Migration - Review these changes
{
  "linter": {
    "rules": {
      "style": {
        "noParameterAssign": "error" // May need explicit error level in v2
      }
    }
  }
}
```

## Performance Optimization

- **LIMIT** rule scope with targeted includes/excludes
- **USE** `--max-diagnostics` to limit output in large codebases
- **PARALLELIZE** with multiple processes on large projects
- **CACHE** results for CI with `--use-server` flag

```json
{
  "files": {
    "include": ["src/**/*.{js,ts,jsx,tsx}"],
    "ignore": ["**/*.d.ts", "**/vendor/**"]
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true
    }
  }
}
```

## Code Suppression Techniques

- **SUPPRESS** single lines: `// biome-ignore lint/suspicious/noExplicitAny: <explanation>`
- **SUPPRESS** entire files: `// biome-ignore-all lint/suspicious/noConsole: <explanation>`
- **SUPPRESS** ranges: `// biome-ignore-start` and `// biome-ignore-end`
- **PROVIDE** explanations for all suppressions

```javascript
// Suppress specific rule for one line
// biome-ignore lint/suspicious/noExplicitAny: Legacy API requires any type
const legacyData: any = getLegacyData();

// Suppress rule for entire file
// biome-ignore-all lint/suspicious/noConsole: Debug logging file

// Suppress range
// biome-ignore-start lint/complexity/noExcessiveComplexity: Generated code
function complexGeneratedFunction() {
  // ... complex logic
}
// biome-ignore-end
```

## Type-Aware Rules (v2+)

- **ENABLE** type-aware rules like `noFloatingPromises` for enhanced analysis
- **CONFIGURE** TypeScript-specific rules in nursery category
- **UNDERSTAND** that type inference doesn't require TypeScript compiler installation

```json
{
  "linter": {
    "rules": {
      "nursery": {
        "noFloatingPromises": "error"
      },
      "correctness": {
        "noUnusedVariables": "error"
      }
    }
  }
}
```

## Known Issues and Mitigations

- **FILE PATTERN ISSUES**: After v2 migration, verify file patterns work correctly with `biome check --verbose`
- **RULE SEVERITY CHANGES**: Style rules may need explicit error levels in v2
- **UNSAFE FIXES**: Review unsafe fixes before applying; configure as safe only when confident
- **CI INTEGRATION**: Use `--error-on-warnings` if warnings should fail CI
- **MEMORY USAGE**: For large codebases, use file includes/excludes to limit scope

## Best Practices

- **START** with recommended rules and gradually customize
- **DOCUMENT** rule suppressions with clear explanations
- **REGULAR** updates to latest Biome version for bug fixes and new rules
- **TEAM** alignment on rule severity levels and suppression policies
- **CI** integration with appropriate error handling and reporting
- **PERFORMANCE** monitoring on large codebases with targeted configuration

## Integration with Development Tools

- **VSCODE**: Use official Biome extension with `"biome.lspBin"` setting
- **NEOVIM**: Configure LSP for formatting and code actions on save
- **WEBPACK/VITE**: Integrate with build process using appropriate plugins
- **GITHUB ACTIONS**: Use `--reporter=github` for CI-friendly output
- **PRE-COMMIT**: Add Biome to pre-commit hooks for automatic checking

```json
// VS Code settings.json
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.biome": "explicit"
  }
}
```