# FlowForge GitOps

GitOps repository for deploying and managing the **FlowForge** platform on Kubernetes using **Argo CD**.

This repository contains the Kubernetes manifests, Helm configuration, and Argo CD applications required to continuously deploy FlowForge services and their supporting infrastructure.

## Architecture

```text
                         GitHub
                           │
                           │ Git
                           ▼
                    ┌─────────────┐
                    │   Argo CD   │
                    └──────┬──────┘
                           │
                           │ Sync
                           ▼
                    ┌─────────────┐
                    │ Kubernetes  │
                    │    / k3s    │
                    └──────┬──────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
      FlowForge          Kafka          PostgreSQL
        API             Broker           Database
          │                │                │
          └────────────────┴────────────────┘
                           │
                           ▼
                       Job Queue
```

## Repository Structure

```text
flowforge-gitops/
├── apps/
│   ├── flowforge/
│   ├── kafka/
│   └── postgresql/
│
├── argocd/
│   └── applications/
│
├── infrastructure/
│   └── ...
│
└── README.md
```

> The exact structure may evolve as additional platform components are introduced.

## GitOps Workflow

FlowForge follows a GitOps deployment model.

```text
Developer
   │
   │ Push changes
   ▼
GitHub
   │
   │ Argo CD watches repository
   ▼
Argo CD
   │
   │ Reconcile desired state
   ▼
Kubernetes
   │
   ├── API
   ├── Workers
   ├── Kafka
   └── PostgreSQL
```

The Git repository represents the **desired state** of the cluster.

When a manifest or Helm configuration changes:

1. Changes are committed and pushed to GitHub.
2. Argo CD detects the change.
3. Argo CD compares the desired state with the cluster state.
4. The required Kubernetes resources are synchronized.
5. Kubernetes reconciles the workloads.

This avoids manually applying application manifests directly to the cluster.

## Applications

### FlowForge API

The API service exposes the FlowForge backend and handles requests from clients.

Responsibilities include:

* API endpoints
* Authentication
* Job creation
* Job status
* Communication with platform services

### FlowForge Worker

Workers consume jobs from Kafka and process them asynchronously.

Example workflow:

```text
Client
  │
  ▼
FlowForge API
  │
  │ publish job
  ▼
Kafka
  │
  │ consume
  ▼
FlowForge Worker
  │
  ▼
Job processing
```

### Kafka

Kafka provides asynchronous communication between FlowForge services.

The primary job-processing topic is:

```text
flowforge.jobs
```

Workers consume messages using a dedicated consumer group.

### PostgreSQL

PostgreSQL stores persistent application state, including job metadata and related application data.

Database access is handled by the FlowForge backend and worker services rather than exposing PostgreSQL directly to the public internet.

## Continuous Delivery

Application images are built outside this repository and published to a container registry.

The GitOps repository defines which image version Kubernetes should run.

For example:

```yaml
image:
  repository: ghcr.io/rainzler-exagone/flowforge-api
  tag: <version>
```

Updating the image reference in Git creates a new desired state.

Argo CD then reconciles the cluster with that state.

## Secrets

**Secrets must not be committed to this repository.**

Do not commit:

```text
.env
*.secret.yaml
database passwords
JWT secrets
private keys
cloud credentials
registry credentials
```

For local development, use environment-specific secret mechanisms.

For production deployments, use a dedicated secret-management solution such as:

* Kubernetes Secrets with appropriate access controls
* External Secrets Operator
* AWS Secrets Manager
* HashiCorp Vault
* Another approved secret-management system

## Deployment

The cluster must have Argo CD installed and configured to access this repository.

Once the Argo CD applications are configured, deployments are managed through Git.

A typical deployment flow is:

```bash
git add .
git commit -m "Update FlowForge deployment"
git push
```

Argo CD detects the repository change and synchronizes the cluster.

## Local Validation

Before committing Kubernetes configuration, validate manifests locally where possible.

For example:

```bash
kubectl apply --dry-run=client -f <manifest>
```

For Helm-based applications:

```bash
helm lint <chart>
```

Inspect generated manifests with:

```bash
helm template <release> <chart>
```

## Security Principles

The repository follows several basic security principles:

* No credentials committed to Git.
* Kubernetes services are exposed only when required.
* Database and internal infrastructure should remain private.
* Container images should use immutable version references where practical.
* Kubernetes RBAC should follow least privilege.
* Production secrets should be managed externally.
* Infrastructure changes should be reviewed through Git.
* Publicly exposed services should use TLS.

## Environments

Environment-specific configuration should be separated rather than embedding production values directly into shared manifests.

A typical future structure could be:

```text
environments/
├── development/
├── staging/
└── production/
```

This allows the same application definitions to be promoted between environments while keeping environment-specific configuration separate.

## Deployment Philosophy

FlowForge uses the following principles:

**Infrastructure as Code**

Infrastructure is defined declaratively and managed through Terraform.

**GitOps**

Kubernetes desired state is stored in Git and reconciled by Argo CD.

**Immutable Deployments**

Application deployments reference specific container image versions rather than relying on mutable `latest` tags.

**Separation of Concerns**

Application source code, infrastructure, and deployment configuration are maintained separately.

**Least Privilege**

Services and Kubernetes components should receive only the permissions they require.

## Related Repositories

* **FlowForge application:** application source code and services
* **FlowForge GitOps:** Kubernetes deployment configuration
* **Infrastructure:** Terraform configuration for cloud infrastructure

## Status

FlowForge GitOps is an actively developed project.

Current focus:

* Kubernetes deployment
* Argo CD GitOps workflow
* Kafka-based asynchronous processing
* PostgreSQL persistence
* Containerized FlowForge services

Planned improvements include:

* Centralized logging and monitoring
* Distributed tracing
* Production-grade secret management
* Environment promotion
* Automated image updates
* High-availability infrastructure
* Backup and disaster recovery
