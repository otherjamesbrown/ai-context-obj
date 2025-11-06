# Spec 005: User & Organization Service

**Status**: In Progress  
**Version**: 1.1  
**Last Updated**: 2025-11-06

## Purpose

Microservice for managing users, organizations, API keys, and RBAC. This is the authentication and authorization service for the entire platform. Now includes GitOps reconciliation support for declarative org management.

## Dependencies

- **003-database-schemas**: Database tables exist
- **004-shared-libraries**: pkg/ utilities available
- **002-local-dev-environment**: For local testing

## Success Criteria

- [ ] All management API endpoints functional
- [ ] RBAC middleware enforces permissions
- [ ] JWT token generation/validation works
- [ ] API key creation/revocation works
- [ ] Audit logging for all write operations
- [ ] Unit tests (>80% coverage)
- [ ] Integration tests with real PostgreSQL
- [ ] OpenAPI spec documented

## User Stories

### Organization Management
- **As a System Admin**: Create/manage organizations, set budgets
- **As an Org Admin**: View my organization's details, update budgets

### User Management  
- **As a System Admin**: Invite users to any organization
- **As an Org Admin**: Invite/remove users in my organization
- **As an Org User**: View my profile, update my password

### API Key Management
- **As an Org Admin/User**: Create API keys for programmatic access
- **As an Org Admin/User**: View/revoke my own keys
- **As an Org Admin**: View/revoke all keys in my organization

### Authentication
- **As any user**: Login with email/password to get JWT tokens
- **As any user**: Refresh expired access tokens
- **As a service**: Validate API keys for inference requests

## API Endpoints

### Authentication
```
POST   /api/v1/auth/login              # Email/password login → JWT tokens
POST   /api/v1/auth/refresh            # Refresh access token
POST   /api/v1/auth/logout             # Revoke refresh token
POST   /api/v1/auth/validate-api-key   # Validate API key (internal)
```

### Organizations
```
POST   /api/v1/organizations           # Create organization
GET    /api/v1/organizations           # List all (System Admin only)
GET    /api/v1/organizations/:id       # Get organization details
PATCH  /api/v1/organizations/:id       # Update organization
DELETE /api/v1/organizations/:id       # Delete organization (soft delete)
PATCH  /api/v1/organizations/:id/budget # Update token budget
```

### Users
```
POST   /api/v1/users                   # Create user (System Admin)
GET    /api/v1/users                   # List users (filtered by org)
GET    /api/v1/users/:id               # Get user details
PATCH  /api/v1/users/:id               # Update user
DELETE /api/v1/users/:id               # Delete user (soft delete)
POST   /api/v1/users/invite            # Invite user to org
POST   /api/v1/users/accept-invite     # Accept invitation, set password
POST   /api/v1/users/:id/reset-password # Reset password (System Admin only)
```

### API Keys
```
POST   /api/v1/api-keys                # Create API key
GET    /api/v1/api-keys                # List keys (own or org-wide)
GET    /api/v1/api-keys/:id            # Get key details
DELETE /api/v1/api-keys/:id            # Revoke API key
```

### Audit Logs
```
GET    /api/v1/audit-logs              # List audit logs (filtered by org)
GET    /api/v1/audit-logs/:id          # Get specific audit log
```

### GitOps Management (New in v1.1)
```
POST   /api/v1/organizations/:id/gitops/enable   # Enable GitOps for org
POST   /api/v1/organizations/:id/gitops/disable  # Disable GitOps for org
POST   /api/v1/organizations/:id/gitops/sync     # Trigger manual sync
GET    /api/v1/organizations/:id/gitops/status   # Get sync status
GET    /api/v1/organizations/:id/gitops/drift    # Detect config drift
```

### Health & Metrics
```
GET    /health                         # Liveness probe
GET    /ready                          # Readiness probe (checks DB)
GET    /metrics                        # Prometheus metrics
```

## Service Architecture

```
services/user-service/
├── cmd/
│   └── server/
│       └── main.go              # Entry point, wiring
│
├── internal/
│   ├── api/                     # HTTP handlers
│   │   ├── auth_handler.go
│   │   ├── org_handler.go
│   │   ├── user_handler.go
│   │   ├── apikey_handler.go
│   │   └── audit_handler.go
│   │
│   ├── service/                 # Business logic
│   │   ├── auth_service.go
│   │   ├── org_service.go
│   │   ├── user_service.go
│   │   ├── apikey_service.go
│   │   ├── audit_service.go
│   │   └── gitops_reconciler.go  # NEW: GitOps reconciliation
│   │
│   ├── repository/              # Database access
│   │   ├── org_repository.go
│   │   ├── user_repository.go
│   │   ├── apikey_repository.go
│   │   └── audit_repository.go
│   │
│   └── middleware/              # Custom middleware
│       ├── auth.go              # JWT validation
│       ├── rbac.go              # Permission checks
│       └── audit.go             # Audit logging
│
├── tests/
│   ├── unit/
│   │   └── service_test.go
│   └── integration/
│       └── api_test.go
│
├── Dockerfile
├── Makefile
└── go.mod
```

## Key Implementation Details

### JWT Token Generation
```go
// Generate access token (1 hour) + refresh token (7 days)
accessToken, refreshToken, err := auth.GenerateTokenPair(user.ID, user.OrgID, user.Role)

// Store refresh token hash in database
// Return both tokens to client
```

### API Key Creation
```go
// Generate random API key with prefix
apiKey := "ias_" + generateRandomString(32)

// Hash with SHA-256 for storage
keyHash := sha256.Sum256([]byte(apiKey))
keyHashStr := hex.EncodeToString(keyHash[:])

// Store: key_prefix (ias_xxxx...), key_hash, user_id, org_id, name
// Return plain text key to user ONCE
```

### API Key Validation (Internal Endpoint)
```go
// Hash provided key
keyHash := sha256.Sum256([]byte(providedKey))

// Query database for matching hash WHERE revoked_at IS NULL
// Return user_id, org_id, permissions

// Update last_used_at asynchronously (don't block)
```

### RBAC Middleware
```go
func RequireRole(allowedRoles ...string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Extract user from JWT claims in context
            user := getUserFromContext(r.Context())
            
            // Check if user's role is in allowed list
            if !contains(allowedRoles, user.Role) {
                errors.WriteProblemDetails(w, errors.ErrForbidden)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}
```

### Audit Logging
```go
// Middleware that logs all write operations
func AuditMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Method != "GET" {
            user := getUserFromContext(r.Context())
            
            // Capture request body
            body, _ := io.ReadAll(r.Body)
            r.Body = io.NopCloser(bytes.NewBuffer(body))
            
            // After request completes, log to audit_log table
            defer func() {
                logAuditEvent(user, r.Method, r.URL.Path, body)
            }()
        }
        
        next.ServeHTTP(w, r)
    })
}
```

### GitOps Reconciliation (New in v1.1)
```go
// Background service that syncs Git repo config to database
type GitOpsReconciler struct {
    repo GitRepository
    db   Database
    interval time.Duration // 30 seconds default
}

func (r *GitOpsReconciler) Run(ctx context.Context) {
    ticker := time.NewTicker(r.interval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            r.reconcileAll(ctx)
        case <-ctx.Done():
            return
        }
    }
}

func (r *GitOpsReconciler) reconcileAll(ctx context.Context) {
    // Get all orgs with GitOps enabled
    orgs, _ := r.db.GetGitOpsEnabledOrgs(ctx)
    
    for _, org := range orgs {
        r.reconcileOrg(ctx, org)
    }
}

func (r *GitOpsReconciler) reconcileOrg(ctx context.Context, org Organization) {
    // 1. Fetch org config from Git repo (orgs/<org-name>/org.yaml)
    config, commit, err := r.repo.FetchOrgConfig(org.Name)
    if err != nil {
        r.db.UpdateGitOpsSyncState(org.ID, "error", err.Error())
        return
    }
    
    // 2. Parse YAML config
    desiredState, _ := parseOrgConfig(config)
    
    // 3. Get current state from database
    currentState, _ := r.db.GetOrgState(org.ID)
    
    // 4. Calculate diff
    diff := calculateDiff(currentState, desiredState)
    
    // 5. Apply changes (user budgets, allowed models, user membership)
    if len(diff) > 0 {
        r.db.ApplyOrgChanges(org.ID, diff)
        
        // Log drift in audit log
        for _, change := range diff {
            r.db.LogAudit(org.ID, nil, "gitops_reconcile", change)
        }
    }
    
    // 6. Update sync state
    r.db.UpdateGitOpsSyncState(org.ID, "synced", commit)
}
```

### Password Reset (CLI-Only, v1.1)
```go
// POST /api/v1/users/:id/reset-password (System Admin only)
func (h *UserHandler) ResetPassword(w http.ResponseWriter, r *http.Request) {
    user := getUserFromContext(r.Context())
    
    // Enforce: System Admin only
    if user.Role != "system_admin" {
        errors.WriteProblemDetails(w, errors.ErrForbidden)
        return
    }
    
    // Validate password complexity
    newPassword := r.FormValue("password")
    if !validatePasswordComplexity(newPassword) {
        errors.WriteProblemDetails(w, errors.New(
            "Password must be 12+ chars with mixed case, numbers, symbols"))
        return
    }
    
    // Hash and update
    hash := bcrypt.GenerateFromPassword([]byte(newPassword), 12)
    h.userService.UpdatePasswordHash(targetUserID, hash)
    
    // Audit log
    h.auditService.Log(user.ID, "password_reset", targetUserID)
    
    w.WriteHeader(http.StatusNoContent)
}
```

## Configuration

### Environment Variables
```bash
PORT=8080
DATABASE_URL=postgres://...
JWT_SECRET=<random-secret>
JWT_ACCESS_TTL=1h
JWT_REFRESH_TTL=168h  # 7 days
LOG_LEVEL=info
```

## Contracts

See `contracts/` for:
- `api-spec.yaml` - Complete OpenAPI 3.0 specification
- `rbac-matrix.md` - Permission mappings
- `examples/` - Example API requests/responses

## Testing Strategy

### Unit Tests
- Business logic in service layer
- RBAC permission checks
- JWT token generation/validation
- API key hashing

### Integration Tests (Testcontainers)
- All API endpoints against real PostgreSQL
- Authentication flow end-to-end
- RBAC enforcement
- Audit log creation

### Test Scenarios
1. **Happy Path**: Create org → Create user → Generate API key → Validate key
2. **RBAC**: Org User attempts to delete another user's key → 403
3. **Audit**: Create key → Check audit_log table has entry
4. **Token Refresh**: Login → Wait 1 hour → Refresh token → Get new access token

## Deployment

### Docker Image
- Multi-stage build (builder + runtime)
- Alpine-based final image (~20MB)
- Runs as non-root user

### Kubernetes Deployment
```yaml
replicas: 3
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### Health Checks
- Liveness: `/health` (always returns 200)
- Readiness: `/ready` (checks database connection)

## Monitoring

### Key Metrics
- `user_service_requests_total{endpoint, method, status}`
- `user_service_request_duration_seconds{endpoint}`
- `user_service_db_connections{state}`
- `user_service_api_keys_total{org_id}`

### Logs
All requests logged with:
- Request ID
- User ID (if authenticated)
- Endpoint
- Status code
- Duration

## Security

### Password Storage
- bcrypt with cost factor 12
- Never log passwords

### API Key Storage
- SHA-256 hash only
- Never return full key after creation

### Rate Limiting
Not implemented in V1 (defer to API Gateway or V2)

## Open Questions

### Resolved
- ✅ JWT vs session cookies? → JWT
- ✅ Where to store refresh tokens? → Database table
- ✅ How to bootstrap first admin? → CLI tool (spec 009)

### To Be Resolved
- Email sending for invitations? (Defer to V2, use invite links)
- Web UI password reset flow? (Defer to V2, CLI available in V1)

### Newly Added (v1.1)
- ✅ GitOps reconciliation service for declarative org management
- ✅ Password reset endpoint (System Admin only, for CLI use)
- ✅ Per-user token limit support
- ✅ Allowed models per org (GitOps or API managed)

## Next Steps

After this service is complete:
1. Implement **006-api-router-service** (uses this for auth)
2. Implement **009-admin-cli** (calls these APIs)
3. Implement **008-web-portal** (consumes these APIs)

## References

- OpenAPI Specification: https://swagger.io/specification/
- RFC 7807 Problem Details: https://tools.ietf.org/html/rfc7807
- JWT Best Practices: https://tools.ietf.org/html/rfc8725

