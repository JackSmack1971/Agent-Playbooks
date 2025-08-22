---
trigger: glob
description: Comprehensive rules for Postman API testing, collection organization, Newman CLI usage, and automation best practices
globs: ["*.postman_collection.json", "*.postman_environment.json", "**/postman/**", "**/newman/**"]
---

# Postman API Testing Rules

## Collection Organization and Structure

- **Use semantic naming**: Name collections, folders, and requests with descriptive, consistent naming conventions (e.g., "User Management API v2", "POST Create User Account")
- **Organize hierarchically**: Group related requests into folders, use subfolders for complex APIs, and maintain logical request ordering within folders
- **Include comprehensive descriptions**: Add descriptions to collections, folders, and individual requests explaining their purpose, expected behavior, and any special considerations
- **Set collection-level variables**: Define base URLs, API versions, and common parameters as collection variables using `{{variableName}}` syntax
- **Use environment separation**: Create separate environments for development, staging, and production with appropriate variable values
- **Version control collections**: Export collections as JSON files and store in version control, using meaningful commit messages for changes

## Request Configuration Best Practices

- **Use HTTPS protocols**: Always prefer HTTPS over HTTP for API calls to ensure encrypted communication
- **Configure proper headers**: Set appropriate `Content-Type`, `Accept`, and `User-Agent` headers for each request type
- **Implement authentication consistently**: Use collection-level or folder-level authentication when possible, choose appropriate auth types (Bearer Token, Basic Auth, OAuth 2.0, API Key)
- **Parameterize dynamic values**: Use variables for IDs, timestamps, and other dynamic values instead of hardcoding them in request URLs or bodies
- **Handle file uploads correctly**: Use `formdata` body type for file uploads, ensure files are in the working directory when using Newman
- **Set appropriate timeouts**: Configure reasonable timeout values for requests to prevent hanging in automated runs

## Test Scripts and Assertions

- **Write comprehensive test assertions**: Use `pm.test()` with descriptive names and `pm.expect()` for readable assertions
```javascript
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

pm.test("Response time is acceptable", function () {
    pm.expect(pm.response.responseTime).to.be.lte(1000);
});
```

- **Validate response structure**: Use JSON schema validation for complex response structures
```javascript
const schema = {
    type: "object",
    required: ["id", "name"],
    properties: {
        id: { type: "number" },
        name: { type: "string" }
    }
};

pm.test("Response schema is valid", function() {
    pm.response.to.have.jsonSchema(schema);
});
```

- **Extract and store response data**: Capture data from responses for use in subsequent requests
```javascript
pm.test("Extract user ID", function () {
    const responseJson = pm.response.json();
    pm.environment.set("userId", responseJson.id);
});
```

- **Handle error scenarios**: Write tests for both success and failure cases, validate error message formats
- **Use pre-request scripts**: Set up test data, generate timestamps, or perform calculations before requests
- **Avoid hardcoded test data**: Use variables and data files for test inputs to improve maintainability

## Security and Authentication

- **Secure sensitive data**: Store API keys, passwords, and tokens in environment variables, never hardcode in collections
- **Use initial and current values**: Set sensitive data only in "current value" field in environments, keep "initial value" empty for security
- **Implement proper token refresh**: Write scripts to automatically refresh expired authentication tokens
- **Test authentication scenarios**: Include tests for valid authentication, expired tokens, and unauthorized access attempts
- **Validate HTTPS usage**: Ensure all requests use HTTPS, test for proper SSL certificate validation
- **Check for security headers**: Assert presence of security headers like `Strict-Transport-Security`, `X-Content-Type-Options`, `X-Frame-Options`
- **Test CORS configuration**: Verify proper CORS headers and policies are implemented correctly

## Newman CLI and Automation

- **Use specific Newman commands**: Structure Newman runs with clear parameters
```bash
newman run collection.json \
  --environment environment.json \
  --globals globals.json \
  --reporters cli,html,json \
  --reporter-html-export report.html \
  --reporter-json-export report.json \
  --timeout-request 10000 \
  --delay-request 500
```

- **Configure CI/CD integration**: Set up Newman in build pipelines with proper error handling
```bash
# Exit on first failure for fast feedback
newman run collection.json -e environment.json --bail

# Generate detailed reports for analysis
newman run collection.json -e environment.json \
  --reporters cli,html \
  --reporter-html-export newman-report.html
```

- **Use environment files**: Create separate environment files for different deployment stages
- **Handle exit codes**: Check Newman exit codes in CI/CD scripts (0 = success, 1 = failure)
- **Set up data-driven testing**: Use CSV or JSON data files for parameterized test runs
```bash
newman run collection.json \
  --iteration-data test-data.csv \
  --iteration-count 10
```

- **Configure proxy settings**: Use environment variables for proxy configuration when needed
```bash
export HTTP_PROXY=http://proxy.company.com:8080
export HTTPS_PROXY=http://proxy.company.com:8080
```

## Performance and Monitoring

- **Set performance assertions**: Include response time validations in test scripts
```javascript
pm.test("Response time is under 2 seconds", function () {
    pm.expect(pm.response.responseTime).to.be.below(2000);
});
```

- **Monitor API health**: Use Postman Monitors for continuous API health checks in production environments
- **Test under load**: Use collection runner or Newman with iteration data to simulate multiple requests
- **Validate response sizes**: Assert reasonable response payload sizes to catch data bloat
- **Track API changes**: Monitor for unexpected changes in response structure or performance
- **Use request delays**: Add delays between requests in automated runs to avoid overwhelming APIs
```bash
newman run collection.json --delay-request 1000  # 1 second delay
```

## Error Handling and Troubleshooting

- **Implement comprehensive error checking**: Test for various HTTP status codes and error conditions
```javascript
pm.test("Handle error responses gracefully", function () {
    if (pm.response.code >= 400) {
        const errorResponse = pm.response.json();
        pm.expect(errorResponse).to.have.property('error');
        pm.expect(errorResponse.error).to.have.property('message');
    }
});
```

- **Use detailed logging**: Add console.log statements for debugging complex flows
- **Validate error message formats**: Ensure error responses follow consistent structure
- **Handle network timeouts**: Set appropriate timeout values and test timeout scenarios
- **Debug failed requests**: Use Newman's verbose output for troubleshooting
```bash
newman run collection.json --verbose
```

- **Check environment variable resolution**: Verify all variables are properly resolved before running tests
- **Test authentication failures**: Include scenarios for expired tokens, invalid credentials, and permission errors

## Data Management and Environments

- **Use environment hierarchies**: Structure environments with global, collection, and environment variables in proper precedence
- **Implement data cleanup**: Include teardown requests to clean up test data after runs
- **Validate test data**: Ensure test data is valid and follows API requirements
- **Use meaningful variable names**: Choose descriptive names for variables that indicate their purpose
- **Document variable usage**: Include comments or descriptions explaining variable purposes
- **Handle dynamic data**: Generate unique identifiers, timestamps, and random values as needed
```javascript
// Generate unique email for testing
const timestamp = Date.now();
pm.environment.set("testEmail", `test.user.${timestamp}@example.com`);
```

## Collection Maintenance and Best Practices

- **Regular collection reviews**: Audit collections monthly to remove obsolete tests and update outdated scenarios
- **Use collection templates**: Create reusable templates for common API patterns and testing scenarios
- **Document API changes**: Update collections promptly when API contracts change
- **Implement test isolation**: Ensure tests can run independently without relying on specific execution order
- **Use semantic versioning**: Version collection exports using semantic versioning principles
- **Create comprehensive documentation**: Generate and maintain API documentation from collections
- **Share collections effectively**: Use Postman workspaces for team collaboration and collection sharing
- **Export regularly**: Backup collections and environments by exporting JSON files to version control

## Known Issues and Mitigations

- **Variable resolution failures**: Ensure all referenced variables are defined in appropriate scope (global, collection, environment)
- **Authentication token expiry**: Implement token refresh logic in pre-request scripts for long-running test suites
- **CORS issues in browser**: Use Postman desktop app or Newman CLI for requests that fail due to CORS in browser environments
- **File upload limitations**: Ensure files for upload are in the correct directory when using Newman CLI
- **Memory issues with large responses**: Use streaming or pagination for APIs returning large datasets
- **Flaky network tests**: Implement retry logic and appropriate timeouts for unreliable network conditions
- **Environment variable conflicts**: Use specific naming conventions to avoid variable name collisions between different scopes
- **SSL certificate errors**: Configure Newman to ignore SSL errors only in development environments using `--insecure` flag

## Integration and Collaboration

- **API-first development**: Design and test APIs using Postman before implementation begins
- **Mock server usage**: Create mock servers for frontend development while backend APIs are under development
- **Team workspace management**: Organize team workspaces with proper access controls and shared resources
- **Documentation automation**: Generate live API documentation from collections and keep it synchronized with code
- **CI/CD pipeline integration**: Include API tests as mandatory gates in deployment pipelines
- **Performance monitoring**: Set up continuous monitoring for critical API endpoints in production
- **Test data management**: Coordinate test data setup and cleanup across team members and environments