# Inference-as-a-Service (IAS)

**A centralized, multi-tenant AI inference platform for self-hosted open-source models.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## 🎯 Project Vision

Provide a single, unified API (OpenAI-compatible) for accessing self-hosted AI models with:
- **Multi-tenancy**: Organizations, users, and role-based access control (RBAC)
- **Cost Control**: Token budgets and usage tracking
- **Self-Service**: Web UI and CLI for managing everything
- **Performance**: Low-latency inference with streaming support
- **Observability**: Complete monitoring, logging, and tracing

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Clients                               │
│  (Web UI, CLI, Developer Applications)                      │
└────────────┬────────────────────────────────────┬───────────┘
             │                                    │
             ▼                                    ▼
    ┌────────────────┐                  ┌────────────────┐
    │ User/Org       │                  │ API Router     │
    │ Service        │◄─────auth────────│ Service        │
    │ (Management)   │                  │ (Inference)    │
    └───────┬────────┘                  └────────┬───────┘
            │                                    │
            │                                    ▼
            │                           ┌────────────────┐
            │                           │ vLLM Engines   │
            │                           │ (GPU Node Pool)│
            │                           └────────────────┘
            │                                    │
            ▼                                    ▼
    ┌─────────────────────────────────────────────────────┐
    │              PostgreSQL + TimescaleDB                │
    │  (Users, Orgs, Keys, Usage Events)                  │
    └─────────────────────────────────────────────────────┘
            ▲                                    │
            │                                    │
            │                                    ▼
    ┌───────────────┐                  ┌────────────────┐
    │ Analytics     │◄─────queue───────│ RabbitMQ       │
    │ Service       │                  │ (Usage Logs)   │
    └───────────────┘                  └────────────────┘
```

## ✨ Features

### V1.0 Features
- ✅ **OpenAI-Compatible API**: `/v1/chat/completions` with streaming
- ✅ **RBAC**: System Admin, Org Admin, Org User roles
- ✅ **API Key Management**: Create, revoke, track usage
- ✅ **Token Budgets**: Per-organization monthly limits
- ✅ **Usage Analytics**: Token consumption dashboards
- ✅ **Audit Logging**: All management operations logged
- ✅ **Multiple Models**: Dynamic model registry
- ✅ **Web Portal**: React UI for self-service management
- ✅ **CLI Tool**: `ias-admin` for automation
- ✅ **Observability**: Prometheus, Grafana, Loki, Tempo

### Out of Scope (V2+)
- SSO/SAML/OIDC authentication
- External API routing (OpenAI, Anthropic)
- Custom role creation
- Automated billing/invoicing
- Model fine-tuning

## 🚀 Quick Start

### Prerequisites
- Go 1.21+
- Docker & Docker Compose
- Make
- Node.js 20+ (for web portal)
- kubectl (for K8s deployment)
- Linode account (for production)

### Local Development Setup

```bash
# Clone repository
git clone https://github.com/linode/inference-as-a-service.git
cd inference-as-a-service

# Run setup script
make setup

# Start local development environment (PostgreSQL, Redis, RabbitMQ, mock inference)
make dev-up

# The services will be available at:
# - PostgreSQL: localhost:5432
# - Redis: localhost:6379
# - RabbitMQ UI: http://localhost:15672 (user: ias_dev, pass: dev_password)
```

### Build and Run Services

```bash
# Build all services
make build

# Run user service
cd services/user-service
make run

# Run API router
cd services/api-router
make run
```

### Run Tests

```bash
# Run all tests
make test

# Run tests for specific service
cd services/user-service
make test
```

## 📚 Documentation

- **[Constitution](memory/constitution.md)**: Architectural principles and technical standards
- **[PRD](PRD/prd.md)**: Original product requirements document
- **[Specifications](specs/README.md)**: Detailed feature specifications
- **API Documentation**: OpenAPI specs in `docs/api/`
- **Deployment Guide**: See `docs/deployment/`

## 🛠️ Tech Stack

### Backend Services (Go)
- **Language**: Go 1.21+
- **HTTP Framework**: chi router
- **Database**: pgx (PostgreSQL driver)
- **Testing**: Testify + Testcontainers

### Databases
- **State**: PostgreSQL 15 (Linode Managed)
- **Analytics**: TimescaleDB extension
- **Cache**: Redis 7
- **Queue**: RabbitMQ 3.12

### Infrastructure
- **Orchestration**: Kubernetes (Linode LKE)
- **Inference**: vLLM on GPU nodes
- **IaC**: Terraform + Helm
- **GitOps**: ArgoCD
- **CI/CD**: GitHub Actions

### Frontend
- **Framework**: React 18 + TypeScript
- **Styling**: TailwindCSS + shadcn/ui
- **Build**: Vite
- **State**: React Query

### Observability
- **Metrics**: Prometheus + Grafana
- **Logs**: Loki + Promtail
- **Traces**: Tempo + OpenTelemetry
- **Dashboards**: Grafana

## 📦 Project Structure

```
.
├── memory/                  # Spec-kit memory (constitution)
├── specs/                   # Feature specifications
├── PRD/                     # Product requirements
├── services/                # Go microservices
│   ├── user-service/
│   ├── api-router/
│   └── analytics-service/
├── pkg/                     # Shared Go libraries
├── web/                     # React web portal
├── cli/                     # ias-admin CLI tool
├── infrastructure/          # Terraform configurations
├── deployments/             # Helm charts & K8s manifests
├── migrations/              # Database migrations
├── tests/                   # E2E tests
├── docs/                    # Documentation
└── scripts/                 # Utility scripts
```

## 🎓 Development Workflow

### 1. Read the Constitution
Start with [`memory/constitution.md`](memory/constitution.md) to understand architectural principles.

### 2. Pick a Spec
See [`specs/README.md`](specs/README.md) for the implementation order.

### 3. Implement
Follow the spec, create tests, and ensure quality.

### 4. Deploy
Use Helm charts and ArgoCD for GitOps-based deployment.

## 🧪 Testing

### Test Pyramid
1. **Unit Tests**: Pure business logic (>80% coverage)
2. **Integration Tests**: Real databases via Testcontainers
3. **E2E Tests**: Full system testing

### Running Tests
```bash
# Unit tests
go test ./...

# Integration tests (requires Docker)
go test -tags=integration ./...

# E2E tests (requires deployed environment)
cd tests/e2e
go test -v ./...
```

## 🚀 Deployment

### Local (Development)
```bash
make dev-up
```

### Staging/Production (Linode LKE)

#### 1. Provision Infrastructure
```bash
cd infrastructure/terraform
terraform init
terraform plan
terraform apply
```

#### 2. Deploy Services via Helm
```bash
# Install services
helm install user-service deployments/charts/user-service -n ias-production
helm install api-router deployments/charts/api-router -n ias-production
helm install analytics-service deployments/charts/analytics-service -n ias-production
```

#### 3. Deploy vLLM
```bash
helm install llama3-8b deployments/charts/vllm-llama3 -n ias-production
```

#### 4. Bootstrap System
```bash
ias-admin bootstrap create-super-admin \
  --email admin@linode.com \
  --password <secure-password>
```

## 🤝 Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

### Development Principles
1. **API-First**: All functionality via REST API
2. **Stateless Services**: No in-memory state
3. **Async Non-Critical Paths**: Don't block the critical path
4. **Security by Default**: Authentication, authorization, audit logging
5. **Observable**: Logs, metrics, traces on everything
6. **Tested**: Unit, integration, E2E tests required

## 📋 Roadmap

### V1.0 (Current) - Core Platform
- [x] Project setup and infrastructure
- [x] Database schemas
- [x] User/Org management service
- [x] API Router service
- [x] Analytics service
- [x] Web portal
- [x] Admin CLI
- [x] vLLM deployment
- [x] Observability stack
- [ ] E2E tests
- [ ] Production deployment

### V2.0 (Future)
- [ ] SSO/SAML/OIDC integration
- [ ] External API routing (OpenAI, Anthropic)
- [ ] Advanced rate limiting
- [ ] Model fine-tuning support
- [ ] Cost allocation and billing
- [ ] Multi-region support

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- Built using [Spec-Kit](https://github.com/github/spec-kit) methodology
- Inspired by OpenAI API design
- Powered by [vLLM](https://docs.vllm.ai/) for inference
- Running on [Linode](https://www.linode.com/) infrastructure

## 📞 Support

- **Issues**: [GitHub Issues](https://github.com/linode/inference-as-a-service/issues)
- **Discussions**: [GitHub Discussions](https://github.com/linode/inference-as-a-service/discussions)
- **Email**: ias-support@linode.com

---

**Built with ❤️ for the AI community**
