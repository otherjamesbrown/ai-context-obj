# Spec 009: Admin CLI Tool (ias-admin)

**Status**: In Progress  
**Version**: 1.1  
**Last Updated**: 2025-11-06

## Purpose

Command-line tool for System Admins to manage the platform, bootstrap initial setup, and enable GitOps workflows.

## Dependencies

- **005-user-org-service**: Management APIs

## Success Criteria

- [ ] Bootstrap command creates first admin user
- [ ] All management operations available via CLI
- [ ] Output formats: table, JSON
- [ ] Can be used in scripts/CI/CD
- [ ] Single binary distribution

## Commands

### Bootstrap
```bash
ias-admin bootstrap create-super-admin \
  --email admin@linode.com \
  --password <password>
```

### Organizations
```bash
ias-admin org create --name team-alpha --budget 1000000000
ias-admin org list
ias-admin org get --id <uuid>
ias-admin org update --id <uuid> --budget 2000000000
ias-admin org delete --id <uuid>
```

### Users
```bash
ias-admin user invite \
  --org team-alpha \
  --email user@example.com \
  --role org_admin

ias-admin user list --org team-alpha
ias-admin user delete --id <uuid>

# NEW in v1.1: Password Reset (System Admin only)
ias-admin user reset-password \
  --email user@example.com \
  --new-password <secure-password>
```

### API Keys
```bash
ias-admin key list --org team-alpha
ias-admin key revoke --id <uuid>
```

### Models
```bash
ias-admin model add \
  --name llama3 \
  --display-name "Llama 3 8B" \
  --backend-url http://llama-3-8b.inference:8000 \
  --input-cost 0.001 \
  --output-cost 0.002

ias-admin model list
ias-admin model disable --name llama3
ias-admin model enable --name llama3
```

### Usage
```bash
ias-admin usage --org team-alpha --last 30d
ias-admin usage --org team-alpha --model llama3
```

### GitOps (New in v1.1)
```bash
# Enable GitOps for an organization
ias-admin gitops enable \
  --org team-alpha \
  --repo https://github.com/company/ias-config \
  --path orgs/team-alpha/org.yaml

# Trigger manual sync
ias-admin gitops sync --org team-alpha

# Check sync status
ias-admin gitops status --org team-alpha

# Detect drift (Git vs Database)
ias-admin gitops drift --org team-alpha
```

## Tech Stack

```
Language: Go
Framework: Cobra CLI
Config: Viper (load from env, flags, config file)
Output: tablewriter for tables, JSON for scripts
```

## Project Structure

```
cli/
├── cmd/
│   └── ias-admin/
│       └── main.go
├── internal/
│   ├── commands/
│   │   ├── bootstrap.go
│   │   ├── org.go
│   │   ├── user.go
│   │   ├── key.go
│   │   ├── model.go
│   │   └── gitops.go  # NEW: GitOps commands
│   ├── client/
│   │   └── api_client.go    # HTTP client for APIs
│   └── output/
│       ├── table.go
│       └── json.go
├── Dockerfile
├── Makefile
└── go.mod
```

## Configuration

```yaml
# ~/.ias-admin/config.yaml
api_url: https://api.ias.linode.com
email: admin@linode.com
# JWT token stored after login
auth_token: eyJ...
```

## Contracts

See `contracts/` for:
- `commands.md` - Complete command reference
- `examples.sh` - Example usage scenarios

## Installation

```bash
# Via Go install
go install github.com/linode/ias-admin@latest

# Via release binary
wget https://github.com/linode/ias/releases/download/v1.0.0/ias-admin-linux-amd64
chmod +x ias-admin-linux-amd64
mv ias-admin-linux-amd64 /usr/local/bin/ias-admin
```

## Next Steps

After completion:
- System Admins can manage platform via CLI
- Can be used in automation scripts

