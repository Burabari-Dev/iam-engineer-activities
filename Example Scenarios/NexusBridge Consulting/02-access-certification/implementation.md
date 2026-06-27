# Scenario 2: Access Certification — Implementation

## Prerequisites

- PowerShell 7.2+ with Microsoft Graph and AWS PowerShell modules
- AWS CLI v2+ configured with IAM Identity Center admin access
- Splunk HEC token
- Jira Service Management API token (project admin)
- Isengard test environment for dry-run campaigns

---

## 1. PowerShell Module: Access Certification Engine

### `Certification-Campaign.psm1`

```powershell
# Certification-Campaign.psm1
# Main module for access certification lifecycle

# ── Configuration ──────────────────────────────────────────────
$script:CampaignDbPath = "$PSScriptRoot\campaigns"
$script:SplunkHECUrl   = "https://hec.splunk.nexusbridge.co.uk:8088/services/collector"
$script:SplunkToken    = $env:SPLUNK_HEC_TOKEN
$script:JiraBaseUrl    = "https://nexusbridge.atlassian.net"
$script:JiraProjectKey = "IAM"
$script:JiraIssueType  = "Task"

# ── Helper ────────────────────────────────────────────────────
function Write-CampaignLog {
    param($EventType, $Data)
    $body = @{
        event = @{
            source     = "access-certification"
            sourcetype = "iam:certification"
            event_type = $EventType
            timestamp  = (Get-Date -Format "o")
        } + $Data
    } | ConvertTo-Json -Depth 10
    $headers = @{ Authorization = "Splunk $script:SplunkToken" }
    Invoke-RestMethod -Uri $script:SplunkHECUrl -Method Post -Body $body -ContentType "application/json" -Headers $headers -ErrorAction SilentlyContinue
}

# ── 1. Export Current Access State ──────────────────────────
function Export-AccessReviewData {
    <#
    .SYNOPSIS
    Aggregates all current access from Entra ID and IAM Identity Center.
    .DESCRIPTION
    Exports a unified dataset of user → access mappings across both identity sources,
    enriched with manager hierarchy from Entra ID.
    #>

    $outputPath = Join-Path $script:CampaignDbPath "$(Get-Date -Format 'yyyy-MM-dd')_access_snapshot.json"

    # ── Entra ID users and group memberships ──
    Connect-MgGraph -Scopes "User.Read.All", "Group.Read.All", "Directory.Read.All" -NoWelcome

    $entraUsers = Get-MgUser -All -Property "id,displayName,userPrincipalName,department,jobTitle,employeeId,manager"

    $entraGroups = Get-MgGroup -All -Filter "startsWith(displayName, 'AWS-') or startsWith(displayName, 'HR-') or startsWith(displayName, 'Finance-') or startsWith(displayName, 'Legal-') or startsWith(displayName, 'Exec-') or startsWith(displayName, 'Sales-')"

    $entraAccess = foreach ($user in $entraUsers) {
        $groups = Get-MgUserMemberOf -UserId $user.Id
        $manager = if ($user.Manager) {
            $mgr = Get-MgUser -UserId ($user.Manager.Id -replace ".*/")
            $mgr.UserPrincipalName
        } else { $null }

        [PSCustomObject]@{
            UserPrincipalName = $user.UserPrincipalName
            DisplayName       = $user.DisplayName
            Department        = $user.Department
            JobTitle          = $user.JobTitle
            Level             = $user.JobTitle  # parsed externally
            EmployeeId        = $user.EmployeeId
            Manager           = $manager
            IdentitySource    = "EntraID"
            GroupMemberships  = ($groups | ForEach-Object { $_.DisplayName }) -join ";"
        }
    }

    # ── IAM Identity Center users and group/permission set assignments ──
    $instanceArn = aws ssoadmin list-instances --query "Instances[0].InstanceArn" --output text
    $identityStoreId = aws ssoadmin list-instances --query "Instances[0].IdentityStoreId" --output text

    $ssoUsers = aws identitystore list-users --identity-store-id $identityStoreId --query "Users" --output json | ConvertFrom-Json

    $ssoGroups = aws identitystore list-groups --identity-store-id $identityStoreId --query "Groups" --output json | ConvertFrom-Json

    $ssoAccess = foreach ($ssoUser in $ssoUsers) {
        $memberships = aws identitystore list-group-memberships-for-member `
            --identity-store-id $identityStoreId `
            --member-id "UserId=$($ssoUser.UserId)" `
            --query "GroupMemberships" `
            --output json | ConvertFrom-Json

        $groupNames = foreach ($m in $memberships) {
            $group = $ssoGroups | Where-Object { $_.GroupId -eq $m.GroupId }
            $group.DisplayName
        }

        # Map to Entra ID user by email
        $entraMatch = $entraUsers | Where-Object { $_.UserPrincipalName -eq $ssoUser.UserName }

        [PSCustomObject]@{
            UserPrincipalName = $ssoUser.UserName
            DisplayName       = "$($ssoUser.DisplayName.GivenName) $($ssoUser.DisplayName.FamilyName)"
            Department        = $entraMatch.Department
            JobTitle          = $ssoUser.EmployeeNumber  # used as role
            Level             = $null  # lookup from personnel roster
            EmployeeId        = $entraMatch.EmployeeId
            Manager           = $entraMatch.Manager.UserPrincipalName
            IdentitySource    = "IAMIdentityCenter"
            GroupMemberships  = ($groupNames -join ";")
        }
    }

    # ── Merge and export ──
    $allAccess = $entraAccess + $ssoAccess
    $allAccess | ConvertTo-Json -Depth 10 | Set-Content -Path $outputPath

    Write-CampaignLog -EventType "ACCESS_SNAPSHOT_EXPORTED" -Data @{
        total_users       = $allAccess.Count
        entraid_users     = $entraAccess.Count
        identity_center_users = $ssoAccess.Count
        output_path       = $outputPath
    }

    Write-Output "Exported $($allAccess.Count) user records to $outputPath"
    return $outputPath
}

# ── 2. Create Campaign ──────────────────────────────────────
function Start-CertificationCampaign {
    <#
    .SYNOPSIS
    Launches a quarterly certification campaign.
    .PARAMETER QuarterLabel
    e.g., "2026-Q3"
    .PARAMETER DryRun
    If true, generates campaign data without sending notifications.
    #>

    param(
        [Parameter(Mandatory)]
        [string]$QuarterLabel,

        [switch]$DryRun
    )

    Write-Output "Starting certification campaign: $QuarterLabel"
    Write-CampaignLog -EventType "CAMPAIGN_STARTED" -Data @{ quarter = $QuarterLabel; dry_run = $DryRun.IsPresent }

    # Step 1: Export current access state
    $snapshotPath = Export-AccessReviewData

    # Step 2: Group users by manager
    $snapshot = Get-Content $snapshotPath | ConvertFrom-Json
    $byManager = $snapshot | Group-Object Manager

    $campaignRecord = @{
        CampaignId  = $QuarterLabel
        CreatedAt   = (Get-Date -Format "o")
        Status      = if ($DryRun) { "DryRun" } else { "Active" }
        Reviewers   = @()
    }

    foreach ($managerGroup in $byManager) {
        $reviewerEmail = $managerGroup.Name
        if (-not $reviewerEmail) { continue }

        $reviewRecord = @{
            ReviewerEmail  = $reviewerEmail
            DirectReports  = $managerGroup.Group | ForEach-Object {
                @{
                    UserPrincipalName = $_.UserPrincipalName
                    DisplayName       = $_.DisplayName
                    Department        = $_.Department
                    IdentitySource    = $_.IdentitySource
                    CurrentAccess     = $_.GroupMemberships
                    LastReviewDate    = $null  # look up from previous campaign
                    LastDecision      = $null
                    Decision          = $null
                    DecisionDate      = $null
                    Status            = "Pending"
                }
            }
            Status         = "Pending"
            CreatedAt      = (Get-Date -Format "o")
            ReminderSent   = $false
            EscalationSent = $false
        }
        $campaignRecord.Reviewers += $reviewRecord

        if (-not $DryRun) {
            # Create Jira issue for reviewer
            $jiraBody = @{
                fields = @{
                    project     = @{ key = $script:JiraProjectKey }
                    summary     = "Access Certification — $QuarterLabel — $reviewerEmail"
                    description = "Please review and certify access for your direct reports by $(Get-Date)."
                    issuetype   = @{ name = $script:JiraIssueType }
                    assignee    = @{ name = $reviewerEmail }
                    duedate     = (Get-Date).AddDays(14).ToString("yyyy-MM-dd")
                }
            } | ConvertTo-Json -Depth 10

            $jiraHeaders = @{
                Authorization = "Basic $([Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("$env:JIRA_EMAIL:$env:JIRA_API_TOKEN")))"
                "Content-Type" = "application/json"
            }

            Invoke-RestMethod -Uri "$script:JiraBaseUrl/rest/api/3/issue" -Method Post `
                -Body $jiraBody -Headers $jiraHeaders | Out-Null

            # Send notification email (via SendGrid or SMTP)
            # & Send-ReviewNotification -To $reviewerEmail -Campaign $QuarterLabel
        }
    }

    # Save campaign record
    $campaignPath = Join-Path $script:CampaignDbPath "$QuarterLabel-campaign.json"
    $campaignRecord | ConvertTo-Json -Depth 10 | Set-Content -Path $campaignPath

    Write-CampaignLog -EventType "CAMPAIGN_CREATED" -Data @{
        quarter         = $QuarterLabel
        reviewer_count  = $campaignRecord.Reviewers.Count
        user_count      = $snapshot.Count
        dry_run         = $DryRun.IsPresent
    }

    Write-Output "Campaign $QuarterLabel created with $($campaignRecord.Reviewers.Count) reviewers"
    return $campaignPath
}

# ── 3. Process Reviewer Decision ────────────────────────────
function Submit-ReviewDecision {
    <#
    .SYNOPSIS
    Records a reviewer's decision for a specific user.
    .PARAMETER CampaignId
    .PARAMETER ReviewerEmail
    .PARAMETER UserPrincipalName
    .PARAMETER Decision
    "Approved", "Rejected", or "NoChange"
    .PARAMETER Comment
    #>

    param(
        [Parameter(Mandatory)] [string]$CampaignId,
        [Parameter(Mandatory)] [string]$ReviewerEmail,
        [Parameter(Mandatory)] [string]$UserPrincipalName,
        [Parameter(Mandatory)] [ValidateSet("Approved", "Rejected", "NoChange")] [string]$Decision,
        [string]$Comment = ""
    )

    $campaignPath = Join-Path $script:CampaignDbPath "$CampaignId-campaign.json"
    $campaign = Get-Content $campaignPath | ConvertFrom-Json

    $reviewer = $campaign.Reviewers | Where-Object { $_.ReviewerEmail -eq $ReviewerEmail }
    $directReport = $reviewer.DirectReports | Where-Object { $_.UserPrincipalName -eq $UserPrincipalName }

    if (-not $directReport) {
        Write-Error "User $UserPrincipalName not found under reviewer $ReviewerEmail"
        return
    }

    $directReport.Decision     = $Decision
    $directReport.DecisionDate = (Get-Date -Format "o")
    $directReport.Comment      = $Comment
    $directReport.Status       = if ($Decision -eq "Approved") { "Approved" } else { "ChangesRequired" }

    Write-CampaignLog -EventType "DECISION_SUBMITTED" -Data @{
        campaign_id  = $CampaignId
        reviewer     = $ReviewerEmail
        user         = $UserPrincipalName
        decision     = $Decision
        comment      = $Comment
    }

    $campaign | ConvertTo-Json -Depth 10 | Set-Content -Path $campaignPath

    # Check if reviewer has completed all reviews
    $pending = $reviewer.DirectReports | Where-Object { $_.Status -eq "Pending" }
    if ($pending.Count -eq 0) {
        $reviewer.Status = "Completed"
        Write-CampaignLog -EventType "REVIEWER_COMPLETED" -Data @{
            campaign_id = $CampaignId
            reviewer    = $ReviewerEmail
        }
    }

    Write-Output "Decision recorded: $UserPrincipalName → $Decision"
}

# ── 4. Send Reminders ───────────────────────────────────────
function Send-CampaignReminders {
    <#
    .SYNOPSIS
    Sends reminders to reviewers who haven't submitted decisions.
    .PARAMETER CampaignId
    .PARAMETER Day
    Campaign day (7 for first reminder, 14 for escalation)
    #>

    param(
        [Parameter(Mandatory)] [string]$CampaignId,
        [Parameter(Mandatory)] [ValidateSet(7, 14)] [int]$Day
    )

    $campaignPath = Join-Path $script:CampaignDbPath "$CampaignId-campaign.json"
    $campaign = Get-Content $campaignPath | ConvertFrom-Json

    $pendingReviewers = $campaign.Reviewers | Where-Object { $_.Status -ne "Completed" }

    foreach ($reviewer in $pendingReviewers) {
        if ($Day -eq 7) {
            if ($reviewer.ReminderSent) { continue }
            # Send reminder email to reviewer
            # & Send-Email -To $reviewer.ReviewerEmail -Template "Reminder-Day7"
            $reviewer.ReminderSent = $true

            Write-CampaignLog -EventType "REMINDER_SENT" -Data @{
                campaign_id = $CampaignId
                reviewer    = $reviewer.ReviewerEmail
                day         = 7
            }
        }
        elseif ($Day -eq 14) {
            if ($reviewer.EscalationSent) { continue }
            # Look up reviewer's manager from Entra ID
            Connect-MgGraph -Scopes "User.Read.All" -NoWelcome
            $reviewerUser = Get-MgUser -Filter "userPrincipalName eq '$($reviewer.ReviewerEmail)'"
            $managerUser  = Get-MgUser -UserId $reviewerUser.Manager.Id

            # Send escalation to reviewer's manager
            # & Send-Email -To $managerUser.UserPrincipalName -Template "Escalation-Day14" -Data @{ Reviewer = $reviewer.ReviewerEmail }
            $reviewer.EscalationSent = $true

            Write-CampaignLog -EventType "ESCALATION_SENT" -Data @{
                campaign_id  = $CampaignId
                reviewer     = $reviewer.ReviewerEmail
                escalated_to = $managerUser.UserPrincipalName
                day          = 14
            }
        }
    }

    $campaign | ConvertTo-Json -Depth 10 | Set-Content -Path $campaignPath
    Write-Output "Reminders sent for day $Day: $($pendingReviewers.Count) pending reviewers"
}

# ── 5. Enforce Decisions ────────────────────────────────────
function Invoke-CertificationEnforcement {
    <#
    .SYNOPSIS
    Enforces all certification decisions at campaign deadline.
    Revokes access for rejected or unreviewed users.
    #>

    param(
        [Parameter(Mandatory)] [string]$CampaignId,
        [switch]$WhatIf
    )

    $campaignPath = Join-Path $script:CampaignDbPath "$CampaignId-campaign.json"
    $campaign = Get-Content $campaignPath | ConvertFrom-Json

    $toRevoke = $campaign.Reviewers |
        ForEach-Object { $_.DirectReports } |
        Where-Object { $_.Status -ne "Approved" }

    Write-Output "Users to revoke: $($toRevoke.Count)"

    $enforcementLog = @()

    foreach ($user in $toRevoke) {
        Write-Output "Revoking: $($user.UserPrincipalName)"

        if ($user.IdentitySource -eq "EntraID") {
            # Remove from all non-default groups
            Connect-MgGraph -Scopes "Group.ReadWrite.All", "User.ReadWrite.All" -NoWelcome
            $entraUser = Get-MgUser -Filter "userPrincipalName eq '$($user.UserPrincipalName)'"
            $memberships = Get-MgUserMemberOf -UserId $entraUser.Id

            foreach ($groupRef in $memberships) {
                $groupId = $groupRef.Id
                $group = Get-MgGroup -GroupId $groupId
                # Skip default groups (All Users, etc.)
                if ($group.DisplayName -match "^AWS-|Dept$|Team$") {
                    if (-not $WhatIf) {
                        Remove-MgGroupMemberByRef -GroupId $groupId -DirectoryObjectId $entraUser.Id
                    }
                    Write-Output "  Removed from Entra ID group: $($group.DisplayName)"
                }
            }
        }
        else {
            # IAM Identity Center — remove from all groups
            $identityStoreId = aws ssoadmin list-instances --query "Instances[0].IdentityStoreId" --output text
            $ssoUserId = aws identitystore get-user-id `
                --identity-store-id $identityStoreId `
                --alternate-identifier "UniqueAttribute={AttributePath=UserName,AttributeValue=$($user.UserPrincipalName)}" `
                --query "UserId" --output text

            $memberships = aws identitystore list-group-memberships-for-member `
                --identity-store-id $identityStoreId `
                --member-id "UserId=$ssoUserId" `
                --query "GroupMemberships" --output json | ConvertFrom-Json

            foreach ($m in $memberships) {
                if (-not $WhatIf) {
                    aws identitystore delete-group-membership `
                        --identity-store-id $identityStoreId `
                        --membership-id $m.MembershipId | Out-Null
                }
                Write-Output "  Removed from IAM Identity Center group: $($m.GroupId)"
            }
        }

        $enforcementLog += [PSCustomObject]@{
            UserPrincipalName   = $user.UserPrincipalName
            Decision            = $user.Decision
            IdentitySource      = $user.IdentitySource
            Action              = if ($WhatIf) { "PendingRevocation" } else { "Revoked" }
            Timestamp           = (Get-Date -Format "o")
        }
    }

    # Save enforcement log
    $logPath = Join-Path $script:CampaignDbPath "$CampaignId-enforcement-log.json"
    $enforcementLog | ConvertTo-Json -Depth 10 | Set-Content -Path $logPath

    Write-CampaignLog -EventType "ENFORCEMENT_COMPLETED" -Data @{
        campaign_id = $CampaignId
        revoked     = ($enforcementLog | Where-Object { $_.Action -eq "Revoked" }).Count
        what_if     = $WhatIf.IsPresent
    }

    Write-Output "Enforcement complete. Log: $logPath"
}

# ── 6. Generate Audit Package ───────────────────────────────
function Export-CampaignAuditPackage {
    <#
    .SYNOPSIS
    Generates auditor-ready evidence package for a completed campaign.
    #>

    param(
        [Parameter(Mandatory)] [string]$CampaignId
    )

    $campaignPath = Join-Path $script:CampaignDbPath "$CampaignId-campaign.json"
    $enforcementPath = Join-Path $script:CampaignDbPath "$CampaignId-enforcement-log.json"
    $campaign = Get-Content $campaignPath | ConvertFrom-Json

    $auditPackage = @{
        CampaignId   = $CampaignId
        GeneratedAt  = (Get-Date -Format "o")
        CampaignStatus = $campaign.Status
        Summary = @{
            TotalReviewers     = $campaign.Reviewers.Count
            CompletedReviewers = ($campaign.Reviewers | Where-Object { $_.Status -eq "Completed" }).Count
            TotalUsers         = ($campaign.Reviewers | ForEach-Object { $_.DirectReports }).Count
            Approved           = ($campaign.Reviewers | ForEach-Object { $_.DirectReports } | Where-Object { $_.Decision -eq "Approved" }).Count
            Rejected           = ($campaign.Reviewers | ForEach-Object { $_.DirectReports } | Where-Object { $_.Decision -eq "Rejected" }).Count
            Unreviewed         = ($campaign.Reviewers | ForEach-Object { $_.DirectReports } | Where-Object { $_.Decision -eq $null }).Count
        }
        Reviewers    = $campaign.Reviewers | ForEach-Object {
            @{
                ReviewerEmail = $_.ReviewerEmail
                Status        = $_.Status
                CompletedAt   = if ($_.Status -eq "Completed") { $_.DecisionDate } else { $null }
                Decisions     = $_.DirectReports | ForEach-Object {
                    @{
                        User  = $_.UserPrincipalName
                        Decision = $_.Decision
                        Date  = $_.DecisionDate
                    }
                }
            }
        }
        Enforcement  = if (Test-Path $enforcementPath) {
            Get-Content $enforcementPath | ConvertFrom-Json
        } else { @() }
        AuditLog     = @()  # Splunk query results would be embedded here
    }

    $auditPath = Join-Path $script:CampaignDbPath "$CampaignId-audit-package.json"
    $auditPackage | ConvertTo-Json -Depth 10 | Set-Content -Path $auditPath

    Write-Output "Audit package generated: $auditPath"
    return $auditPath
}

Export-ModuleMember -Function Export-AccessReviewData, Start-CertificationCampaign, Submit-ReviewDecision, Send-CampaignReminders, Invoke-CertificationEnforcement, Export-CampaignAuditPackage
```

---

## 2. Campaign Lifecycle Automation — Scheduled Runbook

### `Invoke-QuarterlyCertification.ps1`

```powershell
# Invoke-QuarterlyCertification.ps1
# Scheduled to run on the first day of each quarter (Azure Automation)

param(
    [Parameter(Mandatory)]
    [ValidateSet("Q1", "Q2", "Q3", "Q4")]
    [string]$Quarter,

    [switch]$DryRun
)

$year = (Get-Date).Year
$quarterLabel = "$year-$Quarter"

# Import module
Import-Module "$PSScriptRoot\Certification-Campaign.psm1" -Force

# Step 1: Launch campaign
$campaignPath = Start-CertificationCampaign -QuarterLabel $quarterLabel -DryRun:$DryRun.IsPresent
Write-Output "Campaign launched: $quarterLabel"

# Step 2: Schedule reminder tasks (via Azure Automation schedules)
# Day 7 reminder and Day 14 escalation are handled by separate scheduled runbooks
# that call Send-CampaignReminders

# Step 3: Schedule enforcement at Day 21
if (-not $DryRun) {
    $enforcementDate = (Get-Date).AddDays(21)
    Register-ScheduledJob -Name "Enforce-$quarterLabel" `
        -ScriptBlock {
            Import-Module "$using:PSScriptRoot\Certification-Campaign.psm1" -Force
            Invoke-CertificationEnforcement -CampaignId "$using:quarterLabel"
            Export-CampaignAuditPackage -CampaignId "$using:quarterLabel"
        } -RunNow -At $enforcementDate

    Write-Output "Enforcement scheduled for $enforcementDate"
}
```

---

## 3. AWS CLI: Access Analyzer for Stale Access Detection

### Identify Unused IAM Identity Center Assignments

```bash
# List all IAM Identity Center users
IDENTITY_STORE_ID=$(aws ssoadmin list-instances --query "Instances[0].IdentityStoreId" --output text)

# Get all users
aws identitystore list-users \
    --identity-store-id $IDENTITY_STORE_ID \
    --query "Users[].UserId" \
    --output json > sso-users.json

# For each user, check last sign-in via CloudTrail
# (Pseudo — requires CloudTrail Lake or Athena)
aws athena start-query-execution \
    --query-string "
        SELECT
            useridentity.arn,
            eventname,
            MAX(eventtime) as last_access
        FROM cloudtrail_logs
        WHERE useridentity.arn LIKE '%sso.amazonaws.com%'
        GROUP BY useridentity.arn, eventname
        HAVING MAX(eventtime) < current_timestamp - INTERVAL '45' DAY
    " \
    --work-group IAM-Analytics \
    --output-table iam_access_analyzer.findings
```

### Generate Access Analyzer Findings Report

```bash
aws accessanalyzer list-findings \
    --analyzer-arn arn:aws:accessanalyzer:eu-west-2:222222222222:analyzer/IAMAccessAnalyzer \
    --query 'findings[?status==`ACTIVE`].[resource,principal,condition,updatedAt]' \
    --output table
```

---

## 4. AWS Console: Monthly Access Review Dashboard

### Manual Steps (for auditors)

1. **IAM Access Analyzer**
   - Navigate to AWS Console → IAM → Access Analyzer
   - Review **Active findings** → filter by `UnusedPermission` and `UnusedRole`
   - Export findings to CSV

2. **IAM Identity Center**
   - Navigate to AWS IAM Identity Center → **Dashboard**
   - Review **Users and groups** → export user list
   - Review **AWS accounts** → verify permission set assignments

3. **CloudTrail**
   - Navigate to CloudTrail → **Event history**
   - Filter `userIdentity.type = AssumedRole` and `eventSource = sso.amazonaws.com`
   - Review IAM Identity Center sign-in activity

### Creating an IAM Access Analyzer

```bash
# Ensure Access Analyzer is enabled in the Security account
aws accessanalyzer create-analyzer \
    --analyzer-name IAMAccessAnalyzer \
    --type ACCOUNT \
    --profile security-account
```

---

## 5. Entra ID: Access Reviews (Alternative Built-in Approach)

As an alternative to the custom PowerShell module, Entra ID has a built-in Access Reviews feature:

### Configure in Entra ID Admin Centre

1. **Identity Governance** → **Access reviews** → **New access review**
2. Select:
   - **Review what**: `Microsoft Entra ID groups` (select `AWS-*` groups)
   - **Scope**: `All users`
   - **Reviewers**: `Manager`
   - **Duration**: `14 days`
   - **Auto-apply**: `Remove access` if reviewer doesn't respond
   - **Reminders**: Enable, `7 days` after start
3. Schedule recurrence: **Quarterly**

### Equivalent via Microsoft Graph API

```powershell
# Create access review schedule via Graph
$body = @{
    displayName = "Q3 2026 AWS Access Review"
    descriptionForReviewers = "Review and certify your team's AWS access"
    scope = @{
        "@odata.type" = "#microsoft.graph.accessReviewQueryScope"
        query = "/groups?`$filter=startsWith(displayName,'AWS-')"
        queryType = "MicrosoftGraph"
    }
    reviewers = @(@{
        query = "/users?`$filter=userType eq 'Member'"
        queryType = "MicrosoftGraph"
        queryRoot = $null
    })
    settings = @{
        mailNotificationsEnabled = $true
        reminderNotificationsEnabled = $true
        justificationRequiredOnApproval = $true
        defaultDecisionEnabled = $false
        autoApplyDecisionsEnabled = $true
        recommendationsEnabled = $true
        recurrence = @{
            pattern = @{
                type = "absoluteMonthly"
                interval = 3
            }
            range = @{
                type = "numbered"
                numberOfOccurrences = 4
                startDate = "2026-07-01"
            }
        }
        applyActions = @(@{
            "@odata.type" = "#microsoft.graph.removeAccessApplyAction"
        })
    }
} | ConvertTo-Json -Depth 10

Invoke-MgGraphRequest -Method POST `
    -Uri "https://graph.microsoft.com/v1.0/identityGovernance/accessReviews/definitions" `
    -Body $body
```

---

## 6. Testing

### Test Script — `Test-CertificationCampaign.ps1`

```powershell
# Test the full campaign lifecycle

Import-Module "$PSScriptRoot\Certification-Campaign.psm1" -Force

# ── Dry run ──
Write-Output "=== Dry Run ==="
$campaignPath = Start-CertificationCampaign -QuarterLabel "2026-Q3-DRYRUN" -DryRun
$campaign = Get-Content $campaignPath | ConvertFrom-Json

Write-Output "Reviewers: $($campaign.Reviewers.Count)"
Write-Output "Sample reviewer: $($campaign.Reviewers[0].ReviewerEmail) — $($campaign.Reviewers[0].DirectReports.Count) direct reports"

# ── Submit test decisions ──
Write-Output "`n=== Submit Decisions ==="
$sampleReviewer = $campaign.Reviewers[0]
$sampleUser = $sampleReviewer.DirectReports[0]
Submit-ReviewDecision -CampaignId "2026-Q3-DRYRUN" `
    -ReviewerEmail $sampleReviewer.ReviewerEmail `
    -UserPrincipalName $sampleUser.UserPrincipalName `
    -Decision "Approved"

Submit-ReviewDecision -CampaignId "2026-Q3-DRYRUN" `
    -ReviewerEmail $sampleReviewer.ReviewerEmail `
    -UserPrincipalName $sampleReviewer.DirectReports[1].UserPrincipalName `
    -Decision "Rejected" `
    -Comment "No longer needs production access"

# ── Test enforcement (WhatIf) ──
Write-Output "`n=== Enforcement (WhatIf) ==="
Invoke-CertificationEnforcement -CampaignId "2026-Q3-DRYRUN" -WhatIf

# ── Generate audit package ──
Write-Output "`n=== Audit Package ==="
Export-CampaignAuditPackage -CampaignId "2026-Q3-DRYRUN"
```

### Validation Queries

```bash
# Verify no orphaned assignments remain after enforcement
aws ssoadmin list-account-assignments \
    --instance-arn arn:aws:sso:::instance/ssoins-12345678 \
    --account-id 333333333333 \
    --query "AccountAssignments[?PrincipalType=='USER']" \
    --output table

# Check Splunk for campaign events
curl -k -X POST "https://hec.splunk.nexusbridge.co.uk:8089/services/search/jobs" \
    -d 'search=sourcetype="iam:certification" event_type="ENFORCEMENT_COMPLETED"' \
    -u "admin:$SPLUNK_PASS"
```

---

## 7. Expected Outcomes

| User Story | Acceptance Criteria | Verification |
|-----------|-------------------|-------------|
| **US-01** Campaign Launch | AC-01: Complete user+access snapshot generated | Snapshot JSON contains all 280+ users |
| | AC-02: Each user assigned to manager | No user with `Manager = null` except C-Suite |
| **US-02** Manager Review | AC-03: Dashboard shows user, access, last review | Dashboard view of Priya Sharma's 4 reports |
| **US-03** Escalation | AC-04: Reminder at day 7 | Splunk event `REMINDER_SENT` |
| | AC-05: Escalation at day 14 | Splunk event `ESCALATION_SENT` |
| | AC-06: Auto-revoke at day 21 | Splunk event `ENFORCEMENT_COMPLETED` with revoked count |
| **US-04** Audit | AC-07: Evidence package with all details | JSON with summary + reviewer decisions + enforcement |
| | AC-08: Revocations logged to Splunk | `ENFORCEMENT_COMPLETED` contains per-user revocation log |
