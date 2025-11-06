# Spec 003: Database Schemas - Changelog

## [1.1] - 2025-11-06

### Added
- **Per-user token limits**: Added `monthly_token_limit` column to `users` table (NULL = no limit)
- **GitOps support tables**:
  - `org_allowed_models` - Per-org model access restrictions (API or GitOps managed)
  - `gitops_sync_state` - GitOps reconciliation state tracking
- **Forward-compatible RBAC tables** (foundation for V2):
  - `permissions` - Granular permission definitions
  - `role_permissions` - Role-to-permission mappings
  - Note: V1 uses hard-coded roles, V2 will support custom roles

### Changed
- Updated scale targets in documentation to align with constitution v1.1

## [1.0] - Initial Release
- Core tables: organizations, users, api_keys, model_mappings
- Audit logging table
- TimescaleDB hypertables for usage tracking
- Supporting tables: refresh_tokens, invite_tokens

