# Getting Started with Inference-as-a-Service Development

## 🎉 What We've Built

Your project is now structured using the **Spec-Kit methodology** with complete specifications for all components. Here's what exists:

### Core Documents
1. **`memory/constitution.md`**: Your architectural "bible" - all design decisions, tech stack choices, and principles
2. **`PRD/prd.md`**: Original product requirements (preserved for reference)
3. **`README.md`**: Project overview and quick start guide
4. **`specs/README.md`**: Specification index and implementation guide

### 13 Complete Specifications
Each spec includes:
- Clear purpose and success criteria
- User stories
- Technical implementation details
- Dependencies
- Testing strategy
- Contracts directory (for concrete artifacts like schemas, API specs)

```
000-project-setup          → Repo structure, tooling, CI
001-infrastructure         → Linode LKE + managed PostgreSQL
002-local-dev-environment  → docker-compose setup
003-database-schemas       → PostgreSQL + TimescaleDB schemas
004-shared-libraries       → Go pkg/ utilities
005-user-org-service       → Auth, RBAC, user management
006-api-router-service     → Main inference API
007-analytics-service      → Usage tracking
008-web-portal             → React UI
009-admin-cli              → CLI tool
010-vllm-deployment        → vLLM on GPU nodes
011-observability          → LGTM stack
012-e2e-tests              → Test harness
```

## 🚦 Next Steps - Starting Implementation

### Phase 1: Foundation (Start Here)

#### Step 1: Project Setup (Spec 000)
```bash
# What to implement:
# - Create actual directory structure (services/, pkg/, web/, etc.)
# - Create root Makefile
# - Create go.work file
# - Set up GitHub Actions workflows
# - Create .gitignore, .editorconfig

# Use AI assistant:
"Implement spec 000-project-setup. Create the directory structure, 
Makefile, go.work file, and GitHub Actions templates as specified 
in specs/000-project-setup/spec.md"
```

#### Step 2: Database Schemas (Spec 003)
```bash
# What to implement:
# - Create migration files in migrations/
# - Write SQL for all tables (organizations, users, api_keys, etc.)
# - Set up TimescaleDB hypertable for usage_events
# - Create seed data

# Use AI assistant:
"Implement spec 003-database-schemas. Create golang-migrate files 
with all the table definitions from specs/003-database-schemas/spec.md. 
Follow the exact schema provided."
```

#### Step 3: Local Development Environment (Spec 002)
```bash
# What to implement:
# - Create docker-compose.yml
# - Create mock inference service
# - Set up environment variables
# - Test that `make dev-up` works

# Use AI assistant:
"Implement spec 002-local-dev-environment. Create docker-compose.yml 
with PostgreSQL, Redis, RabbitMQ, and mock inference service as 
specified in specs/002-local-dev-environment/spec.md"
```

#### Step 4: Shared Libraries (Spec 004)
```bash
# What to implement:
# - Create pkg/ packages (auth, rbac, logging, etc.)
# - Implement JWT handling
# - Implement RBAC middleware
# - Create shared data models

# Use AI assistant:
"Implement spec 004-shared-libraries. Create all packages in pkg/ 
as specified in specs/004-shared-libraries/spec.md. Follow the 
constitution principles in memory/constitution.md"
```

### Phase 2: Core Services

#### Step 5: User/Org Service (Spec 005)
```bash
# This is a full Go microservice - good candidate for AI generation

# Use AI assistant:
"Implement spec 005-user-org-service. This is a complete Go microservice.
Review:
- memory/constitution.md (architectural principles)
- specs/005-user-org-service/spec.md (complete specification)
- specs/003-database-schemas/spec.md (database schema)
- pkg/ (shared libraries to use)

Build the service with proper structure, tests, and documentation."
```

#### Step 6: API Router (Spec 006)
```bash
# The most critical service - handles inference requests

# Use AI assistant:
"Implement spec 006-api-router-service. This is the main inference API.
Review specs/006-api-router-service/spec.md for complete details.
Must be OpenAI-compatible and include streaming support."
```

#### Step 7: Analytics Service (Spec 007)
```bash
# RabbitMQ consumer + analytics API

# Use AI assistant:
"Implement spec 007-analytics-service per specs/007-analytics-service/spec.md.
Includes both queue consumer and HTTP API."
```

### Phase 3: User Interfaces

#### Step 8: Web Portal (Spec 008)
```bash
# React application

# Use AI assistant:
"Implement spec 008-web-portal. Create a React application with 
TypeScript, TailwindCSS, and shadcn/ui components as specified in 
specs/008-web-portal/spec.md"
```

#### Step 9: CLI Tool (Spec 009)
```bash
# Go CLI using Cobra

# Use AI assistant:
"Implement spec 009-admin-cli. Create the ias-admin CLI tool 
using Cobra framework as specified in specs/009-admin-cli/spec.md"
```

### Phase 4: Production Deployment

Continue with specs 001, 010, 011, 012 for infrastructure and deployment.

## 💡 How to Work with AI Assistants

### General Pattern
```
1. Point AI to the specific spec
2. Reference the constitution for principles
3. Ask AI to implement following the spec
4. Review and iterate
```

### Example Prompts

#### For Services (Go)
```
I want to implement [spec name].

Please review:
- memory/constitution.md (architectural principles)
- specs/[spec-number]/spec.md (complete specification)
- specs/003-database-schemas/spec.md (database schema)

Implement this service following our standards:
- Go 1.21
- Structured logging (slog)
- OpenTelemetry tracing
- Prometheus metrics
- Unit tests + integration tests
- Follow the exact structure in the spec
```

#### For Contracts (Schemas, APIs)
```
Create the contracts for spec [number].

Based on specs/[spec-number]/spec.md, create:
1. OpenAPI 3.0 specification (api-spec.yaml)
2. Example requests/responses
3. Error response formats

Save to specs/[spec-number]/contracts/
```

#### For Tests
```
Create comprehensive tests for [service name].

Follow the testing strategy in specs/[spec-number]/spec.md:
- Unit tests for business logic
- Integration tests with Testcontainers
- Test all error conditions
- 80%+ coverage
```

## 🎯 Key Principles (From Constitution)

### Always Remember
1. **API-First**: Build APIs before UIs
2. **Stateless**: No in-memory state in services
3. **Async Non-Critical**: Use RabbitMQ for usage logging
4. **Security**: Hash API keys, bcrypt passwords, RBAC on everything
5. **Observable**: Logs, metrics, traces on all services
6. **Tested**: Unit, integration, E2E tests required

### Tech Stack (Non-Negotiable)
- **Services**: Go only
- **State DB**: PostgreSQL with TimescaleDB
- **Cache**: Redis
- **Queue**: RabbitMQ
- **Inference**: vLLM
- **Frontend**: React + TypeScript
- **IaC**: Terraform + Helm

## 🔍 Critical Questions Answered

### Q: Do we need all 13 specs?
**A**: For a complete V1.0, yes. But you can implement incrementally:
- Specs 000-007 give you a working inference platform
- Specs 008-009 add user interfaces
- Specs 010-012 make it production-ready

### Q: Can we simplify the database setup?
**A**: Yes, it's already simplified:
- One managed PostgreSQL (not 4 separate databases)
- TimescaleDB extension (not separate ClickHouse)
- RabbitMQ and Redis deployed via Helm (no StatefulSets to manage)

### Q: What about the first admin user?
**A**: Bootstrap via CLI:
```bash
ias-admin bootstrap create-super-admin \
  --email admin@linode.com \
  --password <password>
```

### Q: Do we need email sending for V1?
**A**: No - invite flow uses copy-paste links (no SMTP needed)

### Q: What's the model deployment strategy?
**A**: 
1. Deploy vLLM with Helm (spec 010)
2. Register model via CLI: `ias-admin model add ...`
3. API Router discovers models from database

## 📊 Progress Tracking

As you implement, update the checkboxes in each spec's "Success Criteria" section.

Example workflow:
1. Pick a spec to implement
2. Read the spec.md completely
3. Create contracts (if not done)
4. Implement following the spec
5. Test thoroughly
6. Check off success criteria
7. Move to next spec

## 🆘 If You Get Stuck

### Common Issues

**"The spec is too big to implement at once"**
- Break it into smaller tasks
- Implement one endpoint at a time
- Reference the "Implementation Tasks" section in each spec

**"AI is not following the architecture"**
- Point AI explicitly to `memory/constitution.md`
- Remind it of specific principles (stateless, API-first, etc.)
- Show example from another similar service

**"Database migrations failing"**
- Check that TimescaleDB extension is installed
- Verify connection string is correct
- Run migrations one at a time to isolate issues

**"Local environment not working"**
- Check all containers are running: `docker-compose ps`
- Check logs: `docker-compose logs`
- Verify ports not already in use

### Getting Help

1. **Check the spec**: Most questions are answered in the detailed specs
2. **Check the constitution**: Architectural decisions are documented
3. **Check the PRD**: Original requirements and context
4. **Ask specific questions**: Reference the spec and ask for clarification

## 🎉 Success Metrics

You'll know you're on track when:

### After Phase 1 (Foundation)
- ✅ `make dev-up` starts all dependencies
- ✅ Database migrations run successfully
- ✅ Shared libraries are tested and working

### After Phase 2 (Services)
- ✅ Can create organization via API
- ✅ Can create user and API key
- ✅ Can call inference API with key
- ✅ Usage is logged to database

### After Phase 3 (UIs)
- ✅ Can login to web portal
- ✅ Can create API key via UI
- ✅ Can view usage dashboard
- ✅ Can manage orgs via CLI

### After Phase 4 (Production)
- ✅ Deployed to Linode LKE
- ✅ vLLM serving real models
- ✅ Observability stack working
- ✅ E2E tests passing

## 🚀 Let's Build!

You now have everything you need:
- ✅ Clear architecture (constitution)
- ✅ Complete specifications (13 specs)
- ✅ Implementation order (phases 1-4)
- ✅ AI assistant guidance (example prompts)

**Start with spec 000 (Project Setup) and work your way through!**

Good luck! 🎉

