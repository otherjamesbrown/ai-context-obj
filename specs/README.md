# Inference-as-a-Service - Specifications

This directory contains all feature specifications for the platform, organized using the [Spec-Kit methodology](https://github.com/github/spec-kit).

## Overview

Each subdirectory represents a distinct feature or component that can be implemented independently (respecting dependencies).

## Implementation Order

Implement specs in this order to respect dependencies:

### Phase 1: Foundation (Week 1-2)
```
000-project-setup          → Repository structure, tooling, CI templates
001-infrastructure         → LKE cluster, GPU/CPU pools, managed PostgreSQL  
002-local-dev-environment  → docker-compose for local development
003-database-schemas       → PostgreSQL schemas + TimescaleDB setup
004-shared-libraries       → Shared Go packages (auth, RBAC, logging, metrics)
```

**Milestone**: Developers can clone repo and run services locally against real databases.

### Phase 2: Core Services (Week 3-4)
```
005-user-org-service       → User/org management, RBAC, API keys
006-api-router-service     → Main inference API, budget enforcement
007-analytics-service      → Usage tracking and analytics
```

**Milestone**: Full inference flow works end-to-end (auth → route → log usage).

### Phase 3: User Interfaces (Week 5)
```
008-web-portal             → React UI for management
009-admin-cli              → CLI tool for system admins
```

**Milestone**: Users can manage platform via UI and CLI.

### Phase 4: Production Ready (Week 6-7)
```
010-vllm-deployment        → Deploy vLLM on GPU nodes
011-observability          → LGTM stack (monitoring, logs, traces)
012-e2e-tests              → End-to-end test harness
```

**Milestone**: Production-ready system with observability and testing.

## Specification Structure

Each spec follows this structure:

```
NNN-feature-name/
├── spec.md              # Complete feature specification
└── contracts/           # Concrete artifacts
    ├── api-spec.yaml    # OpenAPI specs
    ├── schema.sql       # Database schemas
    ├── examples/        # Example code/configs
    └── ...
```

## Spec Contents

### spec.md
- **Purpose**: What problem does this solve?
- **Dependencies**: What must exist before this can be built?
- **Success Criteria**: How do we know it's done?
- **User Stories**: Who uses this and why?
- **Technical Scope**: What's included/excluded?
- **Implementation Details**: Key algorithms, architecture
- **Contracts**: Reference to concrete artifacts
- **Testing Strategy**: How to test this
- **Configuration**: Environment variables, config files
- **Next Steps**: What to build after this

### contracts/
Concrete, implementable artifacts:
- OpenAPI specifications
- Database schemas (SQL)
- Message queue schemas (JSON)
- Helm chart values
- Example code
- Configuration templates

## Quick Reference

| Spec | Name | Purpose | Dependencies |
|------|------|---------|--------------|
| 000 | Project Setup | Repository structure, tooling | None |
| 001 | Infrastructure | LKE cluster, managed PostgreSQL | None |
| 002 | Local Dev | docker-compose setup | 000 |
| 003 | Database Schemas | PostgreSQL + TimescaleDB schemas | 001, 002 |
| 004 | Shared Libraries | Go pkg/ utilities | 000, 003 |
| 005 | User/Org Service | Auth, RBAC, API keys | 003, 004 |
| 006 | API Router | Main inference API | 003, 004, 005 |
| 007 | Analytics Service | Usage tracking, dashboards | 003, 004, 006 |
| 008 | Web Portal | React UI | 005, 007 |
| 009 | Admin CLI | CLI tool | 005 |
| 010 | vLLM Deployment | GPU inference engines | 001, 009 |
| 011 | Observability | LGTM stack | 001, all services |
| 012 | E2E Tests | Test harness | All |

## Key Technologies by Spec

| Spec | Technologies |
|------|-------------|
| 000-004 | Go, PostgreSQL, TimescaleDB, Redis, RabbitMQ, Docker |
| 005-007 | Go, PostgreSQL, Redis, RabbitMQ, OpenTelemetry |
| 008 | React, TypeScript, TailwindCSS, Vite |
| 009 | Go, Cobra CLI |
| 010 | vLLM, Helm, Kubernetes, NVIDIA GPUs |
| 011 | Prometheus, Grafana, Loki, Tempo |
| 012 | Go, Testify |

## Getting Started

1. **Read the Constitution**: Start with `/memory/constitution.md` to understand architectural principles.

2. **Review the PRD**: See `/PRD/prd.md` for the original product requirements.

3. **Implement Sequentially**: Follow the implementation order above.

4. **Create Contracts**: Before coding, create all contracts (schemas, API specs) for a spec.

5. **Test Thoroughly**: Each spec includes testing requirements.

## Using Specs with AI Assistants

These specs are designed to work with AI coding assistants (Cursor, GitHub Copilot, Claude):

1. Point the AI to the spec and contracts
2. Reference the constitution for architectural guidance
3. Implement the feature following the spec
4. Validate against success criteria

Example prompt:
```
I want to implement spec 005 (User/Org Service). 

Please review:
- memory/constitution.md (architectural principles)
- specs/005-user-org-service/spec.md (feature spec)
- specs/005-user-org-service/contracts/ (contracts)

Implement the service following our conventions.
```

## Contributing

When adding a new spec:

1. Follow the naming convention: `NNN-feature-name/`
2. Include complete `spec.md` with all sections
3. Create concrete contracts (no TODOs or placeholders)
4. Define clear success criteria
5. Document dependencies
6. Add to this README

## Questions?

- Architectural questions? → Check `/memory/constitution.md`
- Product questions? → Check `/PRD/prd.md`
- Technical questions? → Check individual spec.md files
- Implementation help? → Check contracts/ directories

---

**Remember**: Specs are living documents. Update them as requirements change or implementation reveals new details.

