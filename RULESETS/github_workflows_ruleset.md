---
trigger: glob
description: Comprehensive ruleset for GitHub Actions workflows covering syntax, security, performance, and best practices (2025)
globs: [".github/workflows/*.yml", ".github/workflows/*.yaml"]
---

# GitHub Workflows Rules

## Workflow Structure and Syntax

- **Use consistent YAML formatting**: Use 2-space indentation throughout workflow files. Avoid mixing tabs and spaces.
- **Name all workflows explicitly**: Always include a `name:` field at the top level for better identification in the Actions UI.
- **Name all jobs and steps**: Provide descriptive names for jobs and steps to improve readability and debugging.
  ```yaml
  name: CI Pipeline
  jobs:
    test:
      name: Run Tests
      steps:
        - name: Checkout repository
          uses: actions/checkout@v4
  ```
- **Use proper trigger syntax**: Specify explicit event triggers. Avoid bare `on: push` without path filters for large repositories.
  ```yaml
  on:
    push:
      branches: [main, develop]
      paths: ['src/**', 'tests/**']
    pull_request:
      branches: [main]
  ```

## Action Versioning and Security

- **Pin actions to specific versions**: Use major version tags (recommended) or commit SHAs for security. Never use `@main` or `@master`.
  ```yaml
  # Recommended: Major version
  - uses: actions/checkout@v4
  # Alternative: Specific version
  - uses: actions/checkout@v4.1.1
  # Most secure: Commit SHA
  - uses: actions/checkout@8ade135a41bc03ea155e62e844d188df1ea18608
  ```
- **Avoid deprecated action versions**: Replace `@v1`, `@v2`, `@v3` versions of core actions with `@v4` equivalents.
- **Use official actions when possible**: Prefer `actions/*` over third-party alternatives for core functionality (checkout, setup-node, cache, etc.).

## Permissions and Security

- **Set minimal GITHUB_TOKEN permissions**: Explicitly define permissions at workflow or job level using principle of least privilege.
  ```yaml
  permissions:
    contents: read
    pull-requests: write  # Only if needed for PR comments
    actions: read        # Only if needed for artifact access
  ```
- **Never hardcode secrets**: Use GitHub secrets and environment variables. Never put sensitive data directly in workflow files.
  ```yaml
  env:
    API_KEY: ${{ secrets.API_KEY }}
  ```
- **Mask sensitive outputs**: Use `::add-mask::` for dynamic sensitive values that aren't GitHub secrets.
  ```yaml
  - name: Mask dynamic token
    run: echo "::add-mask::$DYNAMIC_TOKEN"
  ```
- **Avoid structured data as secrets**: Don't store JSON, XML, or YAML as secret values. Store individual values separately.
- **Use environment-level secrets**: Leverage environment protection rules for deployment workflows requiring approval.

## Performance Optimization

- **Implement concurrency controls**: Prevent resource waste and race conditions with concurrency groups.
  ```yaml
  concurrency:
    group: ${{ github.workflow }}-${{ github.ref }}
    cancel-in-progress: true
  ```
- **Add job timeouts**: Set reasonable timeout limits to prevent hung workflows from consuming resources.
  ```yaml
  jobs:
    build:
      timeout-minutes: 15  # Adjust based on typical job duration
  ```
- **Use dependency caching**: Cache package managers, build outputs, and tools to reduce execution time.
  ```yaml
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-node-
  ```
- **Implement path filtering**: Use `paths:` and `paths-ignore:` to avoid unnecessary workflow runs.
  ```yaml
  on:
    push:
      paths-ignore: ['docs/**', '*.md', '.gitignore']
  ```

## Matrix Strategy Best Practices

- **Use matrix for multiple configurations**: Test across different versions, OS, or environments efficiently.
  ```yaml
  strategy:
    matrix:
      node-version: [18, 20, 22]
      os: [ubuntu-latest, windows-latest, macos-latest]
    fail-fast: false  # Allow other combinations to complete
  ```
- **Exclude problematic combinations**: Use `exclude:` to skip known incompatible matrix combinations.
- **Set max-parallel limits**: Prevent overwhelming shared resources or external services.
  ```yaml
  strategy:
    max-parallel: 4
    matrix:
      # matrix definition
  ```

## Error Handling and Debugging

- **Use conditional execution wisely**: Employ `if:` conditions for steps that should only run under specific circumstances.
  ```yaml
  - name: Deploy to production
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    run: npm run deploy
  ```
- **Add step output validation**: Check that critical steps completed successfully before proceeding.
  ```yaml
  - name: Build application
    id: build
    run: npm run build
  - name: Verify build output
    run: test -d dist || exit 1
  ```
- **Use workflow commands appropriately**: Leverage `::error::`, `::warning::`, and `::notice::` for better log visibility.
  ```yaml
  - name: Validate configuration
    run: |
      if [ ! -f config.json ]; then
        echo "::error file=config.json::Configuration file missing"
        exit 1
      fi
  ```

## Workflow Organization and Reusability

- **Use composite actions for repeated logic**: Extract common step sequences into `.github/actions/` directories.
- **Implement reusable workflows**: Use `workflow_call` for shared workflow logic across repositories.
  ```yaml
  on:
    workflow_call:
      inputs:
        environment:
          required: true
          type: string
  ```
- **Modularize with scripts**: Move complex logic into versioned scripts rather than embedding in YAML.
  ```yaml
  - name: Run complex deployment
    run: ./scripts/deploy.sh
    env:
      ENVIRONMENT: ${{ inputs.environment }}
  ```

## Environment and Variables

- **Use environment files for multiple variables**: Prefer `$GITHUB_ENV` over multiple `echo` commands.
  ```yaml
  - name: Set multiple environment variables
    run: |
      cat >> $GITHUB_ENV << EOF
      BUILD_DATE=$(date)
      VERSION=${{ github.sha }}
      EOF
  ```
- **Leverage GitHub context appropriately**: Use built-in context variables instead of recreating them.
  ```yaml
  # Good: Use context
  - run: echo "Building commit ${{ github.sha }}"
  # Avoid: Recreating available data
  - run: echo "Building commit $(git rev-parse HEAD)"
  ```

## Known Issues and Mitigations

- **Workflow file limits**: GitHub has a 20MB limit for workflow files. Use external scripts for large workflows.
- **Token scope limitations**: `GITHUB_TOKEN` permissions are limited to the repository. Use PATs with appropriate scopes for cross-repo operations.
- **Windows path issues**: Use forward slashes in paths and `path.join()` in JavaScript actions for cross-platform compatibility.
- **Matrix job failures**: Use `continue-on-error: true` sparingly and only for non-critical matrix combinations.
- **Secret redaction gaps**: Secrets may not be properly masked if modified (base64 encoded, etc.). Avoid transforming secrets in logs.

## Anti-Patterns to Avoid

- **Don't use `continue-on-error: true` globally**: This masks important failures and should only be used for specific non-critical steps.
- **Avoid complex shell logic in YAML**: Move multi-line scripts to separate files for better maintainability and testing.
- **Don't ignore dependency updates**: Regularly update action versions and review security advisories.
- **Avoid workflow recursion**: Be careful with workflows that modify the repository to prevent infinite loops.
- **Don't rely on default branch assumptions**: Always specify branches explicitly rather than assuming `main` or `master`.

## Documentation and Maintenance

- **Comment complex workflow logic**: Use YAML comments to explain non-obvious business logic or requirements.
- **Maintain workflow documentation**: Keep README files describing workflow purposes and dependencies.
- **Use actionlint for validation**: Implement static analysis with tools like `actionlint` in pre-commit hooks or CI.
- **Regular security audits**: Review workflow permissions and third-party action dependencies quarterly.

## Emergency Response

- **Implement workflow disable capability**: Know how to quickly disable workflows via GitHub UI or API during incidents.
- **Have rollback procedures**: Document how to revert workflow changes and manage failed deployments.
- **Monitor workflow logs**: Set up alerting for workflow failures in critical paths like deployments.
- **Secret rotation procedures**: Maintain documented processes for rotating compromised secrets across all environments.