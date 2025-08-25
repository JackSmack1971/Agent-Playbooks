---
trigger: model_decision
description: Comprehensive OpenTelemetry rules for implementing observability with traces, metrics, and logs. Covers initialization, instrumentation patterns, configuration, performance optimization, and anti-patterns to avoid. Updated for 2024-2025 specification changes.
---

# OpenTelemetry Implementation Rules

## Recent Updates and Specification Changes (2024-2025)

### OpenTelemetry Specification v1.47.0 Changes
- **Sampling threshold in TraceState**: New standardized field for client-side sampling decisions - update sampling logic to support TraceState threshold
- **MeasurementProcessor support**: Added to metrics SDK for on-the-fly processing before export - use for real-time metric transformations
- **Enhanced security defaults**: Collector now binds to localhost only by default - explicitly configure network interfaces

### JavaScript SDK 2.0 Breaking Changes
- **Node.js requirement**: Minimum version raised to ^18.19.0 from previous versions
- **TypeScript requirement**: Minimum version 5.0.4, compilation target ES2022
- **Removed legacy APIs**: `window.OTEL_*` browser globals removed - use explicit imports
- **Improved tree-shaking**: Requires explicit package imports for optimal bundling

### Collector Updates
- **Enhanced redaction processor**: Improved capabilities for sanitizing sensitive attributes
- **Stricter binding defaults**: Server endpoints bind to localhost only by default
- **Performance improvements**: Better batching and queue management in v0.132.0+

# OpenTelemetry Implementation Rules

## Core Initialization and Setup

### Provider Configuration
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

### Environment Variables Configuration
- **Use standard environment variables**: Leverage `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES`, `OTEL_EXPORTER_OTLP_ENDPOINT`
- **Set appropriate defaults**: Configure `OTEL_TRACES_EXPORTER=otlp`, `OTEL_METRICS_EXPORTER=otlp`, `OTEL_LOGS_EXPORTER=otlp` for production
- **Use collector in production**: Always set `OTEL_EXPORTER_OTLP_ENDPOINT` to point to OpenTelemetry Collector, not directly to backends

## Tracing Best Practices

### Span Creation and Management
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

### Context Propagation
- **Always propagate context**: Ensure trace context flows through async operations, message queues, and service boundaries
- **Use context managers**: Leverage language-specific context propagation mechanisms
- **Handle async operations**: Properly propagate context in async/await, threading, and concurrent operations
- **Validate context headers**: Check for valid traceparent/tracestate headers in HTTP requests

### Anti-patterns to Avoid
- **Don't create spans for every function**: Only instrument meaningful operations (HTTP requests, DB queries, external calls)
- **Avoid excessive span nesting**: Limit span depth to prevent overwhelming trace visualizations
- **Don't log full payloads**: Sanitize or truncate large request/response bodies
- **Never block on span operations**: Tracing should be non-blocking and fail gracefully

## Metrics Implementation

### Instrument Selection and Creation
- **Choose appropriate instrument types**: Use Counter for monotonic values, UpDownCounter for non-monotonic, Histogram for distributions, Gauge for snapshots
- **Create instruments early**: Initialize all metrics instruments during startup, not during request handling
- **Use descriptive names**: Follow metric naming conventions (e.g., `http.server.request.duration`, `db.client.connections.usage`)
- **Set appropriate units**: Always specify units for instruments (`ms`, `s`, `By`, `1`)

```python
# Good - proper metric instrument creation
meter = metrics.get_meter(__name__, "1.0.0")

request_counter = meter.create_counter(
    name="http.server.requests.total",
    description="Total number of HTTP requests",
    unit="1"
)

request_duration = meter.create_histogram(
    name="http.server.request.duration",
    description="HTTP request duration",
    unit="ms"
)

active_connections = meter.create_up_down_counter(
    name="db.connections.active",
    description="Active database connections",
    unit="1"
)
```

### Recording Measurements
- **Use consistent attributes**: Apply same attribute sets across related metrics for proper correlation
- **Limit attribute cardinality**: Keep unique attribute combinations under 10,000 to prevent cardinality explosion
- **Record measurements at operation completion**: Capture metrics when operations finish, not at start
- **Handle exceptions in measurements**: Don't let metric recording errors crash application logic

### Asynchronous Metrics
- **Use observable instruments for system metrics**: CPU, memory, disk usage should use observable gauges/counters
- **Register callbacks properly**: Ensure callback functions are efficient and non-blocking
- **Handle callback errors**: Observable instrument callbacks should never throw exceptions

## Logging Integration

### Structured Logging
- **Always use structured logs**: Emit JSON or key-value formatted logs, avoid plain text
- **Include trace context**: Ensure log records contain trace_id and span_id for correlation
- **Use appropriate log levels**: DEBUG for verbose info, INFO for normal operations, WARN for recoverable issues, ERROR for failures
- **Sanitize sensitive data**: Remove or mask PII, credentials, and sensitive business data

```python
# Good - structured logging with trace context
import logging
from opentelemetry import trace

# Configure structured logging
logging.basicConfig(
    format='%(asctime)s %(name)s [%(levelname)s] [trace_id=%(otelTraceID)s span_id=%(otelSpanID)s] %(message)s'
)

def process_request(user_id, request_data):
    current_span = trace.get_current_span()
    logger.info(
        "Processing user request", 
        extra={
            "user_id": user_id,
            "request_size": len(request_data),
            "otelTraceID": format(current_span.get_span_context().trace_id, "032x"),
            "otelSpanID": format(current_span.get_span_context().span_id, "016x")
        }
    )
```

### Log-Trace Correlation
- **Enable automatic injection**: Use OpenTelemetry logging integrations to auto-inject trace context
- **Verify correlation setup**: Test that logs contain proper trace_id and span_id fields
- **Configure log processors**: Set up log processors to add resource and instrumentation scope information

## Configuration and Deployment

### OpenTelemetry Collector Usage
- **Always use Collector in production**: Never send telemetry directly from applications to backends
- **Configure batching**: Set appropriate batch_size (512-8192) and timeout (5-30s) values
- **Enable compression**: Use gzip compression for OTLP exports to reduce network overhead
- **Set up retry logic**: Configure exponential backoff for failed exports
- **Resource limits**: Set memory_limiter and configure appropriate resource limits

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

### Security Considerations
- **Use TLS for production**: Always enable TLS/SSL for OTLP exports
- **Filter sensitive attributes**: Configure attribute processors to remove PII and credentials
- **Authenticate exports**: Use API keys or OAuth tokens for backend authentication
- **Network security**: Ensure Collector is properly networked and firewalled

## Performance Optimization

### Sampling Strategies
- **Implement appropriate sampling**: Use probabilistic sampling (1-10%) for high-volume services
- **Configure consistent sampling**: Ensure sampling decisions propagate across service boundaries
- **Use trace-based sampling**: Prefer sampling decisions at trace root rather than individual spans
- **Monitor sampling effectiveness**: Track sampling ratios and adjust based on observability needs

```python
# Good - probabilistic sampler configuration
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased

# Sample 5% of traces
sampler = TraceIdRatioBased(0.05)
tracer_provider = TracerProvider(
    resource=resource,
    sampler=sampler
)
```

### Resource Management
- **Configure export batching**: Use batch span processors with appropriate batch sizes (512-2048)
- **Set timeouts properly**: Configure reasonable export timeouts (5-30 seconds)
- **Limit queue sizes**: Set max_queue_size to prevent memory exhaustion
- **Monitor resource usage**: Track CPU and memory impact of instrumentation

### Anti-patterns for Performance
- **Don't synchronously export**: Never use simple span processors in production
- **Avoid excessive attribute collection**: Limit attributes to essential data only
- **Don't instrument every function**: Focus on service boundaries and critical operations
- **Minimize string operations**: Avoid expensive string formatting in hot paths

## Sampling and Data Management

### Sampling Configuration
- **Use consistent probability sampling**: Implement TraceIdRatioBased or ConsistentProbabilityBased samplers
- **Configure per-service sampling**: Different services may need different sampling rates
- **Implement dynamic sampling**: Consider adjusting sampling based on load or error rates
- **Preserve error traces**: Ensure error traces are always sampled regardless of base sampling rate

### Data Volume Control
- **Set attribute limits**: Configure maximum attributes per span/metric/log record
- **Implement data filtering**: Use processors to filter unnecessary or noisy data
- **Control metric cardinality**: Monitor and limit unique attribute combinations
- **Rotate and archive**: Implement appropriate data retention policies

## Known Issues and Mitigations

### Common Configuration Pitfalls
- **Missing Collector deployment**: Applications sending directly to backends causing reliability issues
- **Incorrect environment variables**: Typos in `OTEL_EXPORTER_OTLP_ENDPOINT` or other config variables
- **Synchronous exports**: Using simple span processors instead of batch processors
- **Missing compression**: Not enabling gzip compression for OTLP exports
- **Excessive instrumentation**: Instrumenting every function instead of focusing on meaningful operations
- **Late SDK initialization**: Initializing OpenTelemetry after importing instrumented libraries causes missing early spans
- **Default Collector bindings**: Using 0.0.0.0 bindings exposes telemetry endpoints to unintended networks

### Collector Anti-Patterns (Critical)
- **Single Collector dependency**: Using one Collector for all telemetry creates single point of failure - deploy agents per application/host with gateway pools
- **No Collector self-monitoring**: Ignoring built-in Collector metrics loses visibility into pipeline health and dropped data
- **Contrib distribution bloat**: Using full Contrib distribution in production - build custom Collector with only required components via OCB
- **Stale Collector versions**: Not updating Collector regularly misses performance improvements and security patches
- **Direct application exports**: Bypassing Collector in production forfeits batching, retries, encryption, and filtering capabilities

### Performance Problems
- **High memory usage**: Unbounded queues in span processors causing memory leaks
- **CPU overhead**: Excessive span creation or attribute computation in hot paths
- **Network congestion**: Too frequent exports or lack of batching
- **Context propagation failures**: Missing context in async operations or message queues
- **Queue configuration**: Not setting OTLP exporter queue size limits (recommend max 800) causes memory exhaustion
- **Missing resource limits**: No memory_limiter processor configuration in Collector deployments

### Debugging and Troubleshooting
- **Enable debug logging**: Set `OTEL_LOG_LEVEL=debug` to diagnose export issues
- **Verify connectivity**: Test OTLP endpoint reachability and authentication
- **Check trace completeness**: Ensure all spans in trace have proper parent-child relationships
- **Monitor export failures**: Track export error rates and retry attempts
- **Validate semantic conventions**: Ensure attributes follow OpenTelemetry semantic conventions
- **Inspect Collector metrics**: Monitor `exporter_queue_push_success` and `processor_batch_send_latency` for bottlenecks
- **Test sampling configurations**: Validate head vs tail sampling in isolated environments before production
- **Version compatibility**: Verify Collector and SDK versions are compatible, especially after major releases

### Language-Specific Gotchas

#### Python
- **Mixed instrumentation**: Don't combine auto and manual instrumentation for same libraries - disable auto-instrumentation for manually instrumented components
- **Stable vs experimental APIs**: Inconsistencies between stable and experimental metrics APIs can break on SDK upgrades
- **Missing tail sampling**: Python SDK lacks built-in tail sampling processor - use Collector for advanced sampling

#### Java  
- **GlobalOpenTelemetry race conditions**: Concurrent initialization with application code can cause issues
- **Configuration property changes**: Version 1.37.0+ changed declarative configuration property names - update `otel.javaagent.configuration-file` settings
- **Resource detector ordering**: Newer releases changed resource detector order - may omit expected host/container metadata

#### JavaScript/Node.js
- **SDK 2.0 breaking changes**: Requires Node.js ^18.19.0, TypeScript 5.0.4+, removes legacy browser globals (window.OTEL_*)
- **Tree-shaking requirements**: New SDK requires explicit package imports for optimal bundling
- **Version constraint mismatches**: Incompatible version constraints cause runtime errors in CI/CD
- **Async context propagation**: Requires careful handling with AsyncLocalStorage for proper context flow

#### Go
- **Context propagation**: Manual context passing required through goroutines - context not automatically inherited
- **Performance overhead**: Inefficient span creation in hot paths can significantly impact performance

#### .NET
- **Activity vs Span APIs**: Prefer Activity API for newer applications - avoid mixing paradigms
- **Framework integration**: Ensure proper integration with ASP.NET Core and Entity Framework instrumentation

## Views and Custom Metrics

### Configuring Metric Views
- **Use views for aggregation control**: Configure histogram buckets, drop unwanted attributes
- **Implement custom aggregations**: Use views to change default aggregations when needed
- **Filter metrics**: Use views to ignore noisy or unnecessary metric streams
- **Rename metrics**: Use views to standardize metric names across different libraries

```python
# Good - metric view configuration
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.view import View

# Custom histogram buckets for latency metrics
latency_view = View(
    instrument_name="http.server.request.duration",
    aggregation=ExplicitBucketHistogramAggregation(
        boundaries=[0.005, 0.01, 0.025, 0.05, 0.075, 0.1, 0.25, 0.5, 0.75, 1.0, 2.5, 5.0, 7.5, 10.0]
    )
)

meter_provider = MeterProvider(
    resource=resource,
    views=[latency_view]
)
```

### Instrumentation Library Integration
- **Use auto-instrumentation when available**: Prefer automatic instrumentation for popular frameworks
- **Extend with manual instrumentation**: Add custom spans/metrics where auto-instrumentation is insufficient
- **Configure instrumentation libraries**: Use environment variables to control instrumentation behavior
- **Keep instrumentation updated**: Regularly update instrumentation packages for bug fixes and new features

## Testing and Validation

### Testing Observability
- **Write tests for instrumentation**: Verify spans, metrics, and logs are created correctly
- **Test context propagation**: Ensure trace context flows through your application correctly
- **Validate attribute values**: Check that semantic conventions are followed correctly
- **Test error scenarios**: Verify proper span status and exception recording

### Production Validation
- **Monitor export health**: Track successful vs failed exports to backends
- **Validate data quality**: Ensure traces are complete and metrics are accurate
- **Check resource attribution**: Verify that telemetry data has proper resource attributes
- **Monitor performance impact**: Continuously measure the overhead of instrumentation

This ruleset reflects OpenTelemetry best practices as of 2024-2025 and addresses the most common implementation challenges and pitfalls encountered in production deployments.