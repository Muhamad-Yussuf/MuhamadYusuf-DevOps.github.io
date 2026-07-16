---
layout: post
title: "Standardizing Fragmented Product Infrastructure on GCP"
date: 2026-07-14 09:00:00 +0300
author: Muhamad Yusuf
categories: [Platform Engineering, DevSecOps]
tags: [gcp, terraform, github-actions, workload-identity, pam, iap, secret-manager]
description: How I consolidated inconsistent product infrastructure, created reusable keyless delivery workflows, and introduced secure access and shared DevOps practices across a software house.
image:
  path: /assets/img/posts/gcp-standardization.png
  alt: Standardized GCP infrastructure and keyless delivery platform
---

Software houses often inherit infrastructure one product at a time. Each new application arrives with its own provider, deployment script, credentials, backup method, monitoring assumptions, and monthly bill. The systems may all run, but operating them as a portfolio becomes slow and risky.

That was the starting point when I began supporting Faradah. Applications were distributed across different cloud providers and followed inconsistent infrastructure, CI/CD, access, and cost patterns.

My objective was to create one understandable operating model across products such as Hebaa, WaqfNami, Wdeem, and other internal and client platforms—without forcing every application to become technically identical.

## Standardize the Platform Contract

The first decision was to consolidate the infrastructure foundation on GCP. Standardization did not mean putting every workload in one project or network. It meant giving each product a consistent set of capabilities:

- Terraform-managed project foundations
- Application networking and load balancing
- Compute and managed database patterns
- Public and private asset storage
- Automated backups and retention
- Central secret management
- Monitoring, audit metrics, and security alerts
- Keyless CI/CD identity
- IAP-only administrative connectivity
- Time-bound privileged access

Product-specific scale, domain, data, and availability requirements remained configurable, while the security and delivery baseline stayed consistent.

## Consolidate Without Recreating Everything at Once

A migration from fragmented infrastructure cannot depend on a single cutover. I moved products in controlled stages:

1. Inventory the running application, data, domains, credentials, and deployment process.
2. Recreate the foundation in Terraform on GCP.
3. Establish image build and deployment workflows.
4. Move secrets into Google Secret Manager.
5. Validate backups, monitoring, and administrative access.
6. Run the new environment alongside the old one where required.
7. Cut traffic over only after application and operational checks pass.
8. Retire the old resources after an agreed observation period.

This approach reduced disruption and made each migration a repeatable pattern for the next product.

## Create Central GitHub Actions Workflows

Previously, each repository could implement delivery differently. That creates duplicated YAML, inconsistent permissions, and fixes that must be repeated across many applications.

I created a central workflow repository with reusable GitHub Actions for:

- Building Docker images with layer caching
- Publishing images to Artifact Registry
- Deploying Docker Compose workloads to GCE
- Connecting to private VMs through OS Login and IAP
- Pulling environment configuration from Secret Manager
- Restarting services after controlled configuration updates

Application repositories call these workflows with product-specific inputs. Improvements to authentication, caching, deployment checks, or runtime configuration can therefore be implemented once and inherited across the portfolio.

## Remove Long-Lived Cloud Keys from CI/CD

The central workflows authenticate through GitHub OIDC and Google Workload Identity Federation.

The trust chain is:

1. GitHub Actions issues a short-lived OIDC token for an approved repository and workflow context.
2. Google validates the token through a Workload Identity Pool provider.
3. The workflow impersonates a dedicated service account.
4. That service account can perform only the required build or deployment operations.

No Google service-account JSON key needs to be stored in GitHub. Credentials are short-lived, repository access can be constrained through attribute conditions, and cloud audit logs identify the service account used for each operation.

The same identity builds and pushes the artifact, then uses OS Login and IAP to reach the private deployment target. This keeps SSH keys and broad network exposure out of the repository.

## Make Server Access Private and Temporary

Administrative access followed two controls:

- Firewall rules denied direct public SSH and allowed the IAP proxy path to approved instances.
- Google Privileged Access Manager provided temporary IAM grants for approved administrative tasks.

This changes the default from permanent access to requested access. Engineers receive the permissions they need for a limited period, while the request, grant, and activity remain auditable.

IAP also removes the need to expose SSH broadly or manage a traditional public bastion for every product. Access depends on both identity and IAM authorization, not only possession of a network route.

## Put Secrets and Backups into the Platform

Runtime configuration moved to Google Secret Manager and was pulled during controlled deployments rather than copied manually between servers.

Database and asset protection became part of the Terraform foundation:

- Cloud SQL automated backups
- Configurable retention and backup location
- Dedicated storage for application and legacy backups
- Bucket lifecycle controls
- Repeatable deployment of backup resources

Backups are useful only when ownership, retention, and restoration expectations are clear. Standardizing their deployment made those expectations visible in code and consistent across products.

## Add Cost and Security Feedback

Consolidation created a common view of resource usage and made cost differences easier to compare. I reviewed compute sizing, managed database tiers, storage, networking, and unused resources, then applied reductions where they did not compromise availability.

The infrastructure also generated operational and security signals for events such as:

- Application and infrastructure health failures
- Database pressure
- Changes to firewall and IAM-sensitive resources
- Secret access patterns
- Availability of public endpoints

These controls helped the team detect platform changes instead of discovering them during an application incident.

## The Real Deliverable Was a Shared DevOps Culture

Terraform modules and reusable workflows are only part of the result. The larger change was helping teams work with the platform:

- Developers used one delivery pattern across products.
- Secrets were requested and consumed through a defined process.
- Infrastructure changes were reviewed rather than performed from memory.
- Access became private, temporary, and auditable.
- Backups, monitoring, and cost were discussed during delivery—not after production issues.
- Product teams understood which parts they owned and when to involve DevOps.

My goal in a consulting engagement is not to remain the only person who understands the infrastructure. It is to leave behind a platform and a working culture that make safe delivery easier for everyone.
