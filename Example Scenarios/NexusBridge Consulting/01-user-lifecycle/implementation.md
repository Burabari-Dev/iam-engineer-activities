# Scenario 1: User Identity Lifecycle — Implementation

## Prerequisites

- AWS Organizations with IAM Identity Center enabled
- Entra ID P2 tenant with SCIM provisioning configured to IAM Identity Center
- PeopleHR API key with `employee.read` and `webhook` scopes
- Azure Automation Account (PowerShell 7.2) with managed identity
- Splunk HTTP Event Collector (HEC) token
- Terraform v1.6+ and AWS CLI v2+

---

## 1. Terraform: IAM Identity Center Group & Permission Set Definitions

### `main.tf`

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "eu-west-2"
}
```

### `data.tf`

```hcl
data "aws_ssoadmin_instances" "this" {}

data "aws_identitystore_group" "platform_engineering" {
  identity_store_id = tolist(data.aws_ssoadmin_instances.this.identity_store_ids)[0]

  alternate_identifier {
    unique_attribute {
      attribute_path  = "DisplayName"
      attribute_value = "Platform-Engineering"
    }
  }
}

data "aws_identitystore_group" "app_development" {
  identity_store_id = tolist(data.aws_ssoadmin_instances.this.identity_store_ids)[0]

  alternate_identifier {
    unique_attribute {
      attribute_path  = "DisplayName"
      attribute_value = "App-Development"
    }
  }
}

data "aws_identitystore_group" "iam_team" {
  identity_store_id = tolist(data.aws_ssoadmin_instances.this.identity_store_ids)[0]

  alternate_identifier {
    unique_attribute {
      attribute_path  = "DisplayName"
      attribute_value = "IAM-Team"
    }
  }
}

data "aws_identitystore_group" "soc_team" {
  identity_store_id = tolist(data.aws_ssoadmin_instances.this.identity_store_ids)[0]

  alternate_identifier {
    unique_attribute {
      attribute_path  = "DisplayName"
      attribute_value = "SOC-Team"
    }
  }
}

data "aws_identitystore_group" "it_support" {
  identity_store_id = tolist(data.aws_ssoadmin_instances.this.identity_store_ids)[0]

  alternate_identifier {
    unique_attribute {
      attribute_path  = "DisplayName"
      attribute_value = "IT-Support"
    }
  }
}
```

### `permission-sets.tf`

```hcl
locals {
  identity_store_id = tolist(data.aws_ssoadmin_instances.this.identity_store_ids)[0]

  accounts = {
    management    = "111111111111"
    security      = "222222222222"
    nonproduction = "333333333333"
    production    = "444444444444"
    sandbox       = "555555555555"
  }

  # Permission set definitions
  permission_sets = {
    InfraAdmin = {
      description    = "Full infrastructure admin access"
      session_length = "PT4H"
      policy = jsonencode({
        Version = "2012-10-17"
        Statement = [
          {
            Effect   = "Allow"
            Action   = "*"
            Resource = "*"
            Condition = {
              Bool = {
                "aws:MultiFactorAuthPresent" = "true"
              }
            }
          }
        ]
      })
    }

    InfraReadOnly = {
      description    = "Read-only infrastructure access"
      session_length = "PT8H"
      policy = jsonencode({
        Version = "2012-10-17"
        Statement = [
          {
            Effect   = "Allow"
            Action   = [
              "ec2:Describe*",
              "rds:Describe*",
              "s3:List*",
              "s3:Get*",
              "iam:Get*",
              "iam:List*"
            ]
            Resource = "*"
          }
        ]
      })
    }

    DeveloperPowerUser = {
      description    = "Power user access for developers"
      session_length = "PT8H"
      policy = jsonencode({
        Version = "2012-10-17"
        Statement = [
          {
            Effect   = "Allow"
            NotAction = [
              "iam:*",
              "organizations:*",
              "account:*"
            ]
            Resource = "*"
          }
        ]
      })
    }

    DeveloperReadOnly = {
      description    = "Read-only access for production viewing"
      session_length = "PT8H"
      policy = jsonencode({
        Version = "2012-10-17"
        Statement = [
          {
            Effect   = "Allow"
            Action   = [
              "ec2:Describe*",
              "rds:Describe*",
              "s3:List*",
              "s3:GetObject",
              "logs:Describe*",
              "logs:Get*",
              "cloudwatch:Describe*",
              "cloudwatch:Get*"
            ]
            Resource = "*"
          }
        ]
      })
    }

    SecurityAudit = {
      description    = "Security audit access"
      session_length = "PT8H"
      policy = jsonencode({
        Version = "2012-10-17"
        Statement = [
          {
            Effect   = "Allow"
            Action   = [
              "iam:Get*",
              "iam:List*",
              "iam:Generate*",
              "guardduty:Get*",
              "guardduty:List*",
              "cloudtrail:Describe*",
              "cloudtrail:Get*",
              "cloudtrail:LookupEvents",
              "config:Get*",
              "config:List*",
              "s3:GetBucketPolicy",
              "kms:Describe*",
              "kms:List*"
            ]
            Resource = "*"
          }
        ]
      })
    }

    IAMAdmin = {
      description    = "IAM administrative access"
      session_length = "PT2H"
      policy = jsonencode({
        Version = "2012-10-17"
        Statement = [
          {
            Effect   = "Allow"
            Action   = [
              "iam:*",
              "organizations:DescribeAccount",
              "organizations:ListAccounts",
              "sso:*"
            ]
            Resource = "*"
            Condition = {
              Bool = {
                "aws:MultiFactorAuthPresent" = "true"
              }
            }
          }
        ]
      })
    }

    SupportUser = {
      description    = "Basic support user access"
      session_length = "PT8H"
      policy = jsonencode({
        Version = "2012-10-17"
        Statement = [
          {
            Effect   = "Allow"
            Action   = [
              "ec2:Describe*",
              "ec2:Get*",
              "rds:Describe*",
              "s3:ListAllMyBuckets",
              "support:*",
              "iam:ListUsers",
              "iam:GetAccountSummary"
            ]
            Resource = "*"
          }
        ]
      })
    }

    QATester = {
      description    = "QA tester access"
      session_length = "PT8H"
      policy = jsonencode({
        Version = "2012-10-17"
        Statement = [
          {
            Effect   = "Allow"
            Action   = [
              "ec2:Describe*",
              "ec2:RunInstances",
              "ec2:TerminateInstances",
              "ec2:StartInstances",
              "ec2:StopInstances",
              "s3:List*",
              "s3:GetObject",
              "s3:PutObject"
            ]
            Resource = "*"
          }
        ]
      })
    }
  }

  # Group → Account → Permission Set assignments
  group_assignments = {
    "Platform-Engineering" = [
      { account = "nonproduction", permission_set = "InfraAdmin" },
      { account = "production",    permission_set = "InfraReadOnly" },
      { account = "security",      permission_set = "SecurityAudit" },
    ]
    "App-Development" = [
      { account = "nonproduction", permission_set = "DeveloperPowerUser" },
      { account = "production",    permission_set = "DeveloperReadOnly" },
    ]
    "QA-Testing" = [
      { account = "nonproduction", permission_set = "QATester" },
    ]
    "IAM-Team" = [
      { account = "management",    permission_set = "IAMAdmin" },
      { account = "security",      permission_set = "IAMAdmin" },
      { account = "nonproduction", permission_set = "IAMAdmin" },
      { account = "production",    permission_set = "IAMAdmin" },
    ]
    "SOC-Team" = [
      { account = "security",      permission_set = "SecurityAudit" },
    ]
    "IT-Support" = [
      { account = "nonproduction", permission_set = "SupportUser" },
      { account = "production",    permission_set = "SupportUser" },
    ]
  }
}

# Create permission sets
resource "aws_ssoadmin_permission_set" "this" {
  for_each         = local.permission_sets
  name             = each.key
  description      = each.value.description
  instance_arn     = tolist(data.aws_ssoadmin_instances.this.arns)[0]
  session_duration = each.value.session_length
}

# Attach inline policies to permission sets
resource "aws_ssoadmin_permission_set_inline_policy" "this" {
  for_each           = local.permission_sets
  instance_arn       = tolist(data.aws_ssoadmin_instances.this.arns)[0]
  permission_set_arn = aws_ssoadmin_permission_set.this[each.key].arn
  inline_policy      = each.value.policy
}

# Assign permission sets to groups on accounts
resource "aws_ssoadmin_account_assignment" "this" {
  for_each = {
    for pair in flatten([
      for group_name, assignments in local.group_assignments : [
        for assignment in assignments : {
          key = "${group_name}-${assignment.account}-${assignment.permission_set}"
          group_name       = group_name
          account_alias    = assignment.account
          permission_set   = assignment.permission_set
        }
      ]
    ]) : pair.key => pair
  }

  instance_arn       = tolist(data.aws_ssoadmin_instances.this.arns)[0]
  permission_set_arn = aws_ssoadmin_permission_set.this[each.value.permission_set].arn
  principal_id       = data.aws_identitystore_group.this[each.value.group_name].group_id
  principal_type     = "GROUP"
  target_id          = local.accounts[each.value.account_alias]
  target_type        = "AWS_ACCOUNT"
}

data "aws_identitystore_group" "this" {
  for_each = toset([
    "Platform-Engineering",
    "App-Development",
    "QA-Testing",
    "IAM-Team",
    "SOC-Team",
    "IT-Support",
  ])

  identity_store_id = local.identity_store_id

  alternate_identifier {
    unique_attribute {
      attribute_path  = "DisplayName"
      attribute_value = each.key
    }
  }
}
```

### Deploy

```powershell
# PowerShell
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

---

## 2. Middleware: Azure Automation Runbook (PowerShell)

### Joiner Runbook — `New-IdentityLifecycle.ps1`

```powershell
param(
    [Parameter(Mandatory)]
    [hashtable]$HREvent  # Received from PeopleHR webhook
)

# ── Configuration ──────────────────────────────────────────────
$SplunkHECUrl  = "https://hec.splunk.nexusbridge.co.uk:8088/services/collector"
$SplunkHECToken = Get-AutomationVariable -Name "SplunkHECToken"
$PeopleHRApiKey = Get-AutomationVariable -Name "PeopleHRApiKey"
$EntraIDGroupMap = @{
    "HR"     = "HR-Dept"
    "Finance" = "Finance-Dept"
    "Legal"  = "Legal-Dept"
    "Exec"   = "Exec-Dept"
    "Sales & Marketing" = "Sales-Marketing"
}

# ── Helper Functions ──────────────────────────────────────────
function Write-SplunkEvent {
    param([string]$EventType, [hashtable]$Data)
    $body = @{
        event = @{
            source     = "identity-lifecycle"
            sourcetype = "iam:lifecycle"
            event_type = $EventType
            timestamp  = (Get-Date -Format "o")
        } + $Data
    } | ConvertTo-Json
    $headers = @{ Authorization = "Splunk $SplunkHECToken" }
    Invoke-RestMethod -Uri $SplunkHECUrl -Method Post -Body $body -ContentType "application/json" -Headers $headers
}

# ── Main Logic ────────────────────────────────────────────────
try {
    $eventType = $HREvent.EventType  # "employee.create"
    $employee  = $HREvent.Employee

    Write-Output "Processing $($eventType) for $($employee.FirstName) $($employee.LastName)"

    $department  = $employee.Department
    $level       = $employee.Level
    $email       = "$($employee.FirstName.ToLower()).$($employee.LastName.ToLower())@nexusbridge.co.uk"
    $identitySrc = Get-IdentitySource -Department $department

    if ($identitySrc -eq "EntraID") {
        # Create user in Entra ID (disabled until start date)
        $entraUserParams = @{
            DisplayName        = "$($employee.FirstName) $($employee.LastName)"
            UserPrincipalName  = $email
            MailNickname       = "$($employee.FirstName).$($employee.LastName)"
            AccountEnabled     = $false
            Department         = $department
            JobTitle           = $employee.Role
            EmployeeId         = $employee.EmployeeId
            ManagerId          = $employee.ManagerId
            UsageLocation      = "GB"
        }
        $user = New-MgUser @entraUserParams

        # Add to department group
        $groupName = $EntraIDGroupMap[$department]
        $group = Get-MgGroup -Filter "displayName eq '$groupName'"
        New-MgGroupMember -GroupId $group.Id -DirectoryObjectId $user.Id

        Write-Output "Created Entra ID user: $email"
    }
    else {
        # IAM Identity Center — groups already exist (provisioned by SCIM or Terraform)
        # SCIM from Entra ID handles group membership if Entra ID is upstream
        # Otherwise create user via AWS CLI / API
        Write-Output "User will be provisioned in IAM Identity Center via SCIM"
    }

    # Audit log
    Write-SplunkEvent -EventType "JOINER" -Data @{
        employee_id   = $employee.EmployeeId
        email         = $email
        department    = $department
        role          = $employee.Role
        level         = $level
        identity_src  = $identitySrc
        status        = "provisioned"
    }
}
catch {
    Write-Error "Failed to process joiner: $_"
    Write-SplunkEvent -EventType "JOINER_FAILURE" -Data @{
        employee_id = $employee.EmployeeId
        error       = $_.Exception.Message
    }
    throw
}
```

### Mover Runbook — `Move-IdentityLifecycle.ps1`

```powershell
param(
    [Parameter(Mandatory)]
    [hashtable]$HREvent
)

try {
    $employee     = $HREvent.Employee
    $oldDept      = $HREvent.Changes.OldDepartment
    $newDept      = $HREvent.Changes.NewDepartment
    $email        = "$($employee.FirstName.ToLower()).$($employee.LastName.ToLower())@nexusbridge.co.uk"

    # Remove from old IAM Identity Center group
    $oldGroupName = Convert-DeptToGroupName -Department $oldDept -IdentitySource "IAMIdentityCenter"
    $identityStoreId = (Get-SSOAdminInstance).IdentityStoreId
    $oldGroupId  = (Get-IdentityStoreGroup -IdentityStoreId $identityStoreId -DisplayName $oldGroupName).GroupId
    $userId      = (Get-IdentityStoreUser -IdentityStoreId $identityStoreId -AlternateIdentifier @{
        UniqueAttribute = @{
            AttributePath  = "UserName"
            AttributeValue = $email
        }
    }).UserId

    Remove-IdentityStoreGroupMembership -IdentityStoreId $identityStoreId -GroupId $oldGroupId -MemberId $userId

    # Add to new IAM Identity Center group
    $newGroupName = Convert-DeptToGroupName -Department $newDept -IdentitySource "IAMIdentityCenter"
    $newGroupId   = (Get-IdentityStoreGroup -IdentityStoreId $identityStoreId -DisplayName $newGroupName).GroupId

    New-IdentityStoreGroupMembership -IdentityStoreId $identityStoreId -GroupId $newGroupId -MemberId $userId

    Write-Output "Moved $email from $oldDept to $newDept"
    Write-SplunkEvent -EventType "MOVER" -Data @{
        employee_id  = $employee.EmployeeId
        email        = $email
        old_department = $oldDept
        new_department = $newDept
        status       = "updated"
    }
}
catch {
    Write-Error "Failed to process mover: $_"
    throw
}
```

### Leaver Runbook — `Remove-IdentityLifecycle.ps1`

```powershell
param(
    [Parameter(Mandatory)]
    [hashtable]$HREvent
)

try {
    $employee = $HREvent.Employee
    $email    = "$($employee.FirstName.ToLower()).$($employee.LastName.ToLower())@nexusbridge.co.uk"
    $upn      = $email  # UPN matches email

    # ── 1. Revoke Entra ID sessions and disable account ──
    $user = Get-MgUser -Filter "userPrincipalName eq '$upn'"
    Update-MgUser -UserId $user.Id -AccountEnabled $false
    Revoke-MgUserSignInSession -UserId $user.Id

    Write-Output "Disabled and revoked sessions for $email"

    # ── 2. Remove all group memberships ──
    $groups = Get-MgUserMemberOf -UserId $user.Id
    foreach ($group in $groups) {
        Remove-MgGroupMemberByRef -GroupId $group.Id -DirectoryObjectId $user.Id
    }

    Write-Output "Removed all group memberships for $email"

    # ── 3. Remove IAM Identity Center assignments ──
    $identityStoreId = (Get-SSOAdminInstance).IdentityStoreId
    $userId = (Get-IdentityStoreUser -IdentityStoreId $identityStoreId -AlternateIdentifier @{
        UniqueAttribute = @{
            AttributePath  = "UserName"
            AttributeValue = $email
        }
    }).UserId

    $memberships = Get-IdentityStoreGroupMembership -IdentityStoreId $identityStoreId -MemberId $userId
    foreach ($membership in $memberships) {
        Remove-IdentityStoreGroupMembership -IdentityStoreId $identityStoreId `
            -GroupId $membership.GroupId -MemberId $userId
    }

    Write-Output "Removed IAM Identity Center group memberships for $email"

    # ── 4. Rotate any user-associated service credentials ──
    # (Assuming we have a credential inventory — placeholder)
    # Invoke-RotateServiceCredentials -OwnerEmail $email

    # ── 5. Audit ──
    Write-SplunkEvent -EventType "LEAVER" -Data @{
        employee_id = $employee.EmployeeId
        email       = $email
        status      = "disabled"
        action      = "immediate_revocation"
    }

    Write-Output "Leaver process completed for $email"

    # ── 6. Schedule cleanup (30 days) ──
    $cleanupDate = (Get-Date).AddDays(30)
    Register-ScheduledJob -Name "Cleanup-$($employee.EmployeeId)" `
        -ScriptBlock {
            $upn = "$using:email"
            $user = Get-MgUser -Filter "userPrincipalName eq '$upn'"
            Remove-MgUser -UserId $user.Id
            Remove-IdentityStoreUser -IdentityStoreId $using:identityStoreId -UserId $using:userId
        } -RunNow -At $cleanupDate

    Write-Output "Scheduled permanent cleanup for $cleanupDate"
}
catch {
    Write-Error "Failed to process leaver: $_"
    Write-SplunkEvent -EventType "LEAVER_FAILURE" -Data @{
        employee_id = $employee.EmployeeId
        email       = $email
        error       = $_.Exception.Message
    }
    throw
}
```

---

## 3. SCIM Provisioning Configuration (Entra ID → IAM Identity Center)

### Step 1: Enable SCIM in IAM Identity Center

```powershell
# AWS CLI
aws sso-admin create-scim-credential `
    --instance-arn arn:aws:sso:::instance/ssoins-12345678
```

Save the returned `AccessToken` and `ScimEndpoint`.

### Step 2: Configure Entra ID Provisioning

1. In Entra ID → Enterprise applications → AWS IAM Identity Center
2. Enable automatic provisioning
3. Set:
   - **Tenant URL**: `https://scim.eu-west-2.amazonaws.com/.../scim/v2`
   - **Secret Token**: Token from Step 1
4. Under **Mappings**, map Entra ID attribute `department` → IAM Identity Center group membership
5. Under **Settings**, set scope to "Sync only assigned users and groups"
6. Select **Provision on demand** to test

### Step 3: Group Mapping

In Entra ID, create groups that mirror IAM Identity Center groups:

| Entra ID Group | Mapped to IAM Identity Center Group |
|---------------|--------------------------------------|
| `AWS-Platform-Engineering` | `Platform-Engineering` |
| `AWS-App-Development` | `App-Development` |
| `AWS-QA-Testing` | `QA-Testing` |
| `AWS-IAM-Team` | `IAM-Team` |
| `AWS-SOC-Team` | `SOC-Team` |
| `AWS-IT-Support` | `IT-Support` |

When a user is added to `AWS-Platform-Engineering` in Entra ID, SCIM automatically provisions them to the corresponding group in IAM Identity Center.

---

## 4. AWS Console Workflows

### Joiner — Manual verification

1. Sign in to AWS IAM Identity Center console
2. Navigate to **Users** — Alex Rivera should appear
3. Navigate to **Groups** → `Platform-Engineering` → Alex Rivera listed as member
4. Navigate to **AWS accounts** → Nonproduction → `InfraAdmin` assigned to `Platform-Engineering`
5. Navigate to **AWS accounts** → Production → `InfraReadOnly` assigned to `Platform-Engineering`
6. Navigate to **AWS accounts** → Security → `SecurityAudit` assigned to `Platform-Engineering`

### Leaver — Manual verification

1. Sign in to Entra ID admin centre → Users → Jack Williams → Account enabled = **No**
2. Sign in to AWS IAM Identity Center → Users → Jack Williams → Group memberships = **empty**
3. Attempt SSO as Jack Williams → **Access denied**

---

## 5. Testing

### Test Script — `Test-UserLifecycle.ps1`

```powershell
# Simulate HR events for testing

$joinerEvent = @{
    EventType = "employee.create"
    Employee  = @{
        FirstName   = "Alex"
        LastName    = "Rivera"
        Email       = "a.rivera@nexusbridge.co.uk"
        Department  = "Platform Engineering"
        Role        = "Platform Engineer"
        Level       = "IC2"
        EmployeeId  = "EMP-042"
        ManagerId   = "EMP-019"  # Priya Sharma
        StartDate   = "2026-09-01"
    }
}

$moverEvent = @{
    EventType = "employee.update"
    Employee  = @{
        FirstName   = "Sophie"
        LastName    = "Turner"
        EmployeeId  = "EMP-031"
    }
    Changes   = @{
        OldDepartment = "Application Development"
        NewDepartment = "Platform Engineering"
    }
}

$leaverEvent = @{
    EventType = "employee.terminate"
    Employee  = @{
        FirstName   = "Jack"
        LastName    = "Williams"
        EmployeeId  = "EMP-038"
        TerminationDate = "2026-09-20"
    }
}

# Execute
.\New-IdentityLifecycle.ps1 -HREvent $joinerEvent
.\Move-IdentityLifecycle.ps1 -HREvent $moverEvent
.\Remove-IdentityLifecycle.ps1 -HREvent $leaverEvent
```

### Validation Queries

```bash
# AWS CLI — verify IAM Identity Center group membership
aws identitystore list-group-memberships \
    --identity-store-id d-1234567890 \
    --group-id $(aws identitystore get-group-id \
        --identity-store-id d-1234567890 \
        --display-name "Platform-Engineering" \
        --query GroupId --output text)

# Verify permission set assignments
aws sso-admin list-account-assignments \
    --instance-arn arn:aws:sso:::instance/ssoins-12345678 \
    --account-id 333333333333

# Check Splunk logs
curl -k -X GET "https://hec.splunk.nexusbridge.co.uk:8089/services/search/jobs" \
    -d 'search=source="iam:lifecycle" employee_id=EMP-042' \
    -u "admin:$SPLUNK_PASS"
```

---

## 6. Expected Outcomes

| User Story | Acceptance Criteria | Verification Method |
|-----------|-------------------|-------------------|
| **US-01** Joiner (Alex Rivera) | AC-01: Account created < 15 min from HR event | Splunk event for `JOINER` with status `provisioned` |
| | AC-02: Correct permission sets assigned | AWS CLI `list-account-assignments` confirms InfraAdmin nonprod, InfraReadOnly prod, SecurityAudit security |
| **US-02** Mover (Sophie Turner) | AC-03: Old DeveloperPowerUser removed | AWS CLI — no assignment for Sophie on DeveloperPowerUser |
| | AC-04: New InfraAdmin/InfraReadOnly granted | AWS CLI — assignments confirmed |
| **US-03** Leaver (Jack Williams) | AC-05: Entra ID disabled | Entra ID admin centre → Account enabled = false |
| | AC-06: IAM Identity Center access removed | IAM Identity Center → Jack Williams → no group memberships |
| **All** | AC-07: Logged to Splunk | Splunk query for event_type=JOINER/MOVER/LEAVER for each employee_id |
| | AC-08: No orphaned assignments | Script to diff expected vs actual assignments |
