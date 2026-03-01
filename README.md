# blackroad-terraform 🛣️🏗️

> **Infrastructure as Code** — Terraform modules and configurations powering the BlackRoad OS production stack: Cloudflare DNS, Railway services, GitHub org wiring, TLS, and billing infrastructure.

[![License: Proprietary](https://img.shields.io/badge/License-Proprietary-black.svg)](./LICENSE)
[![Terraform](https://img.shields.io/badge/Terraform-%3E%3D1.5-purple.svg)](https://www.terraform.io/)
[![Stripe](https://img.shields.io/badge/Stripe-Billing-blueviolet.svg)](https://stripe.com/)
[![npm](https://img.shields.io/badge/npm-%40blackroad%2Fsdk-red.svg)](https://www.npmjs.com/package/@blackroad/sdk)
[![Status: Production](https://img.shields.io/badge/Status-Production-brightgreen.svg)](#)

---

## Table of Contents

1. [Overview](#overview)
2. [Repository Structure](#repository-structure)
3. [Prerequisites](#prerequisites)
4. [Quick Start](#quick-start)
5. [npm Integration](#npm-integration)
6. [Stripe Billing Integration](#stripe-billing-integration)
7. [Terraform Providers](#terraform-providers)
8. [Environments](#environments)
9. [Domain Topology](#domain-topology)
10. [E2E Testing](#e2e-testing)
11. [CI/CD](#cicd)
12. [Security](#security)
13. [Related Repositories](#related-repositories)
14. [Support](#support)
15. [License](#license)

---

## Overview

`blackroad-terraform` is the **single source of truth** for all Terraform-managed infrastructure at BlackRoad OS, Inc. It provisions and manages:

- 🌐 **Cloudflare** — DNS records, zone configuration, and Pages deployments across 16+ domains
- 🚂 **Railway** — Container service definitions for all BlackRoad OS microservices
- 🐙 **GitHub** — Organization runner registration and repository auto-wiring
- 💳 **Stripe** — Subscription billing products and webhook configuration
- 🔒 **TLS** — Certificate provisioning and rotation
- 📦 **npm** — Package registry integration via `@blackroad/sdk`

**What this repo is NOT:**

- ❌ Application source code — belongs in individual service repos
- ❌ Secret storage — secrets live in Railway, GitHub Secrets, or a vault
- ❌ Binary or media assets

---

## Repository Structure

```
blackroad-terraform/
├── environments/               # Per-environment Terraform roots
│   ├── dev/
│   │   ├── main.tf
│   │   ├── services.tf
│   │   └── backend.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   ├── services.tf
│   │   └── backend.tfvars
│   └── prod/
│       ├── main.tf
│       ├── services.tf
│       └── backend.tfvars
├── modules/                    # Reusable Terraform modules
│   ├── cloudflare-dns/         # DNS record management
│   ├── railway-service/        # Railway service provisioning
│   ├── github-repo/            # GitHub repository wiring
│   ├── stripe-product/         # Stripe product and price creation
│   └── tls-cert/               # TLS certificate management
├── stripe/                     # Stripe billing configuration
│   └── stripe-config.json      # Product tiers and webhook events
├── npm/                        # npm package publishing configuration
│   └── .npmrc.example          # Registry and auth configuration
├── tests/                      # E2E and validation tests
│   ├── e2e/                    # End-to-end infrastructure tests
│   └── unit/                   # Module unit tests
├── docs/                       # Extended documentation
│   ├── providers.md            # Provider version matrix
│   ├── stripe-setup.md         # Stripe integration guide
│   └── npm-setup.md            # npm publishing guide
├── LICENSE
└── README.md
```

---

## Prerequisites

### Required Tools

| Tool | Minimum Version | Install |
|------|----------------|---------|
| [Terraform](https://developer.hashicorp.com/terraform/install) | `>= 1.5.0` | `brew install terraform` |
| [Node.js](https://nodejs.org/) | `>= 18.0.0` | `brew install node` |
| [npm](https://www.npmjs.com/) | `>= 9.0.0` | Bundled with Node.js |

### Required Environment Variables

```bash
# Cloudflare
export CLOUDFLARE_API_TOKEN="<your-token>"
export CLOUDFLARE_ZONE_ID="<zone-id>"

# Railway
export RAILWAY_TOKEN="<your-token>"
export TF_VAR_railway_project_id="<project-id>"

# Stripe
export STRIPE_SECRET_KEY="sk_live_..."
export STRIPE_WEBHOOK_SECRET="whsec_..."

# GitHub
export GITHUB_TOKEN="<pat-with-org-scope>"

# npm (for publishing @blackroad/sdk)
export NPM_TOKEN="<npm-automation-token>"
```

> ⚠️ **Never commit secrets.** Use Railway environment variables, GitHub Actions secrets, or a secrets manager for all credentials.

---

## Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/BlackRoad-OS/blackroad-terraform.git
cd blackroad-terraform

# 2. Install npm dependencies (for test tooling and SDK publishing)
npm install

# 3. Initialize Terraform for your target environment
cd environments/prod
terraform init -backend-config=backend.tfvars

# 4. Review the execution plan
terraform plan

# 5. Apply infrastructure changes
terraform apply
```

---

## npm Integration

BlackRoad OS publishes the official SDK to npm under the `@blackroad` scope.

### Installing the SDK

```bash
npm install @blackroad/sdk
```

### Usage

```typescript
import { BlackRoadClient } from '@blackroad/sdk';

const client = new BlackRoadClient({
  apiKey: process.env.BLACKROAD_API_KEY,
  environment: 'production', // 'production' | 'staging' | 'development'
});
```

### Publishing (Maintainers Only)

```bash
# Authenticate with npm
npm login --scope=@blackroad --registry=https://registry.npmjs.org

# Publish a new version
npm publish --access public
```

The `.npmrc.example` in `npm/` provides the recommended registry configuration. Copy it to your project root as `.npmrc` and populate `NPM_TOKEN`.

---

## Stripe Billing Integration

BlackRoad OS uses Stripe for subscription billing across all products. Terraform manages Stripe products, prices, and webhook endpoints via the [Stripe Terraform provider](https://registry.terraform.io/providers/lukasaron/stripe/latest).

### Subscription Tiers

| Plan | Price | Interval | Description |
|------|-------|----------|-------------|
| **Basic** | $9 / month | Monthly | Core infrastructure access |
| **Pro** | $29 / month | Monthly | Full platform access |
| **Enterprise** | $99 / month | Monthly | Unlimited scale + SLA |

### Webhook Events

The following Stripe webhook events are handled by the BlackRoad OS API:

| Event | Description |
|-------|-------------|
| `checkout.session.completed` | New subscription initiated |
| `customer.subscription.created` | Subscription record created |
| `customer.subscription.updated` | Plan change or renewal |
| `invoice.paid` | Successful payment received |

### Stripe Configuration

See `stripe/stripe-config.json` for the full product and webhook configuration. To apply Stripe resources:

```bash
cd environments/prod
terraform apply -target=module.stripe-product
```

### Environment Keys

| Environment | Key Prefix |
|-------------|-----------|
| Production | `sk_live_...` |
| Staging / Test | `sk_test_...` |

> Always use `sk_test_` keys in `dev` and `staging` environments.

---

## Terraform Providers

| Provider | Purpose | Version Constraint |
|----------|---------|-------------------|
| [cloudflare/cloudflare](https://registry.terraform.io/providers/cloudflare/cloudflare) | DNS, zones, Pages | `~> 4.0` |
| [railwayapp/railway](https://registry.terraform.io/providers/railwayapp/railway) | Container services | `~> 0.4` |
| [integrations/github](https://registry.terraform.io/providers/integrations/github) | Org runners, repos | `~> 6.0` |
| [lukasaron/stripe](https://registry.terraform.io/providers/lukasaron/stripe) | Billing products | `~> 1.0` |
| [hashicorp/tls](https://registry.terraform.io/providers/hashicorp/tls) | Certificate generation | `~> 4.0` |
| [hashicorp/null](https://registry.terraform.io/providers/hashicorp/null) | Utility resources | `~> 3.0` |

---

## Environments

| Environment | Branch | Purpose | Auto-Deploy |
|-------------|--------|---------|-------------|
| `dev` | `develop` | Local development and feature testing | No |
| `staging` | `staging` | Pre-production validation | On merge |
| `prod` | `main` | Live production systems | On merge + approval |

### Applying an Environment

```bash
cd environments/<env>
terraform init -backend-config=backend.tfvars
terraform plan -out=tfplan
terraform apply tfplan
```

---

## Domain Topology

Terraform manages DNS records for all BlackRoad OS domains via Cloudflare:

| Service | Pages URL | Custom Domain |
|---------|-----------|---------------|
| Web | `blackroad-os-web.pages.dev` | `blackroad.systems` |
| API | `blackroad-os-api.pages.dev` | `api.blackroad.systems` |
| Operator | `blackroad-os-operator.pages.dev` | `operator.blackroad.systems` |
| Prism Console | `blackroad-os-prism-console.pages.dev` | `console.blackroad.systems` |
| Docs | `blackroad-os-docs.pages.dev` | `docs.blackroad.systems` |
| Brand | `blackroad-os-brand.pages.dev` | `brand.blackroad.systems` |
| Research | `blackroad-os-research.pages.dev` | `research.blackroad.systems` |
| Infra | `blackroad-os-infra.pages.dev` | `infra.blackroad.systems` |
| Core | `blackroad-os-core.pages.dev` | `core.blackroad.systems` |

**Additional Domains Under Management:**

- `blackroad.io`, `blackroad.me`, `blackroad.network`
- `blackroadai.com`, `blackroadqi.com`, `blackroadquantum.com`
- `blackroadinc.us`
- `aliceqi.com`, `lucidia.earth`, `lucidiaqi.com`, `lucidia.studio`

---

## E2E Testing

End-to-end infrastructure tests validate that Terraform plans apply cleanly and that provisioned resources are reachable.

### Running E2E Tests

```bash
# Install test dependencies
npm install

# Run all E2E tests
npm run test:e2e

# Run unit tests for Terraform modules
npm run test:unit

# Validate Terraform formatting and syntax
terraform fmt -check -recursive
terraform validate
```

### Test Coverage

| Test Suite | What It Validates |
|------------|------------------|
| `tests/e2e/dns.test.ts` | Cloudflare DNS records resolve correctly |
| `tests/e2e/stripe.test.ts` | Stripe products and prices exist in live account |
| `tests/e2e/npm.test.ts` | `@blackroad/sdk` is published and installable |
| `tests/e2e/railway.test.ts` | Railway services are healthy and responding |
| `tests/unit/modules/*.test.ts` | Each Terraform module validates its inputs |

### Pre-PR Checklist

Before opening a pull request:

- [ ] `terraform fmt -check -recursive` passes
- [ ] `terraform validate` passes for all environments
- [ ] `npm run test:e2e` passes against staging
- [ ] No secrets committed (run `git-secrets` or equivalent)
- [ ] Plan reviewed by a second team member for `prod` changes

---

## CI/CD

GitHub Actions workflows automate plan, validate, and apply operations:

| Workflow | Trigger | Action |
|----------|---------|--------|
| `terraform-plan.yml` | Pull Request | `terraform plan` for all environments |
| `terraform-apply-staging.yml` | Merge to `staging` | Auto-apply staging environment |
| `terraform-apply-prod.yml` | Merge to `main` | Plan + manual approval → apply prod |
| `e2e-tests.yml` | Nightly + PR | Run full E2E test suite |

---

## Security

- **Never commit secrets** — use Railway, GitHub Secrets, or a vault
- Rotate `CLOUDFLARE_API_TOKEN` and `RAILWAY_TOKEN` every 90 days
- Stripe keys: `sk_live_` in prod only; use `sk_test_` everywhere else
- `NPM_TOKEN` should be an automation token scoped to `@blackroad` only
- Report vulnerabilities to [security@blackroad.io](mailto:security@blackroad.io)

---

## Related Repositories

| Repository | Purpose |
|------------|---------|
| [blackroad-os-infra](https://github.com/BlackRoad-OS/blackroad-os-infra) | Broader infra: Kubernetes, Ansible, Helm, Cloudflare Workers |
| [BlackRoad-Public](https://github.com/BlackRoad-OS/BlackRoad-Public) | Public SDKs, docs, and specifications |
| [blackroad-prism-console](https://github.com/BlackRoad-OS/blackroad-prism-console) | Unified management interface (Prism) |
| [blackroad-workspace](https://github.com/BlackRoad-OS/blackroad-workspace) | Workspace management |
| [developers-blackroad-io](https://github.com/BlackRoad-OS/developers-blackroad-io) | Developer portal at developers.blackroad.io |

---

## Support

- **Issues**: [Open an issue](https://github.com/BlackRoad-OS/blackroad-terraform/issues) in this repository
- **Infrastructure emergencies**: [infrastructure@blackroad.io](mailto:infrastructure@blackroad.io)
- **Security vulnerabilities**: [security@blackroad.io](mailto:security@blackroad.io)
- **Developer portal**: [developers.blackroad.io](https://developers.blackroad.io)

---

## License

© 2024–2026 BlackRoad OS, Inc. All Rights Reserved.  
Founder, CEO & Sole Stockholder: Alexa Louise Amundson  
See [LICENSE](./LICENSE) for full terms.

---

*Maintained by the BlackRoad OS Infrastructure Team · [blackroad.systems](https://blackroad.systems)*
