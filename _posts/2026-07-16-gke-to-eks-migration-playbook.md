---
layout: post
title: "From Private GKE to AWS EKS: A Practical Migration Playbook"
date: 2026-07-15 09:00:00 +0300
author: Muhamad Yusuf
categories: [Kubernetes, Cloud]
tags: [gke, eks, terraform, helm, argocd, devsecops, migration]
description: A practical approach to building a production Kubernetes platform on GCP and migrating its environments to AWS EKS without sacrificing repeatability, security, or operational control.
image:
  path: /assets/img/og.png
  alt: GKE to AWS EKS Kubernetes migration playbook
---

Moving Kubernetes workloads between cloud providers is not primarily a cluster migration. The difficult part is rebuilding the platform contracts around the cluster: identity, secrets, networking, ingress, delivery, observability, and operational ownership.

This article distills the approach I used after first building and operating a multi-service platform on private GKE, then leading the migration of its environments to AWS EKS. The goal here is not to present a one-click recipe. It is to show the decisions that made the migration controlled, repeatable, and supportable after cutover.

## The Starting Point

The source platform was already running containerized services on GCP. Its foundation included:

- Private GKE networking
- Terraform-managed cloud infrastructure
- Reusable Helm deployment patterns
- CI pipelines with security scanning
- Google Apigee for API exposure and policy enforcement
- Centralized secrets and workload-level access controls
- Separate environments with different release expectations

The target was not simply "the same YAML on EKS." It needed an AWS-native operating model with EKS, Argo CD, AWS load balancing, workload IAM roles, and External Secrets integration.

## Translate Capabilities, Not Resource Names

A cloud-to-cloud migration becomes easier when the design starts with capabilities rather than individual services.

| Platform concern | GCP implementation | AWS implementation |
| --- | --- | --- |
| Kubernetes control plane | Private GKE | EKS |
| Workload identity | GKE Workload Identity | IRSA or EKS Pod Identity |
| Secrets | Google Secret Manager | AWS Secrets Manager or Parameter Store |
| Ingress | GKE-integrated load balancing | AWS Load Balancer Controller |
| Application packaging | Helm | Helm |
| Deployment control | CI-driven releases | Argo CD GitOps |
| Infrastructure | Terraform | Terraform |

Keeping Terraform and Helm as stable interfaces reduced unnecessary application changes. The provider-specific details changed underneath those interfaces, while application teams continued to work with familiar deployment inputs.

## Phase 1: Inventory the Real Dependencies

Before provisioning EKS, I mapped every dependency that could block a workload from becoming healthy:

1. Container registries and image-pull permissions
2. Ingress rules, certificates, and DNS ownership
3. Persistent storage requirements
4. Database and third-party network paths
5. Secrets and their refresh behavior
6. Service accounts and cloud API permissions
7. Scheduled jobs and background workers
8. Health checks, resource requests, and autoscaling assumptions
9. Logs, metrics, alerts, and on-call ownership

This inventory exposed hidden coupling early. A deployment can be syntactically valid while still failing because a workload identity, private route, DNS record, or secret refresh policy was not recreated correctly.

## Phase 2: Build the EKS Foundation First

The target cluster had to be operational before application migration started. The platform layer included:

- VPC and private subnet design
- EKS control plane and worker capacity
- Container registry access
- AWS Load Balancer Controller
- DNS and certificate integration
- Cluster autoscaling and workload scheduling rules
- Central logging and metrics
- Argo CD for deployment reconciliation
- External Secrets for application configuration

Terraform owned cloud resources and cluster prerequisites. Kubernetes add-ons were then installed in a controlled order so that application releases did not race against missing controllers or permissions.

## Phase 3: Rebuild Identity Deliberately

Identity is one of the highest-risk parts of the migration. Copying broad node permissions into AWS would have made the platform work quickly, but it would also have created a weak security boundary.

Instead, each workload received only the AWS permissions it required. Kubernetes service accounts were mapped to dedicated IAM roles through IRSA or EKS Pod Identity. This preserved the same core principle used on GKE: applications authenticate through workload identity rather than static cloud credentials.

A useful rule is:

> If two services have different operational responsibilities, they should not automatically share the same cloud role.

This makes permission reviews, incident analysis, and future service ownership much clearer.

## Phase 4: Keep Secrets Outside Git

Application manifests should reference secrets, not contain them. External Secrets provided a consistent Kubernetes-facing interface while the backing provider changed from Google Secret Manager to AWS Secrets Manager or Parameter Store.

A simplified pattern looks like this:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payments-api
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets
    kind: ClusterSecretStore
  target:
    name: payments-api-runtime
  data:
    - secretKey: database-url
      remoteRef:
        key: production/payments-api
        property: database_url
```

The important details are not only the secret paths. The platform must also define rotation ownership, refresh intervals, failure alerts, and what should happen when a secret is unavailable during deployment.

## Phase 5: Separate Build from Deployment

The source platform used CI pipelines to build and release applications. During the migration, I separated these responsibilities:

- CI builds, tests, scans, and publishes immutable images.
- Git records the desired deployment version and environment configuration.
- Argo CD reconciles the cluster against that desired state.

This reduced cloud-specific logic inside application pipelines and made deployments auditable. It also enabled different promotion policies: automatic reconciliation in lower environments and controlled approval for production.

Helm remained the packaging layer, with environment-specific values kept separate from reusable templates. The same chart could therefore support multiple environments without copying entire manifest trees.

## Phase 6: Migrate in Small, Verifiable Units

The migration sequence followed service dependencies rather than repository order:

1. Platform controllers and observability
2. Stateless internal services
3. Public services and ingress paths
4. Scheduled and asynchronous workloads
5. Services with persistent or external dependencies
6. Production traffic and final DNS cutover

For each workload, the verification checklist covered:

- Pod readiness and restart behavior
- Secret synchronization
- Cloud API authorization
- Internal service discovery
- External connectivity
- Ingress health checks
- Logs, metrics, and alerts
- Rollback to the previous environment

Running both platforms during the transition provided a safer rollback path than treating migration day as a one-way event.

## Common Failure Modes

### Treating EKS as GKE with different resource names

Controllers, identities, load balancers, storage classes, and network behavior differ. The application YAML may be portable, but the platform contracts are not automatically portable.

### Giving permissions to nodes instead of workloads

This can make early testing pass while creating excessive access and weak auditability. Workload-level IAM should be part of the foundation, not a later hardening task.

### Delaying observability until after cutover

A green deployment is not enough. Before production traffic moves, teams need logs, metrics, alerts, dashboards, and clear on-call ownership.

### Allowing environment configuration to drift

Manual cluster changes are difficult to reproduce and easy to forget. Terraform, Helm, and Argo CD should remain the source of truth throughout the migration.

### Migrating every service at once

Large cutovers make diagnosis difficult. Smaller migration units create cleaner rollback boundaries and faster feedback.

## What Made the Migration Sustainable

The migration succeeded as a platform transition because a few principles stayed consistent:

- Infrastructure remained reproducible through Terraform.
- Applications stayed packaged through reusable Helm patterns.
- CI produced immutable, scanned artifacts.
- Argo CD controlled deployment state.
- Workloads used dedicated cloud identities.
- Secrets remained outside repositories and pipelines.
- Observability and rollback were designed before production cutover.

The key lesson is simple: do not migrate only the cluster. Migrate the operating model around it. That is what turns a collection of running workloads into a production platform that teams can safely maintain.

If you are working through a GKE-to-EKS migration or standardizing Kubernetes delivery across clouds, feel free to connect with me on [LinkedIn](https://www.linkedin.com/in/muhamad-devops/).
