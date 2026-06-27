# NexusBridge Consulting — AWS Account Structure

## Overview

NexusBridge operates a multi-account AWS environment using AWS Organizations with the following account structure:

```
AWS Organizations (nexusbridge.co.uk)
│
├── Management Account (root)
│   └── IAM Identity Center (SSO)
│   └── CloudTrail (org trail)
│   └── GuardDuty (delegated admin)
│
├── Security Account
│   └── Audit logging
│   └── GuardDuty findings
│   └── AWS Config aggregator
│   └── KMS key vault
│   └── Incident response tools
│
├── Production Account
│   └── Client-facing workloads
│   └── Production databases
│   └── Customer data
│
├── Nonproduction Account
│   └── Development
│   └── Staging
│   └── Testing environments
│
└── Sandbox Account
    └── Employee experimentation
    └── Proof-of-concept work
    └── Training
```

## Account Details

| Account | Purpose | OU | Access Model |
|---------|---------|----|-------------|
| **Management** | Org root, billing, SSO, org-wide logging | Root | Break-glass only (SSO admins) |
| **Security** | Audit logs, security tooling, KMS | Security OU | IAM Identity Center + break-glass roles |
| **Production** | Production client workloads | Workloads OU | IAM Identity Center (just-in-time) |
| **Nonproduction** | Dev, staging, test environments | Workloads OU | IAM Identity Center (full access) |
| **Sandbox** | Employee experimentation | Sandbox OU | IAM Identity Center (self-service) |

## Identity Federation Model

```
┌──────────────┐     SAML/OIDC     ┌────────────────────┐
│  Entra ID    │◄────────────────►│  IAM Identity       │
│  (Corporate) │                   │  Center             │
│  HR, Finance,│                   │  (Engineering, IT)  │
│  Legal, Exec │                   │                     │
└──────┬───────┘                   └─────────┬───────────┘
       │                                      │
       │                             ┌────────┴────────┐
       │                             │   Permission     │
       │                             │   Sets           │
       │                             │   & Groups       │
       │                             └─────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────┐
│              AWS Accounts                         │
│  (Management, Security, Prod, Nonprod, Sandbox)  │
└──────────────────────────────────────────────────┘
```

## SSO Permission Sets (IAM Identity Center)

| Permission Set | Accounts | Assignment |
|---------------|----------|-----------|
| `AdministratorAccess` | Management | Break-glass (MFA enforced, approved) |
| `SecurityAudit` | Security, All | SOC team |
| `IAMAdmin` | Security, Management | IAM team |
| `InfraAdmin` | Nonproduction | Platform Engineering |
| `InfraReadOnly` | Production | Platform Engineering |
| `DeveloperPowerUser` | Nonproduction | App Developers |
| `DeveloperReadOnly` | Production | App Developers |
| `BillingView` | Management | Finance team |
| `SupportUser` | All | IT Support (limited) |

## Network Architecture (High-Level)

```
              Internet
                 │
            ┌────┴────┐
            │  AWS WAF │
            └────┬────┘
                 │
         ┌───────┴───────┐
         │               │
    ┌────┴────┐    ┌────┴────┐
    │  Public │    │ Private │
    │  Subnets│    │ Subnets │
    └─────────┘    └─────────┘
         │               │
         │         ┌─────┴──────┐
         │         │  VPC       │
         │         │  Endpoints │
         │         └─────┬──────┘
         │               │
    ┌────┴────┐          │
    │  ALB /  │          │
    │  NLB    │          │
    └─────────┘          │
         │               │
    ┌────┴────┐    ┌─────┴──────┐
    │  EC2 /  │    │  S3 / RDS  │
    │  ECS    │    │  DynamoDB  │
    └─────────┘    └────────────┘
```

## Key IAM Policies & Guardrails

- **SCP: Restrict root user actions** — Enforced on all accounts except Management
- **SCP: Deny access without MFA** — Enforced on Production and Security accounts
- **SCP: Limit instance types** — Prevents expensive instance launches in Sandbox
- **CloudTrail** — Enabled on all accounts, delivered to Security account bucket
- **AWS Config** — Enables compliance rules, provides resource inventory
- **Access Analyzer** — Identifies unused access and external access in all accounts
