# Spec 007: Analytics Service

## Purpose

Consumes usage events from RabbitMQ, writes to TimescaleDB, and provides analytics API for dashboards.

## Dependencies

- **003-database-schemas**: usage_events hypertable, usage_hourly view
- **004-shared-libraries**: Database, logging, metrics
- **006-api-router-service**: Publishes usage events to queue

## Success Criteria

- [ ] Consumes usage events from RabbitMQ
- [ ] Writes to usage_events hypertable
- [ ] Analytics API returns aggregated data
- [ ] Dashboard queries are fast (<500ms)
- [ ] No message loss (idempotent consumer)

## User Stories

**As an Org Admin:**
- View total tokens consumed this month
- See token breakdown by model
- See token breakdown by user
- View cost estimates

**As a System Admin:**
- View system-wide usage across all orgs
- Identify top consuming organizations

## API Endpoints

```
GET    /api/v1/analytics/orgs/:id/usage       # Org usage (time range)
GET    /api/v1/analytics/orgs/:id/users       # Per-user usage
GET    /api/v1/analytics/orgs/:id/models      # Per-model usage
GET    /api/v1/analytics/system/usage         # System-wide (System Admin only)
GET    /health
GET    /ready
GET    /metrics
```

## Service Components

### 1. Queue Consumer (Background Worker)
```go
// Continuously consume from "usage-events" queue
// Write to usage_events table
// ACK message after successful write
```

### 2. Analytics API (HTTP Server)
```go
// Query usage_hourly continuous aggregate for fast results
// Support filtering by time range, model, user
// Return aggregated statistics
```

## Key Queries

```sql
-- Org usage for last 30 days
SELECT 
    time_bucket('1 day', time) as day,
    SUM(total_tokens) as tokens,
    SUM(cost) as cost
FROM usage_events
WHERE org_id = $1 
  AND time >= NOW() - INTERVAL '30 days'
GROUP BY day
ORDER BY day DESC;

-- Per-user usage (current month)
SELECT 
    user_id,
    SUM(total_tokens) as tokens,
    COUNT(*) as requests,
    SUM(cost) as cost
FROM usage_events
WHERE org_id = $1 
  AND time >= date_trunc('month', NOW())
GROUP BY user_id;
```

## Configuration

```bash
PORT=8080
DATABASE_URL=postgres://...
RABBITMQ_URL=amqp://...
QUEUE_NAME=usage-events
CONSUMER_WORKERS=5
```

## Contracts

See `contracts/` for:
- `api-spec.yaml` - Analytics API specification
- `dashboard-queries.sql` - Optimized query examples

## Next Steps

After completion:
- Implement **008-web-portal** (displays these analytics)

