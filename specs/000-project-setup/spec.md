# Spec 000: Project Setup & Repository Structure

## Purpose

Bootstrap the repository with proper structure, tooling, and CI/CD templates to enable efficient development of all subsequent services.

## Dependencies

None - this is the first spec to implement.

## Success Criteria

- [ ] Repository has a well-defined directory structure
- [ ] Developers can clone repo and run `make setup` to initialize environment
- [ ] Go workspace configured for monorepo development
- [ ] Common Makefile targets work (test, lint, build, docker-build)
- [ ] GitHub Actions CI template exists and can be reused
- [ ] README.md provides clear getting started instructions

## User Stories

**As a Developer:**
- I want to clone the repo and have clear instructions on how to get started
- I want a single command to install dependencies and set up my environment
- I want consistent commands across all services (e.g., `make test` always works)

**As a Platform Engineer:**
- I want CI/CD templates that can be reused for all services
- I want consistent Docker build patterns across all services
- I want to enforce code quality standards (linting, formatting) from day one

## Repository Structure

```
ai-as-a-service/
├── README.md                       # Project overview, getting started
├── CONTRIBUTING.md                 # Contribution guidelines
├── LICENSE                         # MIT or appropriate license
├── Makefile                        # Root-level commands
├── go.work                         # Go workspace file (monorepo)
├── .gitignore                      # Standard Go + Node + IDE ignores
├── .editorconfig                   # Editor configuration
│
├── memory/                         # Spec-kit memory (constitutional principles)
│   └── constitution.md             # ✅ Already created
│
├── specs/                          # All feature specifications
│   ├── 000-project-setup/          # This spec
│   ├── 001-infrastructure/
│   └── ...
│
├── PRD/                            # Original product requirements
│   └── prd.md
│
├── services/                       # All Go microservices
│   ├── user-service/               # User & Org management
│   │   ├── go.mod
│   │   ├── go.sum
│   │   ├── Makefile               # Service-specific tasks
│   │   ├── Dockerfile
│   │   ├── cmd/
│   │   │   └── server/
│   │   │       └── main.go
│   │   ├── internal/              # Private application code
│   │   │   ├── api/               # HTTP handlers
│   │   │   ├── service/           # Business logic
│   │   │   └── repository/        # Database access
│   │   └── tests/
│   │       ├── unit/
│   │       └── integration/
│   │
│   ├── api-router/                # Main inference API
│   │   └── ... (same structure)
│   │
│   └── analytics-service/         # Usage & analytics
│       └── ... (same structure)
│
├── pkg/                           # Shared Go libraries
│   ├── auth/                      # JWT, API key validation
│   ├── rbac/                      # Role-based access control
│   ├── logging/                   # Structured logging helpers
│   ├── metrics/                   # Prometheus metrics helpers
│   ├── config/                    # Configuration loading
│   ├── database/                  # DB connection pooling
│   └── models/                    # Shared data models
│
├── web/                           # React web portal
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── hooks/
│   │   └── App.tsx
│   └── Dockerfile
│
├── cli/                           # ias-admin CLI tool
│   ├── go.mod
│   ├── Makefile
│   ├── Dockerfile
│   ├── cmd/
│   │   └── ias-admin/
│   │       └── main.go
│   └── internal/
│       └── commands/
│
├── infrastructure/                # Terraform configurations
│   ├── terraform/
│   │   ├── lke-cluster/          # LKE cluster + node pools
│   │   ├── managed-dbs/          # Linode PostgreSQL
│   │   └── modules/              # Reusable Terraform modules
│   └── README.md
│
├── deployments/                   # Kubernetes manifests & Helm charts
│   ├── charts/                    # Helm charts for services
│   │   ├── user-service/
│   │   ├── api-router/
│   │   ├── analytics-service/
│   │   ├── web-portal/
│   │   ├── rabbitmq/
│   │   └── redis/
│   └── argocd-apps/              # ArgoCD application definitions
│       ├── dev/
│       ├── staging/
│       └── production/
│
├── migrations/                    # Database migrations (golang-migrate)
│   ├── 000001_initial_schema.up.sql
│   ├── 000001_initial_schema.down.sql
│   └── README.md
│
├── tests/                         # E2E tests
│   └── e2e/
│       ├── go.mod
│       ├── happy_path_test.go
│       └── regression_test.go
│
├── scripts/                       # Utility scripts
│   ├── setup.sh                  # Initial development setup
│   ├── check-prerequisites.sh    # Verify tools installed
│   └── gen-api-key.sh            # Generate test API keys
│
├── docs/                          # Documentation
│   ├── architecture/
│   │   ├── system-overview.md
│   │   └── diagrams/
│   ├── api/                      # OpenAPI specs
│   ├── development/
│   │   ├── local-setup.md
│   │   └── testing-guide.md
│   └── deployment/
│       ├── production-deploy.md
│       └── disaster-recovery.md
│
├── .github/                       # GitHub-specific files
│   ├── workflows/                # GitHub Actions
│   │   ├── service-ci.yml       # Template for service CI
│   │   ├── web-ci.yml           # Frontend CI
│   │   ├── terraform-ci.yml     # Infrastructure CI
│   │   └── e2e-tests.yml        # E2E test runner
│   ├── ISSUE_TEMPLATE/
│   ├── PULL_REQUEST_TEMPLATE.md
│   └── CODEOWNERS
│
└── docker-compose.yml             # Local development environment
```

## Technical Scope

### Included in This Spec

1. **Directory structure creation** - All folders and initial files
2. **Root Makefile** - Common targets for all services
3. **Go workspace** - go.work file for monorepo
4. **GitHub Actions templates** - Reusable CI workflows
5. **Docker build standards** - Multi-stage Dockerfile templates
6. **Development scripts** - setup.sh, prerequisites check
7. **Documentation scaffolding** - README, CONTRIBUTING

### Excluded (Other Specs)

- Actual service implementations (specs 005-007)
- Database schemas (spec 003)
- Infrastructure provisioning (spec 001)
- Docker compose setup (spec 002)

## Implementation Tasks

### Task 1: Create Directory Structure
Create all directories as defined above.

### Task 2: Root Makefile
Create `/Makefile` with targets:
- `setup` - Install dependencies, initialize environment
- `test` - Run tests for all services
- `lint` - Run linters for all services
- `build` - Build all services
- `docker-build` - Build all Docker images
- `clean` - Clean build artifacts

### Task 3: Go Workspace Configuration
Create `/go.work`:
```go
go 1.21

use (
    ./services/user-service
    ./services/api-router
    ./services/analytics-service
    ./cli
    ./tests/e2e
    ./pkg
)
```

### Task 4: GitHub Actions Templates

Create `.github/workflows/service-ci.yml` (reusable workflow):
```yaml
name: Service CI Template

on:
  workflow_call:
    inputs:
      service_name:
        required: true
        type: string
      service_path:
        required: true
        type: string

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.21'
      - name: Lint
        run: |
          cd ${{ inputs.service_path }}
          go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
          golangci-lint run
      - name: Test
        run: |
          cd ${{ inputs.service_path }}
          go test -v -race -coverprofile=coverage.out ./...
      - name: Build
        run: |
          cd ${{ inputs.service_path }}
          go build -v ./...

  docker:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: ${{ inputs.service_path }}
          push: true
          tags: ghcr.io/${{ github.repository }}/${{ inputs.service_name }}:${{ github.sha }}
```

### Task 5: Service Dockerfile Template

Create a template multi-stage Dockerfile:
```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder

WORKDIR /build

# Copy go mod files
COPY go.mod go.sum ./
RUN go mod download

# Copy source
COPY . .

# Build binary
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o service ./cmd/server

# Runtime stage
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /app

COPY --from=builder /build/service .

EXPOSE 8080

CMD ["./service"]
```

### Task 6: EditorConfig

Create `.editorconfig`:
```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true

[*.{go,js,ts,tsx,json,yaml,yml}]
indent_style = space
indent_size = 2

[*.go]
indent_style = tab
indent_size = 4

[Makefile]
indent_style = tab
```

### Task 7: Root README.md

Create comprehensive README.md with:
- Project overview
- Architecture diagram (link to docs/architecture/)
- Prerequisites (Go 1.21+, Docker, Make)
- Quick start instructions
- Development workflow
- Links to detailed documentation

### Task 8: Development Setup Script

Create `scripts/setup.sh`:
```bash
#!/bin/bash
set -e

echo "🚀 Setting up Inference-as-a-Service development environment..."

# Check prerequisites
./scripts/check-prerequisites.sh

# Initialize Go workspace
echo "📦 Initializing Go workspace..."
go work sync

# Install development tools
echo "🔧 Installing development tools..."
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
go install github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Install frontend dependencies (if web/ exists)
if [ -d "web" ]; then
    echo "📦 Installing frontend dependencies..."
    cd web && npm install && cd ..
fi

echo "✅ Setup complete! Run 'make help' to see available commands."
```

## Contracts

See `contracts/` directory for:
- `Makefile` - Root makefile template
- `.github/workflows/` - CI/CD workflow templates
- `Dockerfile.template` - Multi-stage Docker build template
- `.gitignore` - Comprehensive gitignore
- `README.template.md` - README structure

## Testing Strategy

### Validation Tests
After implementation, verify:
- `make setup` completes without errors
- `make help` shows all available commands
- Directory structure matches specification
- CI workflow can be triggered successfully

### Manual Verification
- Clone repo fresh
- Run setup script
- Verify all tools installed
- Check that documentation is clear and accurate

## Deployment Notes

This spec doesn't deploy anything - it's pure repository scaffolding.

## Configuration

No runtime configuration needed.

## Next Steps

After this spec is complete:
1. Implement **003-database-schemas** (define database structure)
2. Implement **002-local-dev-environment** (docker-compose setup)
3. Implement **004-shared-libraries** (pkg/ utilities)
4. Then start building services (005, 006, 007)

## Open Questions

None - this spec is straightforward scaffolding.

## References

- GitHub Spec-Kit: https://github.com/github/spec-kit
- Go Workspace: https://go.dev/doc/tutorial/workspaces
- GitHub Actions Reusable Workflows: https://docs.github.com/en/actions/using-workflows/reusing-workflows

