# Spec 004: Shared Go Libraries (pkg/)

## Purpose

Create reusable Go packages for common functionality across all microservices (authentication, RBAC, logging, metrics, database connection, configuration).

## Dependencies

- **000-project-setup**: Go workspace configured
- **003-database-schemas**: Database schema defined

## Success Criteria

- [ ] All common functionality extracted to pkg/
- [ ] Services can import and use shared packages
- [ ] Unit tests for all shared libraries (>90% coverage)
- [ ] Well-documented public APIs (godoc)
- [ ] No circular dependencies between packages

## Package Structure

```
pkg/
├── auth/              # JWT and API key authentication
│   ├── jwt.go         # JWT token generation/validation
│   ├── apikey.go      # API key hashing/validation
│   └── middleware.go  # HTTP middleware for auth
│
├── rbac/              # Role-based access control
│   ├── roles.go       # Role definitions
│   ├── permissions.go # Permission checks
│   └── middleware.go  # HTTP middleware for authz
│
├── config/            # Configuration loading
│   ├── config.go      # Load from env/files
│   └── validation.go  # Config validation
│
├── database/          # Database connection
│   ├── postgres.go    # PostgreSQL connection pool
│   └── migrations.go  # Migration runner
│
├── logging/           # Structured logging
│   ├── logger.go      # Wrapper around slog
│   └── middleware.go  # HTTP request logging
│
├── metrics/           # Prometheus metrics
│   ├── metrics.go     # Metric registration
│   └── middleware.go  # HTTP metrics collection
│
├── tracing/           # OpenTelemetry tracing
│   ├── tracer.go      # Tracer initialization
│   └── middleware.go  # HTTP tracing
│
├── models/            # Shared data models
│   ├── user.go        # User struct
│   ├── organization.go
│   ├── api_key.go
│   └── usage.go
│
├── errors/            # Error handling
│   ├── errors.go      # Custom error types
│   └── rfc7807.go     # RFC 7807 Problem Details
│
└── httpclient/        # HTTP client utilities
    ├── client.go      # HTTP client with retries
    └── openai.go      # OpenAI-compatible client
```

## Key Packages

### pkg/auth
**Purpose**: Authentication (JWT tokens, API keys)

```go
// Generate JWT access + refresh tokens
func GenerateTokenPair(userID, orgID string, role string) (accessToken, refreshToken string, err error)

// Validate JWT token
func ValidateJWT(token string) (*Claims, error)

// Hash API key (SHA-256)
func HashAPIKey(key string) string

// Validate API key against hash
func ValidateAPIKey(key, hash string) bool

// HTTP middleware
func AuthMiddleware(next http.Handler) http.Handler
```

### pkg/rbac
**Purpose**: Role-based access control checks

```go
type Role string

const (
    RoleSystemAdmin Role = "system_admin"
    RoleOrgAdmin    Role = "org_admin"
    RoleOrgUser     Role = "org_user"
)

type Permission string

const (
    PermissionCreateOrg         Permission = "create_org"
    PermissionManageOrgUsers    Permission = "manage_org_users"
    PermissionViewOrgUsage      Permission = "view_org_usage"
    PermissionCreateAPIKey      Permission = "create_api_key"
    // ... more permissions
)

// Check if role has permission
func (r Role) HasPermission(p Permission) bool

// Check if user can access resource
func CanAccessResource(userRole Role, userOrgID, resourceOrgID string) bool

// HTTP middleware
func RequirePermission(perm Permission) func(http.Handler) http.Handler
```

### pkg/logging
**Purpose**: Structured JSON logging

```go
// Initialize logger
func NewLogger(service string, level string) *slog.Logger

// Add request ID to context
func WithRequestID(ctx context.Context, requestID string) context.Context

// HTTP middleware - logs all requests
func LoggingMiddleware(logger *slog.Logger) func(http.Handler) http.Handler
```

### pkg/metrics
**Purpose**: Prometheus metrics

```go
// Register common metrics
func RegisterMetrics(serviceName string)

// HTTP metrics middleware
func MetricsMiddleware(next http.Handler) http.Handler

// Custom counters/gauges
var (
    HTTPRequestsTotal = prometheus.NewCounterVec(...)
    HTTPDuration = prometheus.NewHistogramVec(...)
    ActiveConnections = prometheus.NewGauge(...)
)
```

### pkg/database
**Purpose**: Database connection pooling

```go
// Connect to PostgreSQL
func NewPostgresPool(ctx context.Context, databaseURL string) (*pgxpool.Pool, error)

// Run migrations
func RunMigrations(databaseURL string, migrationsPath string) error

// Health check
func CheckDatabaseHealth(ctx context.Context, pool *pgxpool.Pool) error
```

### pkg/models
**Purpose**: Shared data structures

```go
type User struct {
    ID        uuid.UUID  `json:"id" db:"id"`
    OrgID     *uuid.UUID `json:"org_id,omitempty" db:"org_id"`
    Email     string     `json:"email" db:"email"`
    Role      string     `json:"role" db:"role"`
    Status    string     `json:"status" db:"status"`
    CreatedAt time.Time  `json:"created_at" db:"created_at"`
    UpdatedAt time.Time  `json:"updated_at" db:"updated_at"`
}

type Organization struct {
    ID                   uuid.UUID `json:"id" db:"id"`
    Name                 string    `json:"name" db:"name"`
    DisplayName          string    `json:"display_name" db:"display_name"`
    BudgetTokensMonthly  int64     `json:"budget_tokens_monthly" db:"budget_tokens_monthly"`
    TokensUsedCurrentMonth int64   `json:"tokens_used_current_month" db:"tokens_used_current_month"`
    Status               string    `json:"status" db:"status"`
    CreatedAt            time.Time `json:"created_at" db:"created_at"`
    UpdatedAt            time.Time `json:"updated_at" db:"updated_at"`
}

// ... more models
```

### pkg/errors
**Purpose**: Standardized error responses (RFC 7807)

```go
type ProblemDetails struct {
    Type     string `json:"type"`
    Title    string `json:"title"`
    Status   int    `json:"status"`
    Detail   string `json:"detail,omitempty"`
    Instance string `json:"instance,omitempty"`
}

// Common errors
var (
    ErrUnauthorized     = NewProblemDetails(401, "Unauthorized", "Authentication required")
    ErrForbidden        = NewProblemDetails(403, "Forbidden", "Insufficient permissions")
    ErrNotFound         = NewProblemDetails(404, "Not Found", "Resource not found")
    ErrBudgetExceeded   = NewProblemDetails(429, "Budget Exceeded", "Organization token budget exceeded")
)

// Write error as JSON response
func WriteProblemDetails(w http.ResponseWriter, pd *ProblemDetails)
```

## Testing Strategy

### Unit Tests Required
- All public functions must have unit tests
- Mock external dependencies (database, HTTP clients)
- Test error conditions
- Minimum 90% coverage for pkg/

### Integration Tests
- pkg/database tested against real PostgreSQL (Testcontainers)
- pkg/auth JWT validation tested end-to-end
- pkg/metrics validated with Prometheus scraping

## Contracts

See `contracts/` for:
- `pkg-structure.md` - Detailed API documentation
- `examples/` - Example usage of each package

## Configuration

Each package should be configurable via:
- Environment variables
- Struct-based configuration passed to constructors
- Sensible defaults

## Next Steps

After shared libraries are complete:
1. Services can import these packages
2. Implement **005-user-org-service** using pkg/
3. Implement **006-api-router-service** using pkg/
4. Implement **007-analytics-service** using pkg/

## References

- Go Standard Library patterns
- slog (structured logging): https://pkg.go.dev/log/slog
- pgx (PostgreSQL driver): https://github.com/jackc/pgx
- Prometheus Go client: https://prometheus.io/docs/guides/go-application/

