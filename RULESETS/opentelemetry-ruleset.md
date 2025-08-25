---
trigger: ["model_decision"]
description: Comprehensive OpenTelemetry rules for implementing observability with traces, metrics, and logs. Covers initialization, instrumentation patterns, configuration, performance optimization, and anti-patterns to avoid.
globs: ["**/*.py", "**/*.js", "**/*.ts", "**/*.java", "**/*.go", "**/*.cs"]
version: "1.47.0"
last_updated: "2025-01-01"
---

# OpenTelemetry Implementation Rules

## Installation & Setup

- **MUST** use compatible versions across SDK and Collector for production stability
- **ALWAYS** initialize providers early during application startup
- **SHOULD** use environment variables for configuration in containerized environments

```bash
# Python
pip install opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp

# Node.js
npm install @opentelemetry/api @opentelemetry/sdk-node @opentelemetry/exporter-otlp-http
```

## Configuration & Initialization

- **Always initialize providers early**: Set up TracerProvider, MeterProvider, and LoggerProvider during application startup, before any business logic
- **Use resource attributes**: Configure Resource with semantic conventions for service.name, service.version, deployment.environment
- **Set instrumentation scope**: Create named tracers/meters with library name and version using `getTracer(name, version)` pattern
- **Configure exporters properly**: Use OTLP exporters for production, console exporters only for development

```python
# Good - proper initialization
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.resources import Resource

resource = Resource.create({
    "service.name": "my-service",
    "service.version": "1.0.0",
    "deployment.environment": "production"
})

trace.set_tracer_provider(TracerProvider(resource=resource))
tracer = trace.get_tracer(__name__, "1.0.0")
```

## Core Concepts / API Usage

- **Follow semantic conventions**: Use official semantic conventions for span names, attributes, and operations
- **Create meaningful span hierarchies**: Establish clear parent-child relationships reflecting actual operation flow
- **Use span.is_recording()**: Check if span is recording before adding expensive attributes or events
- **Properly end spans**: Always ensure spans are ended, use context managers or try-finally blocks
- **Limit span attributes**: Keep attributes under 128KB total, use essential data only

```python
# Good - proper span management
with tracer.start_as_current_span("database_query") as span:
    if span.is_recording():
        span.set_attribute("db.system", "postgresql")
        span.set_attribute("db.statement", sanitized_query)
        span.set_attribute("db.name", database_name)

    try:
        result = execute_query(query)
        span.set_status(Status(StatusCode.OK))
        return result
    except Exception as e:
        span.set_status(Status(StatusCode.ERROR, str(e)))
        span.record_exception(e)
        raise
```

## Security & Permissions

- **Use TLS for production**: Always enable TLS/SSL for OTLP exports
- **Filter sensitive attributes**: Configure attribute processors to remove PII and credentials
- **Authenticate exports**: Use API keys or OAuth tokens for backend authentication
- **Network security**: Ensure Collector is properly networked and firewalled

## Performance & Scalability

- **Implement appropriate sampling**: Use probabilistic sampling (1-10%) for high-volume services
- **Configure export batching**: Use batch span processors with appropriate batch sizes (512-2048)
- **Set timeouts properly**: Configure reasonable export timeouts (5-30 seconds)
- **Limit queue sizes**: Set max_queue_size to prevent memory exhaustion

## Error Handling & Troubleshooting

- **Enable debug logging**: Set `OTEL_LOG_LEVEL=debug` to diagnose export issues
- **Verify connectivity**: Test OTLP endpoint reachability and authentication
- **Check trace completeness**: Ensure all spans in trace have proper parent-child relationships
- **Monitor export failures**: Track export error rates and retry attempts

## Testing

- **Write tests for instrumentation**: Verify spans, metrics, and logs are created correctly
- **Test context propagation**: Ensure trace context flows through your application correctly
- **Validate attribute values**: Check that semantic conventions are followed correctly
- **Test error scenarios**: Verify proper span status and exception recording

## Deployment & Production Patterns

- **Always use Collector in production**: Never send telemetry directly from applications to backends
- **Configure batching**: Set appropriate batch_size (512-8192) and timeout (5-30s) values
- **Enable compression**: Use gzip compression for OTLP exports to reduce network overhead
- **Set up retry logic**: Configure exponential backoff for failed exports

```yaml
# Good - Collector configuration
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    send_batch_size: 1024
    timeout: 5s
  memory_limiter:
    limit_mib: 256

exporters:
  otlp:
    endpoint: https://backend.example.com:443
    compression: gzip
    retry_on_failure:
      enabled: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp]
```

## Known Issues & Mitigations

- **Missing Collector deployment**: Applications sending directly to backends causing reliability issues
- **Incorrect environment variables**: Typos in `OTEL_EXPORTER_OTLP_ENDPOINT` or other config variables
- **Synchronous exports**: Using simple span processors instead of batch processors
- **Late SDK initialization**: Initializing OpenTelemetry after importing instrumented libraries causes missing early spans
- **Default Collector bindings**: Using 0.0.0.0 bindings exposes telemetry endpoints to unintended networks

## Version Compatibility Notes

- **Current version tested**: OpenTelemetry Specification v1.47.0
- **Breaking changes**: Major API changes between versions - test thoroughly when upgrading
- **Language-specific requirements**: Check minimum version requirements for each language SDK
- **Collector compatibility**: Ensure Collector and SDK versions are compatible

## References

- [OpenTelemetry Specification](https://opentelemetry.io/docs/specs/otel/)
- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [Language SDK Documentation](https://opentelemetry.io/docs/languages/)