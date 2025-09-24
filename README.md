# CNPG Backup Configuration

Automated CloudNative-PG backup configuration deployment for Kubernetes clusters.

## Overview

This repository contains GitHub Actions workflows to automatically configure PostgreSQL backups using CloudNative-PG operator across all notebook-link Kubernetes environments. It follows the same deployment pattern as Grafana Alloy, deploying to one environment at a time.

## Features

- **Auto-discovery**: Automatically finds all PostgreSQL clusters in each environment
- **S3 Backup**: Configures backups to environment-specific Scaleway S3 buckets
- **WAL Archiving**: Continuous archiving of Write-Ahead Logs for point-in-time recovery
- **Scheduled Backups**: Daily backups at 2 AM UTC
- **Environment Isolation**: Each environment has its own S3 bucket and credentials

## Deployment Flow

### Automatic Deployment (on push to main)
```
dev → qa → prod
```

### Manual Deployment
Use the GitHub Actions UI to trigger deployment to specific environments:
- test
- infra

## S3 Bucket Structure

Each environment has its own S3 bucket:
- `notebook-link-cnpg-backups-dev`
- `notebook-link-cnpg-backups-qa`
- `notebook-link-cnpg-backups-prod`
- `notebook-link-cnpg-backups-test`
- `notebook-link-cnpg-backups-infra`

Within each bucket, backups are organized by namespace:
```
notebook-link-cnpg-backups-{env}/
└── {namespace}/
    ├── base/          # Full backups
    └── wal/           # WAL archives
```

## How It Works

1. **Infrastructure Setup** (managed by notebook-link-infra):
   - Creates S3 buckets for each environment
   - Generates IAM credentials with access to the bucket
   - Stores credentials in Scaleway Secret Manager

2. **Deployment** (this repository):
   - Connects to the environment's Kubernetes cluster
   - Discovers all CloudNative-PG clusters
   - For each cluster:
     - Creates S3 credentials secret in the namespace
     - Patches the cluster to add backup configuration
     - Creates a scheduled backup resource

3. **Backup Configuration Applied**:
   ```yaml
   spec:
     backup:
       barmanObjectStore:
         destinationPath: s3://notebook-link-cnpg-backups-{env}/{namespace}
         endpointURL: https://s3.fr-par.scw.cloud
         wal:
           retention: "7d"
   ```

## Manual Verification

After deployment, verify backups are configured:

```bash
# Connect to cluster
scw k8s kubeconfig install {cluster-id}

# Check scheduled backups
kubectl get scheduledbackups.postgresql.cnpg.io --all-namespaces

# Check backup status
kubectl get backups.postgresql.cnpg.io --all-namespaces

# View S3 bucket contents
scw object bucket list
```

## Restore from Backup

To restore a PostgreSQL cluster from backup:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: restored-cluster
  namespace: target-namespace
spec:
  instances: 3
  bootstrap:
    recovery:
      source: s3-backup
  externalClusters:
    - name: s3-backup
      barmanObjectStore:
        destinationPath: s3://notebook-link-cnpg-backups-{env}/{namespace}
        endpointURL: https://s3.fr-par.scw.cloud
        s3Credentials:
          accessKeyId:
            name: cnpg-s3-credentials
            key: ACCESS_KEY
          secretAccessKey:
            name: cnpg-s3-credentials
            key: SECRET_KEY
```

## Environment Configuration

Each environment's configuration is managed through GitHub environment secrets:
- `TF_MANAGED_K8S_API_SERVER_URL`: Kubernetes API server URL
- `TF_MANAGED_K8S_CA_CERT`: Kubernetes CA certificate
- `TF_MANAGED_K8S_SA_TOKEN`: Service account token
- `TF_MANAGED_SCALEWAY_APP_ACCESS_KEY`: Scaleway access key
- `TF_MANAGED_SCALEWAY_APP_SECRET_KEY`: Scaleway secret key
- `TF_MANAGED_SCALEWAY_PROJECT_ID`: Scaleway project ID
- `TF_MANAGED_SCALEWAY_ORG_ID`: Scaleway organization ID

These secrets are automatically configured by the notebook-link-infra repository.

## Related Repositories

- [notebook-link-infra](https://github.com/notebook-link/notebook-link-infra): Infrastructure configuration
- [notebook-link-grafana-alloy](https://github.com/notebook-link/notebook-link-grafana-alloy): Similar deployment pattern for observability
- [notebook-link-supabase](https://github.com/notebook-link/notebook-link-supabase): Contains the PostgreSQL clusters being backed up

## Support

For issues or questions, please open an issue in this repository.