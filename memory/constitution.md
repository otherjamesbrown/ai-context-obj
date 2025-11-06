# Inference-as-a-Service Constitutional Principles

**Version**: 1.1  
**Last Updated**: 2025-11-06

## Project Vision

Build a centralized, multi-tenant AI inference platform that provides a unified API for accessing self-hosted open-source models (Llama, Mistral), with complete cost control, usage tracking, and self-service management for all development teams.

## Core Architectural Mandates

### 1. API-First Design
- **All functionality MUST be exposed via REST API first**
- Web UI and CLI are "dumb clients" that consume the API
- No business logic in the UI layer
- All endpoints must be documented with OpenAPI specifications
- **GitOps for declarative management**: Org Admins can manage user limits, budgets, and configurations via Git (see GitOps Management section)

### 2. Microservices Architecture
- **Backend MUST be composed of independent, containerized microservices**
- Each service has a single, well-defined responsibility
- Services communicate via REST APIs or message queues
- No shared databases between services (except the single PostgreSQL instance)
- Services must be independently deployable

### 3. Stateless Services
- **All services MUST be stateless**
- All persistent state externalized to PostgreSQL
- All cache state externalized to Redis
- All message queues externalized to RabbitMQ
- Services can be horizontally scaled without coordination

### 4. Asynchronous Non-Critical Paths
- **Critical path (inference API) must be ultra-low latency**
- Non-essential operations (usage logging, analytics) MUST be asynchronous
- Use RabbitMQ to decouple critical from non-critical operations
- Never block API responses waiting for analytics writes
- Message queue consumers must be idempotent

### 5. Security by Default
- **All access must be authenticated**
- All actions must be authorized via RBAC
- API keys MUST be hashed (SHA-256) in database
- Plain-text keys shown only once at creation
- All passwords MUST be bcrypt hashed
- Network policies enforce service-to-service communication rules

### 6. Declarative Infrastructure
- All infrastructure (K8s cluster, databases) managed via Terraform
- All application deployments managed via Helm charts
- GitOps-based deployment via ArgoCD
- No manual kubectl apply or terraform apply in production
- Infrastructure as Code is the source of truth

## Technology Stack (Non-Negotiable)

### Backend Services
- **Language**: Go (Golang) only for all microservices
- **Version**: Go 1.21 or later
- **Framework**: Standard library + chi/gorilla for routing (no heavy frameworks)
- **Build**: Multi-stage Docker builds for minimal images

### Databases & Infrastructure
- **State Database**: PostgreSQL 15+ (Linode Managed Database)
- **Analytics**: TimescaleDB extension on PostgreSQL (time-series hypertables)
- **Cache**: Redis 7+ (deployed in K8s via Helm)
- **Message Queue**: RabbitMQ 3.12+ (deployed in K8s via Helm)
- **Orchestration**: Kubernetes (Linode LKE with CPU and GPU node pools)

### Inference
- **Inference Engine**: vLLM (latest stable)
- **Deployment**: Helm charts on GPU node pool
- **API Compatibility**: 100% OpenAI-compatible (/v1/chat/completions)

### Frontend
- **Framework**: React 18+ with TypeScript
- **Build**: Vite
- **UI Library**: TailwindCSS + shadcn/ui components
- **State Management**: React Query + Context API
- **Deployment**: Static site served via Nginx container

### CLI Tool
- **Language**: Go
- **Framework**: Cobra CLI
- **Distribution**: Single binary, released via GitHub Releases

### Infrastructure as Code
- **Cloud Infrastructure**: Terraform (LKE cluster, node pools, managed PostgreSQL)
- **Application Infrastructure**: Helm charts (all services, RabbitMQ, Redis, vLLM)
- **GitOps**: ArgoCD for continuous deployment

### CI/CD
- **CI**: GitHub Actions (lint, test, build, push images)
- **Artifacts**: GitHub Container Registry (ghcr.io)
- **CD**: ArgoCD watches Helm charts in Git repository

### Observability (LGTM Stack)
- **Logs**: Grafana Loki (structured JSON logs from all services)
- **Metrics**: Prometheus + Grafana Mimir (long-term storage)
- **Traces**: OpenTelemetry → Grafana Tempo
- **Dashboards**: Grafana

## API Standards

### Inference API Compatibility
- **MUST be 100% OpenAI-compatible** for /v1/chat/completions endpoint
- Clients should be able to switch from OpenAI to our service by changing the base URL
- Support streaming responses (SSE format)
- Return proper OpenAI-format error responses

### Management API Standards
- All APIs return JSON
- Use standard HTTP status codes (200, 201, 400, 401, 403, 404, 429, 500)
- Use RFC 7807 Problem Details for errors
- All timestamps in ISO 8601 format (UTC)
- Use UUIDs for all resource IDs
- Support pagination for list endpoints (cursor-based preferred)

### Health & Observability
- All services MUST expose `/health` endpoint (liveness probe)
- All services MUST expose `/ready` endpoint (readiness probe)
- All services MUST expose `/metrics` endpoint (Prometheus format)
- Structured logging to stdout/stderr (JSON format)
- OpenTelemetry tracing for all requests

## Development Requirements

### Local Development
- Developers can run the entire stack locally using docker-compose
- No Kubernetes required for local development
- Mock inference service for fast iteration (no GPU needed locally)
- Database migrations managed via golang-migrate
- Makefile provides common tasks (test, lint, build, run)

### Code Quality
- All Go code MUST pass golangci-lint
- All services MUST have unit tests (>80% coverage for business logic)
- All services MUST have integration tests (using Testcontainers for real dependencies)
- E2E tests required for all user-facing features
- No mocks for database tests (use Testcontainers)

### Git Workflow
- Trunk-based development (main branch only)
- Short-lived feature branches (< 2 days)
- All PRs require CI to pass (lint, test, build)
- All PRs require at least one code review
- Squash-merge to main

## RBAC Model (Fixed Roles in V1)

The system has exactly 3 roles in V1. Custom roles cannot be created in V1.

**Forward Compatibility**: The permission system is designed to support custom roles in future versions. The database schema and authorization middleware use a permission-based model internally, with V1 roles implemented as predefined permission sets.

### System Admin (Global Scope)
- Manages the entire platform
- Creates/manages all organizations
- Manages model mappings (add/remove vLLM models)
- Views system-wide usage and analytics
- **Cannot** create API keys (not a developer role)

### Org Admin (Organization Scope)
- Manages their own organization
- Invites/removes users from their org
- Sets/adjusts org budgets (token limits)
- Views org-wide usage analytics
- Creates/revokes their own API keys
- Views/revokes all API keys in their org

### Org User (Organization Scope)
- Developer/team member role
- Creates/revokes their own API keys
- Views their own usage analytics
- Views org-wide usage analytics
- **Cannot** manage other users or budgets

## GitOps Management (Hybrid Mode)

### Declarative Organization Configuration
- Org Admins can manage their organization via YAML files in Git repository
- GitOps reconciler syncs desired state from Git to platform every 30 seconds
- Provides audit trail, versioning, and rollback capabilities for org configurations

### Git-Managed Configuration
The following are managed declaratively via Git (reconciler enforces):
- **User budgets and token limits** (per-user monthly token caps)
- **Rate limits** (requests per second, per user or per org)
- **Allowed models** (subset of available models the org can access)
- **User membership and roles** (who belongs to org, their role)
- **Usage alert thresholds** (notifications at 80%, 90% budget)
- **Org metadata** (description, cost center, tags)

### API-Managed Configuration
The following remain API-only (secrets and runtime data):
- **API keys** (secrets should never be in Git)
- **Usage analytics and logs** (runtime data)
- **Audit logs** (immutable history)

### Reconciliation Strategy
- **Hybrid mode**: Git is source of truth for policy, API manages secrets/runtime
- GitOps reconciler runs every 30 seconds or webhook-triggered
- API changes to Git-managed fields are allowed but reconciler will restore Git state
- Drift is logged in audit log for visibility
- Emergency override: System Admin can disable reconciliation per org

### Git Repository Structure
```yaml
# orgs/<org-name>/org.yaml
apiVersion: ias.linode.com/v1
kind: Organization
metadata:
  name: engineering-team
spec:
  budgets:
    monthly_tokens: 10000000
  users:
    - email: jane@example.com
      role: org-admin
      monthly_token_limit: 5000000
    - email: bob@example.com
      role: org-user
      monthly_token_limit: 1000000
  allowed_models:
    - llama3-8b
    - mistral-7b
  alert_thresholds:
    - percent: 80
    - percent: 90
```

## Security Requirements

### Authentication
- Web Portal: JWT tokens (1-hour access token, 7-day refresh token)
- API Calls: Bearer token (API key in Authorization header)
- Passwords: bcrypt hashing with cost factor 12
- API Keys: SHA-256 hashing before storage

### Authorization
- All API endpoints MUST check authentication
- All actions MUST be authorized via RBAC middleware
- Row-level security: users can only access their org's data
- Audit logging for all write operations

### Network Security
- Kubernetes Network Policies enforce service isolation
- Inference engines only accept traffic from API Router Service
- Databases only accept traffic from specific services
- No direct external access to databases or message queues

## Testing Philosophy

### Test Pyramid (No Mocks for Infrastructure)

1. **Unit Tests** (Pure Business Logic)
   - Fast, no I/O
   - Test validation, parsing, calculations
   - 80%+ coverage required

2. **Integration Tests** (Real Dependencies)
   - Use Testcontainers for PostgreSQL, Redis, RabbitMQ
   - Test against real database engines
   - Validate SQL queries, cache behavior, message publishing
   - No mocks for infrastructure

3. **E2E Tests** (Full System)
   - Happy path MUST run in CI pipeline
   - Full regression suite runs nightly
   - Test against real deployed services
   - Validate complete user workflows

### E2E Test Cases (Minimum Required)
- **Happy Path**: Create org → Create user → Generate key → Call inference API → Verify usage logged
- **Auth Failure**: Invalid/revoked key returns 401
- **Budget Failure**: Exceeding budget returns 429
- **RBAC Failure**: Org User cannot revoke another user's key
- **Audit Log**: All write operations appear in audit log

## CLI-Only Features (V1)

The following features are available via CLI but NOT in the Web UI for V1:

### Password Reset
- **System Admin only** can reset user passwords via CLI
- Command: `ias-admin users reset-password --email=user@example.com --new-password=<secure>`
- Password must meet complexity requirements (12+ chars, mixed case, numbers, symbols)
- Audit logged with admin identity and timestamp
- Web UI password reset flow deferred to V2

## Out of Scope (V1.0)

The following are explicitly deferred to V2 or later:

- ❌ SSO/SAML/OIDC authentication
- ❌ External API routing (OpenAI, Anthropic, Claude)
- ❌ Custom role creation (V2 - schema is forward-compatible)
- ❌ Automated billing/invoicing
- ❌ Model fine-tuning or training
- ❌ Email sending (SMTP integration)
- ❌ Web UI password reset flow (CLI available for System Admins)
- ❌ User email verification
- ❌ Multi-factor authentication (MFA)
- ❌ Rate limiting per user (only per-org budget enforcement)
- ❌ GraphQL API
- ❌ Webhooks for usage alerts

## Model Management

### Dynamic Model Registry
- System Admins can add/remove models via API or CLI
- Models stored in `model_mappings` table in PostgreSQL
- Each model has: name, backend_url, cost_per_1k_tokens, enabled flag
- API Router discovers models from database (cached in Redis)
- Hot-reload: new models available within 30 seconds (cache TTL)

### vLLM Deployment
- Each model deployed as separate K8s Service (e.g., llama-3-8b.inference.svc.cluster.local)
- vLLM Helm chart values stored in Git repository
- System Admin registers model: model_name="llama3", backend_url="http://llama-3-8b.inference:8000"
- API Router validates model name in request, routes to corresponding vLLM service

## Performance Requirements

### Latency Targets
- P50 inference API latency: < 100ms (time to first token)
- P95 inference API latency: < 300ms
- P99 management API latency: < 500ms

### Throughput Targets
- API Router: 1,000 requests/second per instance
- User Service: 500 requests/second per instance
- Analytics Service: 100 requests/second per instance

### Scale Targets (V1 Realistic)
- Support: **1,000 organizations**
- Support: **100 users per organization** (100,000 total users)
- Support: **10 API keys per user** (1,000,000 total API keys)
- Support: 1 million inference requests per day
- No hard limits enforced; system designed to scale horizontally beyond these targets

## Deployment Strategy

### Environments
- **Local**: docker-compose on developer machine
- **Dev**: Namespace in LKE cluster (shared, unstable)
- **Staging**: Namespace in LKE cluster (pre-production testing)
- **Production**: Namespace in LKE cluster (customer-facing)

### Deployment Process
1. Developer pushes to feature branch
2. GitHub Actions runs CI (lint, test, build)
3. On merge to main, GitHub Actions builds and pushes Docker image
4. GitHub Actions updates Helm chart values (image tag)
5. ArgoCD detects change and syncs to cluster
6. Rolling deployment with zero downtime

### Rollback Strategy
- ArgoCD can rollback to previous version
- Database migrations MUST be backward-compatible
- Feature flags for risky changes (defer to V2)

## Cost Tracking

### Token Accounting
- Input tokens and output tokens tracked separately
- Token counts from vLLM response headers
- Cost calculated: (input_tokens × input_rate + output_tokens × output_rate) / 1000
- Rates stored in `model_mappings` table

### Budget Enforcement
- Org budget stored as total monthly tokens
- Current usage cached in Redis for fast checks
- API Router checks budget before routing request
- If budget exceeded: return 429 Too Many Requests
- Budget resets on first day of each month (automated job)

## Bootstrap & Initial Setup

### First-Time Setup Sequence
1. Terraform provisions LKE cluster and managed PostgreSQL
2. Helm installs infrastructure (RabbitMQ, Redis, Prometheus, Grafana)
3. Database migrations run (creates all tables)
4. System Admin runs CLI bootstrap command:
   ```bash
   ias-admin bootstrap create-super-admin \
     --email=admin@linode.com \
     --password=<secure-password>
   ```
5. System Admin creates first organization via CLI or Web UI
6. System Admin deploys first vLLM model and registers it
7. Org Admin invited, creates API key, makes first inference call

### Seed Data (Optional)
- Test organization: "demo-team"
- Test models: "mock-gpt" (for testing without GPU)

## Documentation Requirements

### Required Documentation
- README.md: Project overview, getting started
- CONTRIBUTING.md: How to contribute, coding standards
- docs/api/: OpenAPI specs for all services
- docs/deployment/: How to deploy to production
- docs/development/: Local development setup
- docs/architecture/: System architecture diagrams

### Code Documentation
- All public functions/methods MUST have godoc comments
- All complex algorithms MUST have inline comments
- All API endpoints MUST have OpenAPI documentation
- All database tables MUST be documented in schema files

## Monitoring & Alerts (V1 Basics)

### Required Dashboards
- System Overview: Total requests, error rate, P95 latency
- Organization Usage: Tokens consumed per org, top users
- Model Performance: Requests per model, average latency per model
- Infrastructure Health: Pod status, resource utilization

### Required Alerts (V2 - for now just dashboards)
- API error rate > 5%
- P95 latency > 1 second
- Database connection pool exhausted
- RabbitMQ queue depth > 10,000 messages
- vLLM service down

## Compliance & Legal (Out of Scope for V1)

The following are acknowledged but deferred:
- GDPR compliance (data export, right to deletion)
- SOC 2 compliance
- HIPAA compliance
- Data residency requirements
- Terms of Service enforcement
- Usage limits based on contracts

---

## Guiding Principles Summary

When implementing any feature, ask yourself:

1. ✅ Is it API-first? (Can it be done via API without the UI?)
2. ✅ Is it stateless? (Can I scale this service horizontally?)
3. ✅ Is the critical path fast? (Am I blocking on async operations?)
4. ✅ Is it secure? (Authentication, authorization, audit logging?)
5. ✅ Is it observable? (Logs, metrics, traces?)
6. ✅ Is it tested? (Unit, integration, E2E?)
7. ✅ Is it declarative? (IaC, GitOps, no manual changes?)

If the answer to any of these is "no," reconsider the approach.

---

**This constitution is the foundation. All specs, plans, and implementations MUST adhere to these principles.**

