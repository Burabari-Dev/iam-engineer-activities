# Scenario 1: User Identity Lifecycle (Joiner / Mover / Leaver)

## Background

NexusBridge Consulting has grown from 150 to ~300 employees over the past 18 months. User provisioning and deprovisioning are still largely manual: HR emails the IT Support team, who create accounts ad-hoc via the Entra ID admin centre, and the IAM team manually adds users to AWS IAM Identity Center groups.

This manual process has caused:

- **Delayed onboarding** — new hires wait 3–5 days for full access
- **Orphaned accounts** — 23 stale AWS IAM users found in last audit
- **Over-provisioning** — leavers retain access for an average of 9 days
- **Compliance risk** — ISO 27001 requires offboarding within 24 hours

## Objective

Automate the identity lifecycle so that HR events (hire, transfer, termination) in PeopleHR drive identity changes through Entra ID and AWS IAM Identity Center with minimal manual intervention.

## Requirements

| # | Requirement |
|---|-------------|
| R1 | Joiner automatically provisioned in Entra ID and AWS IAM Identity Center within 15 minutes of HR record creation |
| R2 | Group and permission set membership derived from department + level (RBAC) |
| R3 | Mover triggers removal of old role groups / permission sets and addition of new ones |
| R4 | Leaver triggers immediate account disablement in Entra ID and IAM Identity Center, followed by a 30-day grace period before permanent deletion |
| R5 | All lifecycle events logged to Splunk for audit |
| R6 | Approval required for access to Production and Security accounts |
| R7 | SCIM-based provisioning between Entra ID and IAM Identity Center |

## User Stories

### US-01: Joiner — Alex Rivera (Platform Engineer, IC2)

> **As** a new Platform Engineer (IC2) joining Platform Engineering,
> **I want** my accounts and access to be provisioned automatically from my HR record,
> **So that** I can be productive from day one.

| Detail | Value |
|--------|-------|
| Name | Alex Rivera |
| Role | Platform Engineer |
| Level | IC2 |
| Department | Engineering — Platform Engineering |
| Manager | Priya Sharma |
| Start Date | 1 September |
| Identity Source | AWS IAM Identity Center |
| Required Access | InfraAdmin on Nonproduction, InfraReadOnly on Production, SecurityAudit on Security |

### US-02: Mover — Sophie Turner (App Developer → Platform Engineer, IC3)

> **As** an employee transferring from App Development to Platform Engineering,
> **I want** my old development permissions removed and new infrastructure permissions granted,
> **So that** I can work in my new role without retaining unnecessary access.

| Detail | Value |
|--------|-------|
| Name | Sophie Turner |
| Old Role | App Developer (IC3) — App Development |
| New Role | Platform Engineer (IC3) — Platform Engineering |
| Manager (old) | David Kim |
| Manager (new) | Priya Sharma |
| Change Date | 15 September |
| Old Access | DeveloperPowerUser on Nonproduction |
| New Access | InfraAdmin on Nonproduction, InfraReadOnly on Production |

### US-03: Leaver — Jack Williams (Junior App Developer, IC1)

> **As** a departing employee,
> **I want** my access to be revoked within 1 hour of my termination being processed by HR,
> **So that** the company's data remains secure and compliance obligations are met.

| Detail | Value |
|--------|-------|
| Name | Jack Williams |
| Role | Junior App Developer (IC1) |
| Department | Engineering — App Development |
| Termination Date | 20 September |
| Current Access | DeveloperPowerUser on Nonproduction, DeveloperReadOnly on Production |
| Reason | Resignation (voluntary) |

## Acceptance Criteria

| ID | Criterion |
|----|-----------|
| AC-01 | Alex Rivera's Entra ID and IAM Identity Center accounts created within 15 minutes of HR trigger |
| AC-02 | Alex Rivera assigned to correct permission sets (InfraAdmin on Nonprod, InfraReadOnly on Prod, SecurityAudit on Security) |
| AC-03 | Sophie Turner's old DeveloperPowerUser permission set removed within 15 minutes of HR transfer event |
| AC-04 | Sophie Turner's new InfraAdmin and InfraReadOnly permission sets granted within 15 minutes |
| AC-05 | Jack Williams's Entra ID account disabled within 15 minutes of HR termination event |
| AC-06 | Jack Williams's IAM Identity Center access removed immediately |
| AC-07 | All three events logged in Splunk with user identity, event type, and timestamp |
| AC-08 | No orphaned permission assignments remain after mover/leaver actions |
