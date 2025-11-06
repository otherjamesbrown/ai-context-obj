# Spec 012: End-to-End Test Harness

## Purpose

Comprehensive E2E tests that validate the entire system works correctly from user perspective.

## Dependencies

All specs (000-011) - this tests the complete system.

## Success Criteria

- [ ] Happy path test passes in CI
- [ ] All critical user flows covered
- [ ] Tests run against real deployed environment
- [ ] Tests clean up after themselves
- [ ] Test results integrated in CI/CD

## Test Categories

### 1. Happy Path (CI - Fast)
Must complete in < 5 minutes, runs on every PR.

```go
func TestHappyPath(t *testing.T) {
    // 1. Create organization
    org := createOrganization("test-org-" + uuid.New())
    defer deleteOrganization(org.ID)
    
    // 2. Create org admin user
    admin := createUser(org.ID, "admin@test.com", "org_admin")
    defer deleteUser(admin.ID)
    
    // 3. Login and get JWT
    tokens := login("admin@test.com", "password")
    
    // 4. Create API key
    apiKey := createAPIKey(tokens.Access, "test-key")
    defer revokeAPIKey(apiKey.ID)
    
    // 5. Call inference API
    response := callInference(apiKey.Key, "llama3", "Hello world")
    assert.Equal(t, 200, response.StatusCode)
    
    // 6. Wait for usage to be logged
    time.Sleep(10 * time.Second)
    
    // 7. Query analytics - verify usage recorded
    usage := getOrgUsage(tokens.Access, org.ID)
    assert.Greater(t, usage.TotalTokens, 0)
}
```

### 2. Authentication Tests
```go
func TestInvalidAPIKey(t *testing.T)
func TestRevokedAPIKey(t *testing.T)
func TestExpiredJWT(t *testing.T)
func TestRefreshToken(t *testing.T)
```

### 3. RBAC Tests
```go
func TestOrgUserCannotDeleteOthersKeys(t *testing.T)
func TestOrgAdminCanDeleteUsersKeys(t *testing.T)
func TestSystemAdminCanAccessAllOrgs(t *testing.T)
```

### 4. Budget Tests
```go
func TestBudgetEnforcement(t *testing.T) {
    // Set low budget (1000 tokens)
    // Make requests until exceeded
    // Verify 429 error returned
}
```

### 5. Model Management Tests
```go
func TestDisabledModelReturnsError(t *testing.T)
func TestUnknownModelReturnsError(t *testing.T)
func TestMultipleModels(t *testing.T)
```

### 6. Audit Log Tests
```go
func TestAuditLogRecordsKeyCreation(t *testing.T)
func TestAuditLogRecordsKeyRevocation(t *testing.T)
func TestAuditLogRecordsBudgetChange(t *testing.T)
```

## Test Infrastructure

```
tests/e2e/
├── go.mod
├── main_test.go           # Test suite setup
├── happy_path_test.go     # Critical path
├── auth_test.go           # Authentication tests
├── rbac_test.go           # Authorization tests
├── budget_test.go         # Budget enforcement
├── audit_test.go          # Audit logging
├── helpers/
│   ├── api_client.go      # HTTP client
│   ├── fixtures.go        # Test data
│   └── cleanup.go         # Resource cleanup
└── README.md
```

## Environment Configuration

Tests need to know where services are:

```bash
# .env.test
USER_SERVICE_URL=https://api-dev.ias.linode.com
API_ROUTER_URL=https://inference-dev.ias.linode.com
ANALYTICS_SERVICE_URL=https://analytics-dev.ias.linode.com
SYSTEM_ADMIN_EMAIL=test-admin@linode.com
SYSTEM_ADMIN_PASSWORD=<test-password>
```

## Test Execution

### In CI (GitHub Actions)
```yaml
name: E2E Tests

on: [pull_request]

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
      - name: Run E2E Tests
        run: |
          cd tests/e2e
          go test -v ./... -timeout 10m
        env:
          USER_SERVICE_URL: ${{ secrets.DEV_USER_SERVICE_URL }}
          API_ROUTER_URL: ${{ secrets.DEV_API_ROUTER_URL }}
```

### Locally
```bash
cd tests/e2e
export USER_SERVICE_URL=http://localhost:8080
export API_ROUTER_URL=http://localhost:8081
go test -v ./...
```

## Test Data Management

### Setup
- Create test organization with prefix "e2e-test-"
- Create test users
- Create test API keys

### Cleanup
- Delete all resources created during test
- Use `defer` statements for cleanup
- Tag test resources for easy identification

## Nightly Regression Suite

Extended tests that run overnight:
- Load testing (1000 concurrent requests)
- Long-running tests (streaming for 5 minutes)
- Error recovery scenarios
- Database failure scenarios

## Contracts

See `contracts/` for:
- `test-scenarios.md` - Complete test case list
- `test-data.json` - Sample test data

## Success Metrics

### Target Test Coverage
- 100% of critical user flows
- 90% of API endpoints
- All RBAC permission combinations
- All error codes (401, 403, 429, 500)

### Performance Targets
- Happy path test: < 5 minutes
- Full regression: < 30 minutes
- No flaky tests (>99% pass rate)

## Next Steps

After E2E tests pass:
- System is validated and ready for production
- Can deploy with confidence
- Ongoing tests catch regressions

