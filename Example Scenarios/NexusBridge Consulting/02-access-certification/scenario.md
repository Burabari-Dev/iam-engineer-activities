# Scenario 2: Access Certification & Recertification

## Background

NexusBridge Consulting is ISO 27001 certified and SOC 2 compliant. Both frameworks require periodic recertification of user access rights:

- **ISO 27001 (A.9.2.5, A.9.2.6)** — Access rights must be reviewed at regular intervals (NexusBridge policy: every 90 days)
- **SOC 2 (CC6.1, CC6.2)** — Access provisioning and deprovisioning must be periodically verified

The current process is manual: the IAM team exports Entra ID and AWS IAM Identity Center assignments to a spreadsheet, emails managers to review their teams, and manually updates access based on email replies. The last campaign took 23 days to complete and 3 managers never responded — their teams' access went unreviewed.

## Objective

Automate the quarterly access recertification process so that campaigns are generated, distributed, tracked, and enforced with minimal manual effort.

## Requirements

| # | Requirement |
|---|-------------|
| R1 | Quarterly campaign auto-generated based on current IAM Identity Center and Entra ID assignments |
| R2 | Each user reviewed by their direct manager (as recorded in HRIS / Entra ID manager attribute) |
| R3 | Reviewers receive a single dashboard showing all direct reports and their current access |
| R4 | Reviewer can approve all, reject all, or decide per-user |
| R5 | Automatic escalation if reviewer has not responded within 14 days (reminder at day 7, escalation to VP at day 14) |
| R6 | Access not approved within 21 days is auto-revoked |
| R7 | Full audit trail — who reviewed what, when, and what action was taken |
| R8 | Evidence package generated for external auditors |

## User Stories

### US-01: Campaign Launch — Daniel Ortega (IAM Manager)

> **As** the IAM Manager,
> **I want** to launch a quarterly certification campaign with one action,
> **So that** I don't have to manually compile spreadsheets.

| Detail | Value |
|--------|-------|
| Name | Daniel Ortega |
| Role | Manager, IAM |
| Trigger | First day of the quarter (1 Jan, 1 Apr, 1 Jul, 1 Oct) |
| Expected output | Campaign dashboard with 40+ reviewers and ~280 users to review |

### US-02: Manager Review — Priya Sharma (Platform Engineering Manager)

> **As** a department manager,
> **I want** to see all my direct reports and their current AWS permissions in one place,
> **So that** I can quickly confirm or revoke access.

| Detail | Value |
|--------|-------|
| Name | Priya Sharma |
| Role | Manager, Platform Engineering |
| Direct Reports | Tom Baker (IC4), Lena Kraft (IC3), Amir Hussain (IC2), Sophie Turner (IC3, moved from App Dev) |
| Access to review | IAM Identity Center permission sets on Nonproduction, Production, Security |
| Deadline | 14 days from campaign start |

### US-03: Escalation & Enforcement — James Okonkwo (VP Engineering)

> **As** the VP of Engineering,
> **I want** to be notified when my managers haven't completed their reviews,
> **So that** I can ensure compliance before the auto-revocation deadline.

| Detail | Value |
|--------|-------|
| Name | James Okonkwo |
| Role | VP Engineering |
| Managers reporting to him | Priya Sharma (Platform), David Kim (App Dev), Emily Nash (QA) |
| Escalation trigger | Day 14 after campaign start — any incomplete reviews |
| Enforcement deadline | Day 21 — auto-revoke unapproved access |

### US-04: Audit Evidence — Helen Ross (CISO)

> **As** the CISO,
> **I want** an auditor-ready evidence package for the last 4 quarters,
> **So that** ISO 27001 surveillance audits pass without findings.

| Detail | Value |
|--------|-------|
| Name | Helen Ross |
| Role | CISO |
| Audit date | External ISO 27001 surveillance audit in 6 weeks |
| Requirement | Evidence of quarterly access reviews for the past 12 months |

## Acceptance Criteria

| ID | Criterion |
|----|-----------|
| AC-01 | Campaign generates complete list of all active users and their current permission set / group assignments |
| AC-02 | Each user assigned to their manager for review (based on HRIS manager attribute) |
| AC-03 | Reviewer dashboard shows user name, department, level, current access, last review date, and last action |
| AC-04 | Auto-reminder sent at day 7 to non-responsive reviewers |
| AC-05 | Escalation notification sent to VP/director at day 14 |
| AC-06 | Unapproved access auto-revoked at day 21 |
| AC-07 | Audit package contains: campaign summary, reviewer decisions, enforcement log, and timestamped evidence |
| AC-08 | Revocation actions logged to Splunk with reviewer identity, affected user, and access removed |
