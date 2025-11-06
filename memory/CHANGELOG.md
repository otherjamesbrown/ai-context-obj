# Constitution Changelog

All notable changes to the constitutional principles will be documented here.

**For detailed update summaries**, see `/memory/updates/` directory.

## [1.1] - 2025-11-06

### Added
- **GitOps Management (Hybrid Mode)** - Complete section on declarative org configuration
  - Git-managed vs API-managed configurations
  - Reconciliation strategy (30-second sync cycles)
  - Example YAML repository structure
  - Support for user budgets, rate limits, allowed models via Git

- **CLI-Only Features (V1)** section
  - Password reset capability for System Admins via CLI
  - Command specifications and audit requirements

- **Forward-Compatible RBAC**
  - Updated RBAC section to clarify V1 uses 3 fixed roles
  - Noted permission system designed for future custom roles
  - Database schema will use permission-based model internally

### Changed
- **Scale Targets** - Updated from 100 orgs to more realistic V1 targets:
  - 1,000 organizations (was: 100)
  - 100 users per organization = 100,000 total users (was: 1,000)
  - 10 API keys per user = 1,000,000 total keys (was: 10,000)
  - Noted no hard limits, system designed to scale horizontally

- **Out of Scope** items clarified:
  - Custom role creation noted as V2 with schema forward-compatibility
  - Password reset changed to "Web UI password reset" (CLI available in V1)

- **API-First Design** - Added note about GitOps declarative management option

## [1.0] - Initial Release
- Initial constitutional principles established
- Core architectural mandates defined
- Technology stack specified
- RBAC model with 3 fixed roles

