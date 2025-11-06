# Spec 005: User & Organization Service - Changelog

## [1.1] - 2025-11-06

### Added
- **GitOps Management API endpoints**:
  - `POST /api/v1/organizations/:id/gitops/enable` - Enable GitOps for org
  - `POST /api/v1/organizations/:id/gitops/disable` - Disable GitOps
  - `POST /api/v1/organizations/:id/gitops/sync` - Manual sync trigger
  - `GET /api/v1/organizations/:id/gitops/status` - Sync status
  - `GET /api/v1/organizations/:id/gitops/drift` - Drift detection

- **GitOps Reconciler service**:
  - Background service syncing Git configs every 30 seconds
  - Hybrid mode: Git is source of truth for policy
  - Supports user budgets, allowed models, rate limits via Git
  - Drift logging in audit log

- **Password Reset endpoint**:
  - `POST /api/v1/users/:id/reset-password` (System Admin only)
  - Password complexity validation (12+ chars, mixed case, numbers, symbols)
  - Audit logged

- **Per-user token limit support** in user management

### Changed
- Updated service architecture to include `gitops_reconciler.go`
- Clarified RBAC forward compatibility in implementation notes

## [1.0] - Initial Release
- Authentication (JWT tokens, API keys)
- User and organization CRUD
- API key management
- Audit logging
- RBAC middleware

