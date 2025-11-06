# Spec 001: Infrastructure Provisioning

## Purpose

Provision the Linode LKE cluster, GPU/CPU node pools, and managed PostgreSQL database using Terraform. This is the foundation infrastructure for all services.

## Dependencies

None - this is infrastructure foundation.

## Success Criteria

- [ ] LKE cluster running with CPU and GPU node pools
- [ ] Managed PostgreSQL database provisioned
- [ ] Terraform state managed securely
- [ ] Network policies configured
- [ ] Namespaces created (dev, staging, production)
- [ ] kubectl access configured

## Technical Scope

### Included
- LKE cluster provisioning (Terraform)
- CPU node pool (for services)
- GPU node pool (for vLLM)
- Managed PostgreSQL database
- Network configuration
- K8s namespaces

### Excluded
- Application deployments (Helm charts in other specs)
- Service-specific configuration
- ArgoCD setup (spec 011)

## Architecture

### LKE Cluster
- **Region**: Choose based on GPU availability
- **K8s Version**: 1.28+
- **HA Control Plane**: Yes

### Node Pools
1. **CPU Pool** (for services)
   - Node type: Linode 8GB (or appropriate)
   - Min nodes: 3
   - Max nodes: 10
   - Autoscaling: Enabled

2. **GPU Pool** (for vLLM)
   - Node type: GPU instance (e.g., RTX 6000 Ada)
   - Min nodes: 1
   - Max nodes: 3
   - GPU: NVIDIA GPU with CUDA support

### Managed Database
- **Type**: PostgreSQL 15
- **Plan**: Dedicated 4GB or higher
- **High Availability**: Optional (recommended for production)
- **Backups**: Daily automated backups

### Namespaces
```yaml
namespaces:
  - ias-dev
  - ias-staging  
  - ias-production
  - ias-system  # For infrastructure (RabbitMQ, Redis, monitoring)
```

## Terraform Structure

```
infrastructure/terraform/
├── main.tf              # Provider configuration
├── lke.tf               # LKE cluster
├── node-pools.tf        # CPU and GPU pools
├── database.tf          # Managed PostgreSQL
├── network.tf           # Network policies
├── namespaces.tf        # K8s namespaces
├── variables.tf         # Input variables
├── outputs.tf           # Output values
└── terraform.tfvars     # Environment-specific values (gitignored)
```

## Contracts

See `contracts/terraform/` for complete Terraform configuration.

## Configuration

### Required Variables
- `linode_token` - Linode API token
- `region` - Linode region (e.g., "us-east")
- `cluster_name` - Name for LKE cluster
- `postgres_password` - Database password (use secret manager)

## Testing

### Validation
- `terraform plan` shows expected resources
- `terraform apply` succeeds
- `kubectl get nodes` shows nodes ready
- Database connection successful

## Next Steps

After infrastructure is provisioned:
1. Deploy **002-local-dev-environment** (for developers)
2. Run **003-database-schemas** migrations against managed PostgreSQL
3. Deploy services using Helm charts

