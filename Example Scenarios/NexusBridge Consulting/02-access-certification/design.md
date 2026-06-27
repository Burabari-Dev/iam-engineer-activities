# Scenario 2: Access Certification — Design

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                    Certification Campaign                     │
│  (AWS IAM Access Analyzer + Custom PowerShell Module)        │
└──────────────────────┬───────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
┌────────────┐ ┌────────────┐ ┌────────────┐
│  Entra ID  │ │ AWS IAM    │ │  Splunk    │
│  Access    │ │ Identity   │ │  (Audit    │
│  Reviews   │ │ Center     │ │  Logging)  │
│  (API)     │ │ (API)      │ │            │
└────────────┘ └────────────┘ └────────────┘
        │              │
        ▼              ▼
┌──────────────────────────────────────┐
│         Data Aggregation Layer        │
│  (PowerShell: Export-AccessReview)    │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│         Jira Service Management       │
│  (Campaign dashboard, approvals)     │
│  + Custom Power App (reviewer UI)     │
└──────────────────┬───────────────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
        ▼                     ▼
┌──────────────┐   ┌──────────────────┐
│  Email       │   │  Enforcement     │
│  Notifications│   │  (Revoke-Access) │
│  (Reminders  │   │  - Entra ID      │
│   & Escalations)│  │  - IAM Identity │
└──────────────┘   │    Center        │
                   └──────────────────┘
```

## Certification Flow

```
Day 0 — Campaign Launch
├─ IAM Manager triggers: Start-CertificationCampaign -Quarter 2026-Q3
├─ System aggregates:
│   ├─ All active Entra ID users + group memberships
│   ├─ All active IAM Identity Center users + permission set assignments
│   └─ Manager hierarchy from HRIS / Entra ID
├─ Generates certification dataset (CSV + SQLite)
├─ Creates Jira issues per reviewer
└─ Sends notification email to all reviewers

Day 1–14 — Review Window
├─ Reviewer opens dashboard (Power App or Jira portal)
├─ Dashboard shows:
│   ├─ Reviewer name, department
│   ├─ Table of direct reports with:
│   │   ├─ Name, email, level
│   │   ├─ Current access (Entra ID groups + IAM Identity Center permission sets)
│   │   ├─ Last certification date & decision
│   │   └─ Days since last review
│   └─ Actions: [Approve All] [Reject All] [Individual Approve/Reject]
├─ Reviewer submits decisions
└─ Decisions logged to Splunk

Day 7 — First Reminder
├─ Check: which reviewers have not submitted?
├─ Send automated email reminder to outstanding reviewers
└─ Log reminder event to Splunk

Day 14 — Escalation
├─ Check: still outstanding reviews?
├─ Send escalation email to reviewer's manager (VP/Director)
├─ CC the IAM team
└─ Log escalation event to Splunk

Day 21 — Auto-Enforcement
├─ Identify all users whose access was not approved
├─ For rejected and unresponded:
│   ├─ Remove Entra ID group memberships (if applicable)
│   ├─ Remove IAM Identity Center group memberships
│   └─ Revoke active sessions
├─ Generate enforcement report
├─ Log all actions to Splunk
└─ Archive campaign evidence package
```

## Data Model

### Certification Record

```
CampaignId      : 2026-Q3
ReviewerId      : p.sharma@nexusbridge.co.uk
UserId          : l.kraft@nexusbridge.co.uk
IdentitySource  : IAMIdentityCenter
AccessItems     : [
                    { type: "PermissionSet", name: "InfraAdmin", account: "nonproduction" },
                    { type: "PermissionSet", name: "InfraReadOnly", account: "production" },
                    { type: "PermissionSet", name: "SecurityAudit", account: "security" }
                  ]
LastReviewDate  : 2026-04-02
LastDecision    : Approved
Decision        : null (pending)
DecisionDate    : null
ReviewerComment : null
Status          : Pending
```

## AWS IAM Access Analyzer Integration

IAM Access Analyzer identifies unused access to inform reviewer decisions:

| Finding Type | What It Means | Suggested Action |
|-------------|---------------|-----------------|
| `UnusedPermission` | Permission set not used in 45+ days | Flag for potential revocation |
| `UnusedRole` | IAM role not assumed in 45+ days | Flag as potentially stale |
| `ExternalAccess` | Access granted to external principal | Flag for immediate review |

Reviewers see these flags in their dashboard alongside each user's access.

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| Jira + Power App for reviewer UI | Familiar tool for managers; Power App provides custom certification dashboard |
| Auto-revoke at day 21 (not immediately) | Gives reviewers a grace period; compliance requirement is quarterly, not real-time |
| Manager-based (not role-based) review | Manager knows their direct reports' current responsibilities best |
| Access Analyzer flags as advisory only | Automated suggestions; final decision always with the manager |
| Evidence package as signed JSON + CSV | Machine-readable for auditors, importable into compliance tools |
