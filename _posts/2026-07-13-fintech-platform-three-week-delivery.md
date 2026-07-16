---
layout: post
title: "Shipping a Fintech Platform in Three Weeks Without Blocking Development"
date: 2026-07-13 09:00:00 +0300
author: Muhamad Yusuf
categories: [Kubernetes, Fintech]
tags: [gke, terraform, helm, argocd, networking, vpn, tailscale, migration]
description: A practical account of building a secure private GKE platform in parallel with a legacy environment and solving Saudi bank connectivity constraints under a three-week production-readiness target.
image:
  path: /assets/img/posts/fintech-production.png
  alt: Fintech platform production delivery and secure Saudi network access
---

During a Saudi fintech engagement, the immediate objective was clear: take a running corporate-finance platform to a production-ready operating model in approximately three weeks.

The difficult part was not only the deadline. Engineering was still developing against the existing environment, the application was composed of Java microservices with Redis and SQL dependencies, and a partner-bank integration could be reached only through approved Saudi network paths.

The platform had to improve without stopping the team that was building the product.

## Start with the Delivery Constraint

Replacing the existing environment in place would have created unnecessary risk. Any networking, container, database, or deployment issue could have blocked the developers while the new platform was still incomplete.

I therefore built the pre-production path in parallel:

- The existing environment remained available to the engineering team.
- Services were containerized and validated incrementally.
- New cloud infrastructure was provisioned independently.
- Deployment patterns were tested before application cutover.
- Data and external dependencies were mapped before traffic moved.
- Rollback remained possible throughout the transition.

This reduced the migration blast radius and allowed infrastructure and application work to continue at the same time.

## Build the Platform in Layers

The target platform used private GKE as the application runtime, with Terraform managing the cloud foundation and Helm managing reusable workload definitions.

The platform layer included:

- Separate pre-production and production foundations
- Private GKE networking and controlled administration through IAP
- Cloud SQL and Redis connectivity
- Workload Identity for Kubernetes-to-GCP authentication
- Google Secret Manager with External Secrets
- Artifact Registry and Cloud Build pipelines
- Argo CD for controlled Kubernetes reconciliation
- Ingress, health checks, autoscaling, logging, metrics, and alerting

The Java services used standardized health probes, resource settings, OpenTelemetry options, secrets, and service definitions. Reusable charts reduced the time required to onboard each microservice and made production differences explicit rather than hidden in copied manifests.

## Separate Application Readiness from Platform Readiness

Under a short deadline, teams often treat a running pod as proof that the platform is ready. I used a broader readiness checklist.

For each service, we checked:

1. Image build and immutable tagging
2. Secret synchronization
3. Database and Redis connectivity
4. Internal service discovery
5. Readiness, liveness, and startup behavior
6. Resource requests and scaling limits
7. External integrations
8. Logs, metrics, and alerts
9. Deployment and rollback through the delivery path

This separated application defects from infrastructure defects and made discussions with the CTO and engineering teams much faster.

## The Bank Integration Constraint

One integration introduced a location-specific networking problem. Required bank endpoints were not available from outside Saudi Arabia, and routing through ordinary GCP public addresses was not sufficient because the external party recognized specific ISP address ranges.

The application needed a controlled Saudi egress path without moving the whole platform or giving every developer unrestricted remote access.

The solution evolved in two stages.

### Phase 1: Controlled Tunnel and OpenVPN Path

A private server in Dammam provided the approved Saudi network path. During the initial phase, a Cloudflare private tunnel provided controlled administrative reachability, while OpenVPN carried the required development and integration traffic through that server.

This unblocked testing quickly, but it still required client configuration and operational care around routing, availability, and support.

### Phase 2: Private Mesh Connectivity

The second phase moved the developer devices and the Saudi server into a private Tailscale network. Application traffic that needed the bank route used a controlled proxy path through the server, while normal internet traffic continued to use each developer's local connection.

Selective routing was important. A full-tunnel VPN would have increased latency, widened the support surface, and made unrelated developer traffic dependent on one server.

The final pattern provided:

- Identity-based membership of the private network
- A stable path through the approved Saudi ISP
- Selective routing only for required endpoints
- Easier onboarding and revocation for developers
- Less dependence on exposed administrative ports
- A clear separation between platform traffic and developer internet traffic

## Protect the New Platform During Cutover

Running the old and new environments in parallel solves availability risk, but it introduces configuration risk. The two systems can diverge if changes are made independently.

I reduced this risk by treating the new repositories as the source of truth:

- Terraform represented cloud infrastructure.
- Helm represented reusable application configuration.
- Cloud Build produced deployment artifacts.
- Argo CD represented the desired Kubernetes state.
- Secret Manager remained the source for sensitive runtime configuration.

Changes required during the migration were added to the new operating model rather than applied only as one-off fixes.

## Production Readiness Is a Team Outcome

The short delivery window required frequent communication with engineering teams and the CTO. Infrastructure decisions depended on application startup behavior, integration requirements, database expectations, and the order in which services could be validated.

The most useful DevOps work in this situation was not simply building GKE. It was creating a shared delivery process:

- Developers understood how images and configurations moved between environments.
- Technical leadership had visibility into readiness risks and dependencies.
- Infrastructure changes were reproducible.
- Network constraints were translated into an architecture the team could operate.
- Migration work did not stop feature development.

The result was a production-ready platform delivered under a tight timeline, with secure cloud foundations, repeatable releases, and a workable path for partner-bank integration testing.

The broader lesson is that speed and control are not opposites. Parallel migration, reusable infrastructure, clear readiness criteria, and close collaboration make it possible to move quickly without turning the final cutover into an experiment.
