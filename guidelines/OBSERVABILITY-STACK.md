# CodeSpline Observability Stack Reference

**Status**: Organization-level reference for observability implementation across CodeSpline projects.

**Purpose**: Provide consistent guidance on metrics, logging, tracing, dashboards, and alerting.

**Scope**: This guide covers the core Prometheus + Grafana stack. Projects may extend with Elasticsearch for logs, Jaeger/Tempo for distributed tracing, etc.

---

## Overview

Observability means understanding system behavior through three signals:

1. **Metrics**: Numerical measurements over time (latency, request count, error rate)
2. **Logs**: Discrete events and messages (API call, error, user action)
3. **Traces**: Request flow through distributed system (frontend → API → database)

**Goal**: Answer questions like:
- Is the API healthy? (metrics)
- What happened at 3:42 PM? (logs)
- Why is request X slow? (traces)

---

## Metrics Collection — Prometheus

### Architecture

```
Applications
    ↓ (scrape)
Prometheus Server (time-series database)
    ↓
Grafana (dashboards & alerting)
```

**How it works**:

1. Applications expose metrics on `/metrics` endpoint
2. Prometheus scrapes endpoint every 15 seconds (configurable)
3. Metrics stored as time-series (metric_name{labels} value timestamp)
4. Grafana queries Prometheus for visualization and alerts

### Instrumentation (NestJS)

Use `@nestjs/metrics` or `prom-client`:

```typescript
import { register, Counter, Histogram, Gauge } from 'prom-client'

// Counter: counts events (increments only)
const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
})

// Histogram: measures durations with buckets
const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request latency',
  labelNames: ['method', 'route'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5],
})

// Gauge: arbitrary value (up/down)
const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Number of active WebSocket connections',
})

// Middleware
app.use((req, res, next) => {
  const start = Date.now()
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000
    httpRequestsTotal.labels(req.method, req.route, res.statusCode).inc()
    httpRequestDuration.labels(req.method, req.route).observe(duration)
  })
  
  next()
})

// Expose metrics
app.get('/metrics', (req, res) => {
  res.set('Content-Type', register.contentType)
  res.end(register.metrics())
})
```

### Key Metrics to Collect

| Metric | Type | Labels | Purpose |
|--------|------|--------|---------|
| `http_requests_total` | Counter | method, route, status | Request volume and errors |
| `http_request_duration_seconds` | Histogram | method, route | API latency (p50, p95, p99) |
| `bullmq_jobs_completed_total` | Counter | queue_name, job_type | Background job success |
| `bullmq_jobs_failed_total` | Counter | queue_name, job_type | Background job failures |
| `bullmq_queue_size` | Gauge | queue_name | Pending jobs in queue |
| `pg_query_duration_seconds` | Histogram | query_type | Database query latency |
| `pg_connections_active` | Gauge | database | Active DB connections |
| `redis_cache_hits_total` | Counter | key_pattern | Cache hit rate |
| `redis_cache_misses_total` | Counter | key_pattern | Cache misses |

### Prometheus Configuration

`prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # NestJS API
  - job_name: 'api'
    static_configs:
      - targets: ['localhost:3000']
    metrics_path: '/metrics'

  # BullMQ worker
  - job_name: 'worker'
    static_configs:
      - targets: ['localhost:3001']
    metrics_path: '/metrics'

  # PostgreSQL exporter
  - job_name: 'postgres'
    static_configs:
      - targets: ['localhost:9187']

  # Redis exporter
  - job_name: 'redis'
    static_configs:
      - targets: ['localhost:9121']

  # Node.js runtime
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
```

---

## Logging — Structured JSON

### Principles

- **Structured**: Log as JSON, not freetext (easier to parse)
- **Consistent**: Same field names across all services
- **Contextual**: Include request ID, user ID, action for tracing
- **No secrets**: Never log passwords, tokens, API keys

### Standard Fields

```json
{
  "timestamp": "2026-07-23T10:15:30.123Z",
  "level": "INFO",
  "service": "api",
  "trace_id": "550e8400-e29b-41d4-a716-446655440000",
  "span_id": "f1234567-89ab-cdef",
  "user_id": "user-123",
  "action": "release_created",
  "resource_type": "release",
  "resource_id": "rel-456",
  "duration_ms": 145,
  "message": "Release created successfully",
  "status": "success"
}
```

### NestJS Logger

```typescript
import { Logger } from '@nestjs/common'

@Injectable()
export class ReleaseService {
  private logger = new Logger(ReleaseService.name)

  async createRelease(dto: CreateReleaseDto, userId: string, traceId: string) {
    const start = Date.now()
    
    try {
      const release = await this.prisma.release.create({ data: dto })
      
      this.logger.log(JSON.stringify({
        timestamp: new Date().toISOString(),
        level: 'INFO',
        service: 'api',
        trace_id: traceId,
        user_id: userId,
        action: 'release_created',
        resource_type: 'release',
        resource_id: release.id,
        duration_ms: Date.now() - start,
        message: 'Release created successfully',
        status: 'success',
      }))
      
      return release
    } catch (error) {
      this.logger.error(JSON.stringify({
        timestamp: new Date().toISOString(),
        level: 'ERROR',
        service: 'api',
        trace_id: traceId,
        user_id: userId,
        action: 'release_created',
        resource_type: 'release',
        duration_ms: Date.now() - start,
        message: 'Failed to create release',
        error: error.message,
        status: 'error',
      }))
      throw error
    }
  }
}
```

### Log Shipping (Future)

Send logs to:

- **Loki** (Grafana's logs backend) for retention and querying
- **ELK Stack** (Elasticsearch, Logstash, Kibana) for larger scale
- **CloudWatch** if on AWS
- **Application Insights** if on Azure

---

## Distributed Tracing

### Trace Context Propagation

**Problem**: Request flows through multiple services; how do you correlate logs and spans?

**Solution**: Propagate `trace-id` through the entire request lifecycle.

### HTTP Headers

Add W3C Trace Context headers to all HTTP requests:

```
traceparent: 00-550e8400e29b41d4a716446655440000-f1234567-89abc123-01
tracestate: vendor-specific=data
```

**Format**: `00-{trace_id}-{span_id}-{trace_flags}`

### Implementation (NestJS Middleware)

```typescript
import { v4 as uuidv4 } from 'uuid'

@Injectable()
export class TraceMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const traceId = req.headers['traceparent'] 
      ? extractTraceId(req.headers['traceparent'])
      : uuidv4()
    
    const spanId = uuidv4().substring(0, 16)
    
    req.traceId = traceId
    req.spanId = spanId
    
    res.setHeader('traceparent', `00-${traceId}-${spanId}-01`)
    
    next()
  }
}

// Pass trace context to child services/HTTP calls
async callExternalAPI(url: string, req: Request) {
  const response = await axios.get(url, {
    headers: {
      'traceparent': `00-${req.traceId}-${req.spanId}-01`,
    },
  })
  return response.data
}
```

### BullMQ Job Context

For background jobs, pass trace context in job metadata:

```typescript
// Queue producer (in API controller)
await this.releaseQueue.add(
  'sync-jira',
  { releaseId: 'rel-123' },
  {
    jobId: `${req.traceId}-${Date.now()}`,
    opts: {
      metadata: {
        trace_id: req.traceId,
        span_id: req.spanId,
        user_id: req.user.id,
      },
    },
  }
)

// Queue consumer (in worker)
this.releaseQueue.process(async (job) => {
  const traceId = job.data.metadata.trace_id
  // All logs from this job now include the same trace_id
})
```

---

## Dashboards — Grafana

### Provisioned Dashboards

Create 4 core dashboards (YAML-as-code):

#### 1. API Health Dashboard

```yaml
- title: API Health
  panels:
    - title: Request Rate
      targets:
        - expr: rate(http_requests_total[5m])
    
    - title: Error Rate
      targets:
        - expr: rate(http_requests_total{status_code=~"5.."}[5m])
    
    - title: Latency (p99)
      targets:
        - expr: histogram_quantile(0.99, http_request_duration_seconds)
    
    - title: Active Connections
      targets:
        - expr: active_connections
```

#### 2. Queue Health Dashboard

```yaml
- title: Queue Health
  panels:
    - title: Jobs Completed
      targets:
        - expr: rate(bullmq_jobs_completed_total[5m])
    
    - title: Jobs Failed
      targets:
        - expr: rate(bullmq_jobs_failed_total[5m])
    
    - title: Queue Depth
      targets:
        - expr: bullmq_queue_size
    
    - title: Job Duration
      targets:
        - expr: histogram_quantile(0.95, bullmq_job_duration_seconds)
```

#### 3. Database Health Dashboard

```yaml
- title: Database Health
  panels:
    - title: Query Duration (p95)
      targets:
        - expr: histogram_quantile(0.95, pg_query_duration_seconds)
    
    - title: Active Connections
      targets:
        - expr: pg_connections_active
    
    - title: Connection Pool Utilization
      targets:
        - expr: pg_connections_active / pg_max_connections
    
    - title: Query Errors
      targets:
        - expr: rate(pg_query_errors_total[5m])
```

#### 4. Infrastructure Dashboard

```yaml
- title: Infrastructure
  panels:
    - title: CPU Usage
      targets:
        - expr: rate(process_cpu_seconds_total[5m])
    
    - title: Memory Usage
      targets:
        - expr: process_resident_memory_bytes / 1024 / 1024
    
    - title: Disk I/O
      targets:
        - expr: rate(node_disk_io_time_seconds_total[5m])
    
    - title: Network I/O
      targets:
        - expr: rate(node_network_io_bytes_total[5m])
```

---

## Alerting

### Alert Rules

Define alert conditions in Prometheus:

```yaml
groups:
  - name: api_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status_code=~"5.."}[5m]) > 0.05
        for: 5m
        annotations:
          summary: "API error rate > 5%"

      - alert: QueueBacklog
        expr: bullmq_queue_size > 500
        for: 10m
        annotations:
          summary: "Job queue backlog > 500 jobs"

      - alert: HighLatency
        expr: histogram_quantile(0.99, http_request_duration_seconds) > 2
        for: 5m
        annotations:
          summary: "API p99 latency > 2 seconds"

      - alert: DBConnectionSaturation
        expr: (pg_connections_active / pg_max_connections) > 0.9
        for: 5m
        annotations:
          summary: "Database connection pool > 90% utilization"

      - alert: JobFailureSpike
        expr: rate(bullmq_jobs_failed_total[5m]) > 0.1
        for: 5m
        annotations:
          summary: "Job failure rate > 10%"
```

### Notification Channels

Route alerts to:

- **Slack**: Immediate notification in #alerts channel
- **PagerDuty**: For critical/SEV1 alerts (on-call escalation)
- **Email**: For non-urgent warnings (daily digest)
- **Webhooks**: Custom integrations (IFTTT, custom apps)

---

## Evolution Path

### Phase 1: Basic Metrics (MVP)

- ✅ Prometheus + Grafana
- ✅ Basic metrics (request rate, latency, errors)
- ✅ Simple dashboards (4 core dashboards above)
- ✅ Email alerts for critical conditions

### Phase 2: Logging & Tracing

- ✅ Structured JSON logging
- ✅ Log shipping to Loki or Elasticsearch
- ✅ Distributed tracing with W3C context propagation
- ✅ Trace visualization in Grafana or Jaeger

### Phase 3: Advanced Observability

- ✅ OpenTelemetry SDK for unified metrics, logs, traces
- ✅ Sampling strategies for high-volume services
- ✅ Correlation between metrics, logs, and traces
- ✅ Automated alerting (anomaly detection)

### Phase 4: ML & Insights

- ✅ Predictive alerting (forecast latency spikes)
- ✅ Automatic root cause analysis
- ✅ Compliance reporting (audit trail)

---

## Using This Guide in Your Project

Reference this document from your project's `docs/architecture/10-observability.md`:

```markdown
## Observability

For complete guidance on metrics, logging, tracing, dashboards, and alerting,
see the organization-level [OBSERVABILITY-STACK.md](../../../OBSERVABILITY-STACK.md).

In our project specifically:
- Prometheus scrape interval: {{YOUR INTERVAL}}
- Alert channels: {{YOUR CHANNELS}}
- Log retention: {{YOUR RETENTION}}
- Trace sampling rate: {{YOUR RATE}}
```

---

**Last Updated**: 2026-07-23
