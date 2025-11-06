# Spec 003: Database Schemas & Migrations

**Status**: In Progress  
**Version**: 1.1  
**Last Updated**: 2025-11-06

## Purpose

Define the complete database schema for the entire platform, including state data (users, orgs, keys) and time-series analytics data (usage tracking). Use PostgreSQL with TimescaleDB extension for all data persistence needs.

## Dependencies

- **001-infrastructure**: Requires Linode managed PostgreSQL database provisioned
- **002-local-dev-environment**: Local PostgreSQL container for development

## Success Criteria

- [ ] All database tables defined with proper types, constraints, and indexes
- [ ] TimescaleDB extension configured for time-series usage data
- [ ] Database migrations created using golang-migrate
- [ ] Migrations can run successfully up and down
- [ ] Entity relationship diagram (ERD) documented
- [ ] Sample seed data for development/testing

## User Stories

**As a Developer:**
- I want well-defined database schemas so I know what data structures to use
- I want migrations to run automatically so I don't manually create tables
- I want seed data so I can test features without manual setup

**As a DBA/Platform Engineer:**
- I want migrations versioned in Git so schema changes are tracked
- I want rollback capabilities so I can undo problematic migrations
- I want proper indexes so queries perform well at scale

## Database Technology

### PostgreSQL 15+ (Linode Managed)
- Single database for all data (state + analytics)
- TimescaleDB extension for time-series hypertables
- Connection pooling via PgBouncer (configured in K8s)

### Why TimescaleDB?
- Native PostgreSQL extension (no separate database)
- Optimized for time-series queries (usage analytics)
- Automatic data retention policies
- Continuous aggregates for fast dashboards
- No StatefulSets needed (uses managed PostgreSQL)

## Schema Design

### Core Tables

#### 1. organizations
Represents a team/organization using the platform.

```sql
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(255) NOT NULL,
    budget_tokens_monthly BIGINT NOT NULL DEFAULT 0,
    tokens_used_current_month BIGINT NOT NULL DEFAULT 0,
    budget_reset_day INT NOT NULL DEFAULT 1, -- Day of month (1-28)
    status VARCHAR(50) NOT NULL DEFAULT 'active', -- active, suspended, deleted
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ
);

CREATE INDEX idx_organizations_status ON organizations(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_organizations_name ON organizations(name) WHERE deleted_at IS NULL;
```

#### 2. users
Represents individual users (System Admins, Org Admins, Org Users).

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL, -- system_admin, org_admin, org_user
    status VARCHAR(50) NOT NULL DEFAULT 'active', -- active, suspended, pending, deleted
    monthly_token_limit BIGINT, -- Per-user token limit (NULL = no limit, org limit applies)
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ,
    
    CONSTRAINT users_role_check CHECK (role IN ('system_admin', 'org_admin', 'org_user')),
    CONSTRAINT users_status_check CHECK (status IN ('active', 'suspended', 'pending', 'deleted')),
    CONSTRAINT users_org_check CHECK (
        (role = 'system_admin' AND org_id IS NULL) OR
        (role IN ('org_admin', 'org_user') AND org_id IS NOT NULL)
    )
);

CREATE INDEX idx_users_org_id ON users(org_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_role ON users(role) WHERE deleted_at IS NULL;
```

#### 3. api_keys
API keys for programmatic access to the inference API.

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL, -- User-friendly name like "production-app"
    key_prefix VARCHAR(20) NOT NULL, -- First 8 chars of key (for display)
    key_hash VARCHAR(64) NOT NULL UNIQUE, -- SHA-256 hash of full key
    last_used_at TIMESTAMPTZ,
    last_used_ip VARCHAR(45), -- IPv4 or IPv6
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    revoked_at TIMESTAMPTZ,
    revoked_by UUID REFERENCES users(id) ON DELETE SET NULL,
    
    CONSTRAINT api_keys_name_user_unique UNIQUE(user_id, name, revoked_at)
);

CREATE INDEX idx_api_keys_user_id ON api_keys(user_id) WHERE revoked_at IS NULL;
CREATE INDEX idx_api_keys_org_id ON api_keys(org_id) WHERE revoked_at IS NULL;
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash) WHERE revoked_at IS NULL;
CREATE INDEX idx_api_keys_last_used ON api_keys(last_used_at DESC);
```

#### 4. model_mappings
Registry of available models and their vLLM backend endpoints.

```sql
CREATE TABLE model_mappings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_name VARCHAR(255) NOT NULL UNIQUE, -- e.g., "llama3", "mistral-7b"
    display_name VARCHAR(255) NOT NULL, -- e.g., "Llama 3 8B"
    backend_url VARCHAR(500) NOT NULL, -- e.g., "http://llama-3-8b.inference:8000"
    cost_per_1k_input_tokens DECIMAL(10,6) NOT NULL DEFAULT 0.0,
    cost_per_1k_output_tokens DECIMAL(10,6) NOT NULL DEFAULT 0.0,
    max_tokens INT NOT NULL DEFAULT 4096,
    enabled BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID REFERENCES users(id) ON DELETE SET NULL,
    
    CONSTRAINT model_mappings_costs_check CHECK (
        cost_per_1k_input_tokens >= 0 AND 
        cost_per_1k_output_tokens >= 0
    )
);

CREATE INDEX idx_model_mappings_enabled ON model_mappings(enabled);
CREATE INDEX idx_model_mappings_name ON model_mappings(model_name) WHERE enabled = true;
```

#### 5. audit_log
Audit trail for all management operations.

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    action_type VARCHAR(100) NOT NULL, -- user_created, key_created, budget_changed, etc.
    resource_type VARCHAR(100) NOT NULL, -- user, api_key, organization, model
    resource_id UUID, -- ID of the affected resource
    action_payload JSONB, -- Full request details
    ip_address VARCHAR(45),
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_log_org_id ON audit_log(org_id);
CREATE INDEX idx_audit_log_user_id ON audit_log(user_id);
CREATE INDEX idx_audit_log_created_at ON audit_log(created_at DESC);
CREATE INDEX idx_audit_log_action_type ON audit_log(action_type);
CREATE INDEX idx_audit_log_resource ON audit_log(resource_type, resource_id);
```

### Time-Series Tables (TimescaleDB)

#### 6. usage_events (Hypertable)
Individual inference requests for detailed analytics.

```sql
-- First, enable TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;

CREATE TABLE usage_events (
    time TIMESTAMPTZ NOT NULL,
    event_id UUID NOT NULL,
    org_id UUID NOT NULL,
    user_id UUID NOT NULL,
    api_key_id UUID NOT NULL,
    model_name VARCHAR(255) NOT NULL,
    input_tokens INT NOT NULL,
    output_tokens INT NOT NULL,
    total_tokens INT NOT NULL,
    cost DECIMAL(10,6) NOT NULL,
    latency_ms INT, -- Time to first token
    status VARCHAR(50) NOT NULL, -- success, error, budget_exceeded
    error_message TEXT,
    request_id VARCHAR(100),
    
    CONSTRAINT usage_events_tokens_check CHECK (
        total_tokens = input_tokens + output_tokens AND
        input_tokens >= 0 AND
        output_tokens >= 0
    )
);

-- Convert to hypertable (partitioned by time)
SELECT create_hypertable('usage_events', 'time');

-- Create indexes on hypertable
CREATE INDEX idx_usage_events_org_time ON usage_events(org_id, time DESC);
CREATE INDEX idx_usage_events_user_time ON usage_events(user_id, time DESC);
CREATE INDEX idx_usage_events_model_time ON usage_events(model_name, time DESC);
CREATE INDEX idx_usage_events_status ON usage_events(status, time DESC);

-- Continuous aggregate for hourly stats (fast dashboard queries)
CREATE MATERIALIZED VIEW usage_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS hour,
    org_id,
    model_name,
    COUNT(*) as request_count,
    SUM(input_tokens) as total_input_tokens,
    SUM(output_tokens) as total_output_tokens,
    SUM(total_tokens) as total_tokens,
    SUM(cost) as total_cost,
    AVG(latency_ms) as avg_latency_ms,
    COUNT(*) FILTER (WHERE status = 'error') as error_count
FROM usage_events
GROUP BY hour, org_id, model_name;

-- Refresh policy: update aggregate every hour
SELECT add_continuous_aggregate_policy('usage_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- Retention policy: keep raw events for 90 days
SELECT add_retention_policy('usage_events', INTERVAL '90 days');
```

### Supporting Tables

#### 7. refresh_tokens
For web portal JWT refresh tokens.

```sql
CREATE TABLE refresh_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(64) NOT NULL UNIQUE,
    expires_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    revoked_at TIMESTAMPTZ
);

CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id) WHERE revoked_at IS NULL;
CREATE INDEX idx_refresh_tokens_expires ON refresh_tokens(expires_at) WHERE revoked_at IS NULL;
```

#### 8. invite_tokens
For user invitation links.

```sql
CREATE TABLE invite_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL, -- org_admin, org_user
    token_hash VARCHAR(64) NOT NULL UNIQUE,
    invited_by UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    expires_at TIMESTAMPTZ NOT NULL,
    used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    CONSTRAINT invite_tokens_role_check CHECK (role IN ('org_admin', 'org_user'))
);

CREATE INDEX idx_invite_tokens_token ON invite_tokens(token_hash) WHERE used_at IS NULL;
CREATE INDEX idx_invite_tokens_email ON invite_tokens(email) WHERE used_at IS NULL;
```

#### 9. org_allowed_models (GitOps Support)
Per-organization model access restrictions (managed via GitOps or API).

```sql
CREATE TABLE org_allowed_models (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    model_name VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    managed_by VARCHAR(50) NOT NULL DEFAULT 'api', -- 'api' or 'gitops'
    
    CONSTRAINT org_allowed_models_unique UNIQUE(org_id, model_name)
);

CREATE INDEX idx_org_allowed_models_org ON org_allowed_models(org_id);
CREATE INDEX idx_org_allowed_models_managed_by ON org_allowed_models(managed_by);
```

#### 10. gitops_sync_state (GitOps Reconciliation)
Tracks GitOps reconciliation state per organization.

```sql
CREATE TABLE gitops_sync_state (
    org_id UUID PRIMARY KEY REFERENCES organizations(id) ON DELETE CASCADE,
    enabled BOOLEAN NOT NULL DEFAULT false,
    last_sync_at TIMESTAMPTZ,
    last_sync_commit_sha VARCHAR(64),
    sync_status VARCHAR(50) NOT NULL DEFAULT 'never_synced', -- synced, error, drift_detected
    sync_error_message TEXT,
    config_repo_url VARCHAR(500),
    config_file_path VARCHAR(500), -- e.g., orgs/engineering-team/org.yaml
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_gitops_enabled ON gitops_sync_state(enabled) WHERE enabled = true;
CREATE INDEX idx_gitops_status ON gitops_sync_state(sync_status);
```

### RBAC Tables (Forward-Compatible for V2)

**Note**: V1 uses hard-coded roles (system_admin, org_admin, org_user). These tables support custom roles in V2.

#### 11. permissions (V2 - Foundation Only)
Defines granular permissions in the system.

```sql
CREATE TABLE permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL UNIQUE, -- e.g., 'user:create', 'key:revoke', 'model:add'
    resource_type VARCHAR(50) NOT NULL, -- user, api_key, organization, model, etc.
    action VARCHAR(50) NOT NULL, -- create, read, update, delete
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_permissions_resource ON permissions(resource_type, action);
```

#### 12. role_permissions (V2 - Foundation Only)
Maps roles to permissions (currently only used for system roles).

```sql
CREATE TABLE role_permissions (
    role VARCHAR(50) NOT NULL, -- system_admin, org_admin, org_user (V1), or custom role UUIDs (V2)
    permission_id UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    PRIMARY KEY (role, permission_id)
);

CREATE INDEX idx_role_permissions_role ON role_permissions(role);
```

**V1 Implementation Note**: These tables will be populated with predefined permissions for the 3 fixed roles. The authorization middleware checks `users.role` directly in V1. In V2, custom roles will be added as UUIDs in the `role_permissions.role` column, and the middleware will query this table.

## Migration Strategy

### Migration Tool
Use **golang-migrate** for version control of schema changes.

### Migration Files Location
`/migrations/` directory in repository root.

### Migration Naming Convention
```
000001_initial_schema.up.sql      # Create all tables
000001_initial_schema.down.sql    # Drop all tables
000002_add_model_description.up.sql
000002_add_model_description.down.sql
```

### Example Migration Execution
```bash
migrate -path ./migrations \
        -database "postgres://user:pass@host:5432/ias?sslmode=require" \
        up

migrate -path ./migrations \
        -database "postgres://user:pass@host:5432/ias?sslmode=require" \
        down 1
```

## Seed Data (Development Only)

### Bootstrap Organization
```sql
INSERT INTO organizations (id, name, display_name, budget_tokens_monthly)
VALUES ('550e8400-e29b-41d4-a716-446655440000', 'bootstrap-org', 'Bootstrap Organization', 1000000000);
```

### Bootstrap Super Admin
```sql
-- Password: "changeme" (bcrypt hashed)
INSERT INTO users (id, org_id, email, password_hash, role)
VALUES ('550e8400-e29b-41d4-a716-446655440001', NULL, 'admin@linode.com', 
        '$2a$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5GyYQj.Pr8k3u', 'system_admin');
```

### Test Model
```sql
INSERT INTO model_mappings (model_name, display_name, backend_url, cost_per_1k_input_tokens, cost_per_1k_output_tokens)
VALUES ('mock-gpt', 'Mock GPT (Testing)', 'http://mock-inference:8000', 0.001, 0.002);
```

## Entity Relationship Diagram

```
organizations (1) ──< (N) users
organizations (1) ──< (N) api_keys
organizations (1) ──< (N) usage_events
organizations (1) ──< (N) invite_tokens

users (1) ──< (N) api_keys
users (1) ──< (N) usage_events
users (1) ──< (N) refresh_tokens
users (1) ──< (N) audit_log
users (1) ──< (N) model_mappings (created_by)

api_keys (1) ──< (N) usage_events

(Model mappings are standalone - referenced by name in usage_events)
```

## Contracts

See `contracts/` directory for:
- `migrations/000001_initial_schema.up.sql` - Complete schema creation
- `migrations/000001_initial_schema.down.sql` - Complete schema teardown
- `seed-data.sql` - Development seed data
- `erd-diagram.md` - Detailed entity relationship documentation
- `timescaledb-setup.md` - TimescaleDB configuration guide

## Testing Strategy

### Schema Validation Tests
- Run migrations up/down successfully
- Verify all constraints enforce data integrity
- Test foreign key cascades
- Validate unique constraints

### Data Integrity Tests
- Cannot create org_user without org_id
- Cannot create system_admin with org_id
- API keys must belong to same org as user
- Token counts must be non-negative

### TimescaleDB Tests
- Hypertable creation successful
- Continuous aggregates update correctly
- Retention policy works
- Query performance on large datasets

## Configuration

### Required Environment Variables
- `DATABASE_URL` - PostgreSQL connection string
- `TIMESCALEDB_ENABLED` - Set to "true" to enable TimescaleDB features

### Connection Pooling
- Max connections: 100
- Idle connections: 10
- Connection lifetime: 5 minutes

## Performance Considerations

### Index Strategy
- All foreign keys indexed
- Time-based queries optimized with DESC indexes
- Partial indexes (WHERE clauses) for soft deletes
- Covering indexes for common queries

### Query Optimization
- Use continuous aggregates for dashboards (pre-computed)
- Time-bucket queries on hypertables are very fast
- Avoid full table scans on usage_events (always filter by time)

### Scaling Strategy
- Hypertable automatically partitions by time
- Old partitions can be moved to cheaper storage
- Retention policies automatically drop old data
- Read replicas for analytics queries (future)

## Security Considerations

### Access Control
- Each service has its own database user with minimal privileges
- User Service: Read/write to users, orgs, api_keys, audit_log
- API Router: Read-only to api_keys, model_mappings; Write to usage_events
- Analytics Service: Read-only to usage_events, usage_hourly

### Sensitive Data
- All passwords bcrypt hashed (never plaintext)
- All API keys SHA-256 hashed (never plaintext)
- Personal data (email) can be GDPR-deleted (future)

### SQL Injection Prevention
- Use parameterized queries only (never string concatenation)
- ORM/query builder helps prevent SQL injection

## Monitoring

### Key Metrics to Track
- Database connection pool utilization
- Query latency (P50, P95, P99)
- Table sizes (growth over time)
- Index hit ratio (should be >99%)
- Replication lag (if using replicas)

### Alerts
- Connection pool exhaustion
- Long-running queries (>5s)
- Failed migrations
- Disk space low (<20%)

## Open Questions

### Resolved
- ✅ Use TimescaleDB instead of separate ClickHouse
- ✅ Single managed PostgreSQL for all data
- ✅ Soft deletes vs hard deletes (using soft deletes with deleted_at)

### To Be Resolved
- Should we partition audit_log by time as well? (Probably yes in V2)
- Data retention for audit logs? (Suggest 1 year)
- GDPR compliance: How to handle "right to be forgotten"? (V2)

### Newly Added (v1.1)
- ✅ Added GitOps support tables (org_allowed_models, gitops_sync_state)
- ✅ Added per-user token limits (users.monthly_token_limit)
- ✅ Added forward-compatible RBAC tables (permissions, role_permissions)
- ✅ Scale targets updated to 1,000 orgs, 100K users, 1M keys

## Next Steps

After this spec is complete:
1. Implement **004-shared-libraries** (pkg/database for connection pooling)
2. Implement **005-user-org-service** (uses these tables)
3. Implement **006-api-router-service** (uses api_keys, model_mappings, writes usage_events)
4. Implement **007-analytics-service** (queries usage_events and usage_hourly)

## References

- TimescaleDB Documentation: https://docs.timescaledb.com/
- golang-migrate: https://github.com/golang-migrate/migrate
- PostgreSQL Best Practices: https://wiki.postgresql.org/wiki/Don't_Do_This

