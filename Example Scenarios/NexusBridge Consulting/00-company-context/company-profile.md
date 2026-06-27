# NexusBridge Consulting — Company Profile

## Overview

| Attribute | Detail |
|-----------|--------|
| **Name** | NexusBridge Consulting |
| **Industry** | Cloud Consulting & Managed Services |
| **Size** | ~300 employees |
| **HQ** | London, UK |
| **Cloud Provider** | AWS (primary), with Microsoft Entra ID for hybrid identity |
| **AWS Accounts** | Management, Security, Production, Nonproduction, Sandbox |
| **Identity Sources** | Entra ID (HR, Finance, Legal, Exec) + AWS IAM Identity Center (Engineering, IT) |

## Business Context

NexusBridge Consulting provides cloud infrastructure design, migration, and managed services to mid-market and enterprise clients across the UK and Europe. The company operates a multi-account AWS environment with a hybrid identity model:

- **Corporate identities** (HR, Finance, Legal, Exec) are managed in Entra ID, synced from the HRIS (PeopleHR).
- **Engineering & IT identities** are managed in AWS IAM Identity Center, with short-term credential enforcement.
- **Sales & Marketing** operate as guest users with minimal standing access.

The IAM team is responsible for identity lifecycle, access governance, federation, secrets management, and IAM incident response across all internal systems and client-facing platforms.

## Regulatory & Compliance Drivers

- **ISO 27001** — annual certification, requires access recertification every 90 days
- **SOC 2 Type II** — audit evidence for access controls, least privilege, and monitoring
- **GDPR** — data protection impact assessments, timely offboarding, access logging
- **Cyber Essentials Plus** — UK government baseline

## Technology Stack

| Function | Technology |
|----------|-----------|
| Identity Provider (Corporate) | Microsoft Entra ID (P2) |
| Identity Provider (Engineering) | AWS IAM Identity Center |
| HRIS | PeopleHR |
| PAM | CyberArk (self-hosted) |
| Secrets Management | HashiCorp Vault / AWS Secrets Manager |
| Infrastructure as Code | Terraform / AWS CloudFormation |
| Scripting | PowerShell / Python |
| Logging & Monitoring | AWS CloudTrail, Amazon GuardDuty, Splunk |
| Endpoint Management | Microsoft Intune |
| Ticketing | Jira Service Management |
