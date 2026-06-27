# NexusBridge Consulting — IAM Engineering Portfolio Plan

## Portfolio Structure

```
(Example Scenarios/NexusBridge Consulting/)
├── README.md
├── 00-company-context/
│   ├── company-profile.md
│   ├── org-chart.md
│   ├── personnel-roster.md
│   └── aws-account-structure.md
├── 01-user-lifecycle/
│   ├── scenario.md
│   ├── design.md
│   ├── implementation.md
│   └── evidence/
├── 02-access-certification/
│   ├── scenario.md
│   ├── design.md
│   ├── implementation.md
│   └── evidence/
├── 03-rbac-design/
│   ├── scenario.md
│   ├── design.md
│   ├── implementation.md
│   └── evidence/
├── 04-federation-sso/
│   ├── scenario.md
│   ├── design.md
│   ├── implementation.md
│   └── evidence/
├── 05-secrets-rotation/
│   ├── scenario.md
│   ├── design.md
│   ├── implementation.md
│   └── evidence/
└── 06-incident-response/
    ├── scenario.md
    ├── design.md
    ├── implementation.md
    └── evidence/
```

## Company Profile

| Attribute | Detail |
|---|---|
| **Name** | NexusBridge Consulting |
| **Industry** | Cloud Consulting & Managed Services |
| **Size** | ~300 employees |
| **Cloud Provider** | AWS (primary), with Entra ID for hybrid identity |
| **AWS Accounts** | Management, Security, Production, Nonproduction, Sandbox |
| **Identity Sources** | Entra ID (HR, Finance, Legal, Exec) + AWS IAM Identity Center (Engineering, IT) |

## Org Chart (30+ People)

**C-Suite**: CEO, CTO, CFO

**Engineering** (VP Engineering → 3 Managers)
- Platform Engineering (DevOps, Cloud Infrastructure)
- Application Development (App Teams)
- QA & Testing

**IT & Security** (CISO → Managers)
- IAM Team (IAM Lead, IAM Engineers)
- Security Operations (SOC Analysts)
- IT Support / Helpdesk

**Other Departments**
- HR (HRIS — source of truth for identity lifecycle)
- Finance
- Legal
- Sales & Marketing

## Identity Source Mapping

| Department | Identity Source |
|---|---|
| Engineering (Platform, App Dev, QA) | AWS IAM Identity Center |
| IT & Security (IAM, SOC, Helpdesk) | AWS IAM Identity Center |
| HR | Entra ID (synced) |
| Finance | Entra ID (synced) |
| Legal | Entra ID (synced) |
| Exec (CEO, CTO, CFO) | Entra ID (synced) |
| Sales & Marketing | Guest accounts (no permanent identity source) |

## Levels

- **IC1–IC5** — Individual Contributors
- **M1–M3** — Management
- **Exec** — VP, C-Level

## Scenario Template

Each scenario folder follows this structure:

1. **scenario.md** — Background context & requirements
2. **design.md** — IAM solution design
3. **implementation.md** — Terraform, scripts, AWS CLI, AND AWS console workflows
4. **evidence/** — Screenshots, logs, output documentation

## Scenarios (Execution Order)

| # | Scenario |
|---|---|
| 1 | User lifecycle (joiner/mover/leaver) |
| 2 | Access certification / recertification |
| 3 | RBAC design & implementation |
| 4 | Federation / SSO setup |
| 5 | Secrets rotation |
| 6 | IAM incident response |

## Next Steps

1. Build company context files
2. Execute scenarios in order
