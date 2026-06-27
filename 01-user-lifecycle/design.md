# Scenario 1: User Identity Lifecycle — Design

## Architecture Overview

```
┌─────────────┐     Webhook      ┌──────────────┐
│   PeopleHR  │ ──────────────►  │  Middleware   │
│  (HRIS)     │                  │  (Azure Func  │
│  Source of  │                  │   / PowerShell│
│   Truth)    │                  │   Runbook)    │
└─────────────┘                  └──────┬───────┘
                                        │
                         ┌──────────────┼──────────────┐
                         │              │              │
                         ▼              ▼              ▼
                  ┌────────────┐ ┌────────────┐ ┌────────────┐
                  │  Entra ID  │ │   AWS IAM  │ │  Splunk    │
                  │  (Users,   │ │  Identity  │ │  (Audit    │
                  │  Groups)   │ │  Center    │ │  Logging)  │
                  └────────────┘ └────────────┘ └────────────┘
                        │               │
                        ▼               ▼
                  ┌────────────┐ ┌──────────────────┐
                  │  SCIM      │ │  Permission Sets  │
                  │  Sync      │ │  (SSO Groups)     │
                  └────────────┘ └──────────────────┘
```

## Identity Flow

### Joiner Flow

```
HR creates record in PeopleHR
        │
        ▼
PeopleHR sends webhook (employee.create)
        │
        ▼
Middleware processes webhook:
  ├─ Extract: name, email, department, level, manager, startDate
  ├─ Determine identity source (department-based mapping)
  ├─ If Entra ID source → create user in Entra ID (disabled until start date)
  │      └─ Assign to department group → derives AWS permission sets via group mapping
  ├─ If IAM Identity Center source → create user directly in IAM Identity Center
  │      └─ Assign to group(s) → grants permission sets on target accounts
  └─ Log event to Splunk
        │
        ▼
On start date:
  ├─ Enable Entra ID account (if applicable)
  ├─ Send Welcome email via Entra ID automated messaging
  └─ Log successful activation to Splunk
```

### Mover Flow

```
HR updates record in PeopleHR (department or manager change)
        │
        ▼
PeopleHR sends webhook (employee.update)
        │
        ▼
Middleware processes webhook:
  ├─ Identify old department and new department
  ├─ Remove user from old department group(s)
  │      └─ Removes old permission set assignments in IAM Identity Center
  ├─ Add user to new department group(s)
  │      └─ Grants new permission set assignments in IAM Identity Center
  ├─ Update manager attribute
  └─ Log event to Splunk
```

### Leaver Flow

```
HR terminates record in PeopleHR (with terminationDate)
        │
        ▼
PeopleHR sends webhook (employee.terminate)
        │
        ▼
Middleware processes webhook IMMEDIATELY:
  ├─ Disable Entra ID account (revoke sessions, block sign-in)
  ├─ Remove all group memberships in Entra ID
  ├─ Remove all IAM Identity Center group assignments
  ├─ Revoke active AWS sessions (STS)
  ├─ Rotate any service credentials associated with the user
  └─ Log event to Splunk
        │
        ▼
30 days later (post-termination date):
  ├─ Delete Entra ID user (soft-delete first, then hard delete)
  ├─ Remove IAM Identity Center user
  └─ Archive offboarding evidence in secure storage
```

## RBAC Mapping: Department + Level → Permission Sets

### Identity Source Mapping

| Department | Identity Source | Entra ID Group | IAM Identity Center Group |
|------------|----------------|----------------|--------------------------|
| Platform Engineering | IAM Identity Center | — | `Platform-Engineering` |
| Application Development | IAM Identity Center | — | `App-Development` |
| QA & Testing | IAM Identity Center | — | `QA-Testing` |
| IAM | IAM Identity Center | — | `IAM-Team` |
| SOC | IAM Identity Center | — | `SOC-Team` |
| IT Support | IAM Identity Center | — | `IT-Support` |
| HR | Entra ID | `HR-Dept` | — |
| Finance | Entra ID | `Finance-Dept` | — |
| Legal | Entra ID | `Legal-Dept` | — |
| Exec | Entra ID | `Exec-Dept` | — |
| Sales & Marketing | Guest (B2B) | `Sales-Marketing` | — |

### Permission Set Matrix (IAM Identity Center groups)

| IAM Identity Center Group | Nonproduction Account | Production Account | Security Account | Management Account |
|--------------------------|----------------------|--------------------|-----------------|-------------------| 
| `Platform-Engineering` | InfraAdmin | InfraReadOnly | SecurityAudit | — |
| `App-Development` | DeveloperPowerUser | DeveloperReadOnly | — | — |
| `QA-Testing` | QATester | — | — | — |
| `IAM-Team` | IAMAdmin | IAMAdmin | SecurityAudit | IAMAdmin |
| `SOC-Team` | — | — | SecurityAudit | — |
| `IT-Support` | SupportUser | SupportUser | — | — |

### Level-Based Guardrails

| Level | MFA Enforced? | Approval Required? | JIT Elevation? |
|-------|--------------|-------------------|----------------|
| IC1–IC2 | Yes | No | No |
| IC3 | Yes | No | No |
| IC4–IC5 | Yes | Yes (Prod) | Yes |
| M1–M3 | Yes | Yes (Prod/Security) | Yes |
| Exec | Yes | Yes (all accounts) | Yes |

## Approval Workflow

```
Request for elevated/Production access
        │
        ▼
┌─────────────────────┐
│  Jira Service Mgmt  │
│  Auto-triggered     │
│  ticket             │
└──────────┬──────────┘
           │
     ┌─────┴─────┐
     │           │
     ▼           ▼
┌────────┐ ┌────────┐
│ Manager│ │ IAM    │
│ Approve│ │ Approve│
└────┬───┘ └────┬───┘
     │          │
     └────┬─────┘
          │ Both approved?
          ├─ Yes → Provision access (auto-remove after 24h if JIT)
          └─ No  → Deny, notify requester
```

## SCIM Provisioning: Entra ID → AWS IAM Identity Center

Entra ID acts as the upstream identity source for IAM Identity Center via SCIM:

- Entra ID department groups are mapped to IAM Identity Center groups
- When a user is added to an Entra ID group, that group membership is synced to the corresponding IAM Identity Center group
- IAM Identity Center permission sets are assigned to groups (not individual users)
- This ensures consistent, group-based access management

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| Group-based (not user-based) permission assignments | Easier to manage at scale, fewer individual assignments |
| Department groups as the primary RBAC mechanism | Aligns with HR data, predictable mapping |
| Immediate disable on termination (30-day soft-delete) | Balances security (fast revocation) with operational recovery (accidental termination) |
| Splunk logging for all lifecycle events | Required for SOC 2 and ISO 27001 audit evidence |
| Approval gating limited to Production/Security | Minimises friction for non-sensitive environments |
