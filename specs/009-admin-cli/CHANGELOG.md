# Spec 009: Admin CLI Tool - Changelog

## [1.1] - 2025-11-06

### Added
- **Password reset command**:
  ```bash
  ias-admin user reset-password \
    --email user@example.com \
    --new-password <secure-password>
  ```
  - System Admin only
  - Calls new password reset API endpoint
  
- **GitOps commands**:
  ```bash
  ias-admin gitops enable    # Enable GitOps for an org
  ias-admin gitops sync      # Trigger manual sync
  ias-admin gitops status    # Check sync status
  ias-admin gitops drift     # Detect configuration drift
  ```

### Changed
- Added `gitops.go` to internal commands structure
- Updated examples to include new v1.1 features

## [1.0] - Initial Release
- Bootstrap commands
- Organization management
- User management
- API key management
- Model management
- Usage reporting

