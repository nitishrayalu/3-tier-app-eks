# Kubernetes Manifests

This directory contains the Kubernetes manifests for the 3-Tier Web Application.

## Manifest Description

- **namespace.yaml**: Creates the `3-tier-app-eks` namespace.
- **configmap.yaml**: Stores non-sensitive configuration (DB Host, Port).
- **secrets.yaml**: Stores sensitive credentials (DB Password, Username) - *Base64 encoded*.
- **database-service.yaml**: `ExternalName` service mapping RDS endpoint to internal DNS.
- **migration_job.yaml**: Job to initialize database schema.
- **backend.yaml**: Deployment & Service for Flask API.
- **frontend.yaml**: Deployment & Service for React UI.
- **ingress.yaml**: Ingress resource for AWS Application Load Balancer.

## Deployment Order

1. Namespace & Config
2. Database Service
3. Migration Job
4. Backend & Frontend
5. Ingress

For full deployment steps and command references, please see the [Main Project README](../../README.md).
