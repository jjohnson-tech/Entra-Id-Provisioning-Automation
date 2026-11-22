# Entra ID User Provisioning Automation: Step-by-Step Lab

![PowerShell](https://img.shields.io/badge/PowerShell-7.x-5391FE?logo=powershell&logoColor=white)
![Microsoft Graph](https://img.shields.io/badge/Microsoft%20Graph-SDK%202.x-0078D4?logo=microsoft&logoColor=white)
![Entra ID](https://img.shields.io/badge/Microsoft%20Entra%20ID-P2-0078D4?logo=microsoftazure&logoColor=white)
![Auth](https://img.shields.io/badge/Auth-App--only%20%2B%20Certificate-success)

A hands-on lab that automates the **Joiner → Leaver** identity lifecycle in Microsoft Entra ID using **PowerShell 7** and the **Microsoft Graph PowerShell SDK**. New hires are read from an HR export, given accounts, managers, role-based groups, and licenses automatically. Leavers are disabled, signed out everywhere, and stripped of access. The finished automation runs **unattended every morning** using **certificate-based app-only authentication**, with every change written to an audit log.

This README walks through **every step I took**, in order, so anyone can rebuild the lab from an empty tenant.

> **Lab only.** Built in an isolated Entra ID tenant with fictional employees. Tenant IDs, client IDs, object IDs, certificate thumbprints, and admin accounts are redacted in all screenshots.

<!-- HOW TO ADD SCREENSHOTS (this comment is invisible on GitHub):
1. On github.com, open README.md and click the pencil (Edit).
2. Delete a 📸 box and drag the screenshot into that spot (or paste with Ctrl+V).
3. Wait for "Uploading..." to finish. GitHub inserts the image link for you.
4. No screenshot for a box? Delete the box.
5. Blur sensitive values BEFORE uploading. Uploaded image links are public.
-->

---

## Table of Contents

1. [What This Lab Builds](#what-this-lab-builds)
2. [Business Problem](#business-problem)
3. [Lab Environment](#lab-environment)
4. [Repository Contents](#repository-contents)
5. [Step 0: Create the Lab Tenant](#step-0-create-the-lab-tenant)
6. [Step 1: Install Microsoft Graph PowerShell and Connect](#step-1-install-microsoft-graph-powershell-and-connect)
7. [Step 2: Build the HR Data File](#step-2-build-the-hr-data-file)
8. [Step 3: Department Groups and Group-Based Licensing](#step-3-department-groups-and-group-based-licensing)
9. [Step 4: The Role Map](#step-4-the-role-map)
10. [Step 5: The Joiner Script](#step-5-the-joiner-script)
11. [Step 6: The Leaver Script](#step-6-the-leaver-script)
12. [Step 7: Error Handling and Audit Logging](#step-7-error-handling-and-audit-logging)
13. [Step 8: Unattended App-Only Automation](#step-8-unattended-app-only-automation)
14. [Resetting the Lab](#resetting-the-lab)
15. [Troubleshooting](#troubleshooting)
16. [Key Concepts](#key-concepts)
17. [Security Considerations](#security-considerations)
18. [What I'd Do Differently](#what-id-do-differently)
19. [Skills Demonstrated](#skills-demonstrated)

---

## What This Lab Builds

| Component | What it does |
|---|---|
| **Joiner script** | Reads new hires from an HR CSV → creates accounts with unique usernames and temp passwords → sets managers → adds role-based groups → licenses users via group membership |
| **Leaver script** | Reads terminations from an HR CSV → disables the account → revokes all sign-in sessions → removes all groups (frees the license) |
| **Role map** | A JSON file that maps each department to its groups (RBAC rules kept separate from code) |
| **Audit log** | One CSV row per action: when, who (person or app), what, result, error |
| **App-only automation** | An Entra app registration with a certificate, so the joiner runs on a schedule with no human sign-in |

**Safety features built in:** dry run by default, safe to rerun (idempotent), fail-closed on unknown departments, a duplicate-ID guard, a "not connected" guard, separate error handling per step, and offboarded users are never re-granted access.

---

## Business Problem

HR is the source of truth for who works at a company, but accounts are often created and removed by hand. That causes:

| Problem | Risk |
|---|---|
| New hires wait for accounts | Lost day-one productivity |
| Access granted inconsistently ("copy Bob's access") | Over-privileged users |
| Leavers keep working credentials and sessions | Access after termination (a top audit finding) |
| Licenses stay on departed users | Wasted spend |
| No record of changes | Failed audits (NIST 800-53 AC-2, CIS Controls 5 and 6) |

**Goal:** access driven by HR data and role rules, applied the same way every time, removed immediately on exit, fully logged, and running on its own.

---

## Lab Environment

| Item | Value |
|---|---|
| Tenant | Standalone Microsoft Entra ID tenant, renamed **JJ Tech Lab IAM** (created from a free Azure account) |
| License | Microsoft Entra ID Free → **Entra ID P2 trial** (30 days) for group-based licensing |
| Admin account | Cloud-only Global Administrator (`admin@<tenant>.onmicrosoft.com`), MFA via security defaults |
| Workstation | Windows 11 |
| PowerShell | PowerShell 7.x (Microsoft Store build) |
| Graph SDK | Microsoft.Graph 2.41.2 |
| Working folder | `C:\IAM Labs\Project1` (kept **outside Documents**; see Step 1) |
| Automation identity | App registration **IAM-Provisioning-Automation** + self-signed certificate (12 months) |
| Scheduler | Windows Task Scheduler, daily at 6:00 AM |

### Fictional org chart

```
I'teeya Harris (CEO, Executive)
├── Roy James       (IT Manager)        → Jamie King, Veronica Mars, Tina Mcphearson
├── Daniel Turner   (Finance Manager)   → Betty Hart, Heather Jackson, Jamal Montgomery, Andre Lewis
├── Monica Dampier  (HR Manager)        → Michael Johnson, Mia Worthy, Janay Hill
└── Star Harris     (Sales Manager)     → Isais Johnson, Gilbert Arenas, Michael Johnson, Kendra Wallace
```

Deliberate edge cases: an apostrophe in a name (I'teeya), and **two employees named Michael Johnson** in different departments.

---

## Repository Contents

| File | Purpose |
|---|---|
| `README.md` | This walkthrough |
| `Invoke-Joiner.ps1` | Joiner: accounts, managers, groups (3 passes) |
| `Invoke-Leaver.ps1` | Leaver: disable, revoke sessions, remove groups |
| `IAMHelpers.ps1` | Shared functions: `Connect-IamGraph` (app sign-in) and `Write-IamLog` (audit log) |
| `hr_new_hires.csv` | Sample HR new-hire export (19 fictional employees) |
| `hr_leavers.csv` | Sample HR terminations (includes a non-existent ID on purpose) |
| `role_map.json` | Department → groups |
| `app_config.example.json` | Template for `app_config.json` (TenantId, ClientId, Thumbprint) |
| `logs/sample_iam_actions.csv` | Sanitized sample of the audit log |
| `.gitignore` | Keeps secrets out of Git |

**Never committed:** `app_config.json`, `temp_passwords.csv`, live logs, certificates.

> To reuse the scripts: put all files in one folder and change `$domain` at the top of `Invoke-Joiner.ps1` to your tenant's domain. The scripts find everything else relative to their own folder (`$PSScriptRoot`).

---

## Step 0: Create the Lab Tenant

**Why:** a separate tenant gives a clean, isolated environment where test users and broad permissions can't affect anything real.

### 0.1 Create a free Azure account (this creates the tenant)
1. Open a **private/InPrivate browser window**.
2. Create a new Microsoft account at **outlook.com**, used only for this lab.
3. Go to **azure.microsoft.com/free → Try Azure for free** and sign in with it.
4. Verify phone and card. The free account doesn't charge unless upgraded. If you've used a free trial before, choose **Pay-As-You-Go**; Entra ID Free still costs $0.

Azure automatically creates a tenant called **Default Directory** with an auto-generated domain like `<name>outlook.onmicrosoft.com`.

<img width="1408" height="647" alt="AzurePortal_Homescreen" src="https://github.com/user-attachments/assets/9315ee82-8566-4599-9f24-b875ef19d10e" />

### 0.2 Rename the tenant
**entra.microsoft.com → Entra ID → Overview → Properties** → Name: `JJ Tech Lab IAM` → **Save**.

<img width="867" height="516" alt="Tenant_rename" src="https://github.com/user-attachments/assets/745a3ed7-a4e6-43d3-b63f-f626d5d7005c" />

<img width="842" height="500" alt="Tenant_overview" src="https://github.com/user-attachments/assets/55a9d049-ace6-4bfd-b24c-66c7cd40c39b" />

### 0.3 Create a native admin account
The outlook.com account appears inside the tenant as an external account (`#EXT#`). Real organizations use admin accounts that belong to the tenant itself.

1. **Users → All users → + New user → Create new user**
2. UPN: `admin@<your-domain>` · Display name: `JJ Lab Admin` · copy the generated password
3. **Assignments → Add role → Global Administrator → Review + create**
4. In a new private window, sign in as the new admin, set a password, and register **Microsoft Authenticator** (security defaults require MFA).
5. Note the **Tenant ID** and **Primary domain** from the Overview page.

<img width="1138" height="616" alt="Admin_globaladminrole" src="https://github.com/user-attachments/assets/dc33d0d1-f2be-4e23-8275-00a903032742" />

---

## Step 1: Install Microsoft Graph PowerShell and Connect

**Why:** Microsoft Graph is the single API behind Entra ID. The Graph PowerShell SDK (`*-Mg*` cmdlets) replaces the retired AzureAD and MSOnline modules.

### 1.1 Check PowerShell
```powershell
$PSVersionTable.PSVersion
```
Use **7.x**. If you see 5.1, install PowerShell 7 and open **pwsh**:
```powershell
winget install --id Microsoft.PowerShell --source winget
```

<img width="930" height="433" alt="Screenshot 2026-11-03 210249" src="https://github.com/user-attachments/assets/ca947eca-f69e-464e-9248-8a9374cd28ba" />

### 1.2 Create the working folder (outside Documents)
```powershell
New-Item -Path "C:\IAM Labs\Project1" -ItemType Directory -Force
```
Windows **Controlled Folder Access** (ransomware protection) can block PowerShell from writing inside `Documents`. Keeping the lab outside it avoids blocked log writes later.

### 1.3 Install the Graph SDK
```powershell
Install-PSResource -Name Microsoft.Graph -Scope CurrentUser -TrustRepository
Get-InstalledPSResource Microsoft.Graph | Select-Object Name, Version
```
`Install-PSResource` is the newer installer built into PowerShell 7.4+.

**If the install is blocked** (`Install-Module` reports "Administrator rights are required" even with `-Scope CurrentUser`), Controlled Folder Access is the cause. Allow PowerShell instead of turning protection off:
**Windows Security → Virus & threat protection → Manage ransomware protection → Allow an app through Controlled folder access → Add an allowed app → Recently blocked apps → pwsh.exe**

<img width="1423" height="427" alt="Screenshot 2026-11-03 211406" src="https://github.com/user-attachments/assets/d3a9458c-612d-481b-9ec9-17eafe6a237a" />

<img width="1222" height="801" alt="Screenshot 2026-11-03 211856" src="https://github.com/user-attachments/assets/df17196d-f09d-41f7-b11a-15c19d76b087" />

<img width="1335" height="666" alt="Screenshot 2026-11-03 212312" src="https://github.com/user-attachments/assets/dd925ecb-1473-4764-b098-e15eed0a64c2" />

### 1.4 Connect (delegated, as the admin)
**Scopes** are the permissions the session requests. Ask only for what's needed (least privilege):

```powershell
Set-MgGraphOption -DisableLoginByWAM $true     # one-time; avoids a sign-in loop with the Windows broker
Connect-MgGraph -TenantId "<your-domain>.onmicrosoft.com" `
    -Scopes "User.ReadWrite.All","Group.ReadWrite.All" -ContextScope Process
```
Sign in as **admin@<your-domain>** (not the outlook.com account) and click **Accept** on the permissions prompt.

```powershell
Get-MgContext                                              # who and what you're connected as
Get-MgUser -All | Select-Object DisplayName, UserPrincipalName
```

<img width="857" height="212" alt="Screenshot 2026-11-03 213741" src="https://github.com/user-attachments/assets/a7e01aec-74c2-4377-8273-30bc08479d17" />

<img width="1285" height="658" alt="Screenshot 2026-11-03 214937" src="https://github.com/user-attachments/assets/39191232-f40a-400c-96c5-0e94dd0e8692" />

<img width="1303" height="577" alt="Screenshot 2026-11-03 215058" src="https://github.com/user-attachments/assets/0826b7f0-9078-48c7-b7c7-d6246ce4f1f1" />

---

## Step 2: Build the HR Data File

**Why:** HR is the source of truth. The scripts only act on what this file says, so the data has to be clean.

### 2.1 Columns
| Column | Purpose |
|---|---|
| `EmployeeId` | Unique, permanent ID. Scripts match people on this, **never on name** |
| `FirstName`, `LastName` | Build the username |
| `Department` | Looked up in the role map to decide groups |
| `JobTitle` | Profile field |
| `ManagerEmployeeId` | The manager's EmployeeId (blank only for the CEO) |
| `UsageLocation` | Country code; **required before a license can be assigned** |

Save as `C:\IAM Labs\Project1\hr_new_hires.csv` (see the sample file in this repo).

### 2.2 Validate before using it
```powershell
$hr  = Import-Csv "C:\IAM Labs\Project1\hr_new_hires.csv"
$hr.Count

# Duplicate EmployeeIds (should return nothing)
$hr | Group-Object EmployeeId | Where-Object Count -gt 1 | Select-Object Name, Count

# Department spelling (exactly 5)
$hr | Group-Object Department | Select-Object Name, Count

# Manager IDs that don't exist (should return nothing)
$ids = $hr.EmployeeId
$hr | Where-Object { $_.ManagerEmployeeId -and $_.ManagerEmployeeId -notin $ids } |
    Select-Object EmployeeId, FirstName, ManagerEmployeeId

# Blank UsageLocation (should return nothing)
$hr | Where-Object { -not $_.UsageLocation } | Select-Object EmployeeId, FirstName

# Extra spaces in titles (should return nothing)
$hr | Where-Object { $_.JobTitle -ne $_.JobTitle.Trim() } | Select-Object EmployeeId, JobTitle

# Name clash test case (should show Michael Johnson, 2)
$hr | Group-Object FirstName, LastName | Where-Object Count -gt 1 | Select-Object Name, Count
```
My first draft had a duplicated row, misspelled departments ("Finacne"), wrong manager IDs, a stray comma, and leading spaces. All of them were fixed before provisioning.

<img width="1092" height="728" alt="image" src="https://github.com/user-attachments/assets/d738c6a4-6541-4e3e-9dd6-8c75a0d998a9" />


<img width="1007" height="661" alt="image" src="https://github.com/user-attachments/assets/f7f59306-bc48-4541-9080-00818f75eaf9" />


---

## Step 3: Department Groups and Group-Based Licensing

**Why:** access is given to **groups**, not individuals. Attaching a license to a group means anyone added to the group is licensed automatically, and anyone removed frees the license. Group-based licensing requires **Entra ID P1 or higher**.

### 3.1 Start the Entra ID P2 trial
**entra.microsoft.com → Billing → Licenses → All products → + Try / Buy → Microsoft Entra ID P2 → Free trial → Activate**

The trial lasts 30 days and is limited to one per tenant.

<img width="1108" height="562" alt="Screenshot 2026-11-05 183915" src="https://github.com/user-attachments/assets/6192c5a3-a870-4087-9652-4eeea6e56fcf" />

### 3.2 Reconnect with a scope to read licenses
```powershell
Connect-MgGraph -TenantId "<your-domain>.onmicrosoft.com" `
    -Scopes "User.ReadWrite.All","Group.ReadWrite.All","Organization.Read.All" -ContextScope Process
```

<img width="875" height="892" alt="Screenshot 2026-11-05 184319" src="https://github.com/user-attachments/assets/0a6f095c-2bd1-46b6-b040-498782a94220" />

### 3.3 Create the department groups (safe to rerun)
Naming convention: `SG-Dept-<Name>` (SG = security group). Entra **allows duplicate group names**, so the loop checks before creating:
```powershell
$depts = "Executive","IT","Finance","HR","Sales"
foreach ($d in $depts) {
    $name = "SG-Dept-$d"
    if (Get-MgGroup -Filter "displayName eq '$name'") {
        Write-Host "$name already exists - skipping" -ForegroundColor Yellow
    } else {
        New-MgGroup -DisplayName $name -MailEnabled:$false -MailNickname $name -SecurityEnabled `
            -Description "$d department - managed by provisioning script"
        Write-Host "$name created" -ForegroundColor Green
    }
}
Get-MgGroup -Filter "startswith(displayName,'SG-Dept-')" | Select-Object DisplayName, Id
```
`-MailEnabled:$false` + `-SecurityEnabled` = a security group (not an email/Teams group). `-MailNickname` is required by Graph even without email.

<img width="963" height="518" alt="Screenshot 2026-11-05 185613" src="https://github.com/user-attachments/assets/36a9dd28-eb53-4f77-a755-c0a6410bbaf4" />

### 3.4 Attach the P2 license to each department group
Look the license up by name rather than pasting its ID:
```powershell
Get-MgSubscribedSku | Select-Object SkuPartNumber, SkuId, ConsumedUnits

$sku    = Get-MgSubscribedSku | Where-Object SkuPartNumber -eq "AAD_PREMIUM_P2"
$groups = Get-MgGroup -Filter "startswith(displayName,'SG-Dept-')"
foreach ($g in $groups) {
    Set-MgGroupLicense -GroupId $g.Id -AddLicenses @(@{SkuId = $sku.SkuId}) -RemoveLicenses @()
    Write-Host "License attached to $($g.DisplayName)" -ForegroundColor Green
}

# Verify
Get-MgGroup -Filter "startswith(displayName,'SG-Dept-')" -Property DisplayName,AssignedLicenses |
    Select-Object DisplayName, @{n='LicenseSkuId';e={$_.AssignedLicenses.SkuId}}
```
`ConsumedUnits` stays **0** until users are actually in the groups.

<img width="957" height="546" alt="Screenshot 2026-11-05 185703" src="https://github.com/user-attachments/assets/563a2145-b711-4373-8013-998b1799bc9d" />

<img width="907" height="242" alt="Screenshot 2026-11-05 190957" src="https://github.com/user-attachments/assets/7c41d701-9b30-4aa7-9ea5-bac0bc7f4283" />

<img width="1145" height="506" alt="Screenshot 2026-11-05 191215" src="https://github.com/user-attachments/assets/edd54a3b-3b2d-43a8-990c-4a1ea16f8d15" />

---

## Step 4: The Role Map

**Why:** the role map is the RBAC rulebook: "if you're in this department, you get these groups." Keeping it in its own file means access rules change without code changes, and auditors can read them in one place.

### 4.1 Create the all-employees group (no license on this one)
```powershell
$name = "SG-All-Employees"
if (-not (Get-MgGroup -Filter "displayName eq '$name'")) {
    New-MgGroup -DisplayName $name -MailEnabled:$false -MailNickname $name -SecurityEnabled `
        -Description "All employees - managed by provisioning script"
}
```

### 4.2 Write `role_map.json`
```json
{
  "Executive": ["SG-Dept-Executive", "SG-All-Employees"],
  "IT":        ["SG-Dept-IT",        "SG-All-Employees"],
  "Finance":   ["SG-Dept-Finance",   "SG-All-Employees"],
  "HR":        ["SG-Dept-HR",        "SG-All-Employees"],
  "Sales":     ["SG-Dept-Sales",     "SG-All-Employees"]
}
```
Department names must match the HR file **exactly**.

### 4.3 Validate the map
```powershell
$roleMap = Get-Content "C:\IAM Labs\Project1\role_map.json" -Raw | ConvertFrom-Json
$roleMap.Finance                                   # lookup test

# Every HR department exists in the map (should return nothing)
$hr.Department | Sort-Object -Unique | Where-Object { -not $roleMap.$_ }

# Every mapped group exists in Entra (6 green OKs)
$roleMap.PSObject.Properties.Value | Sort-Object -Unique | ForEach-Object {
    if (Get-MgGroup -Filter "displayName eq '$_'") { Write-Host "OK: $_" -ForegroundColor Green }
    else { Write-Host "MISSING: $_" -ForegroundColor Red }
}
```

<img width="1386" height="675" alt="Screenshot 2026-11-05 193235" src="https://github.com/user-attachments/assets/717938c6-cba7-4f24-b5dd-0325282adcd4" />

<img width="928" height="545" alt="Screenshot 2026-11-05 193904" src="https://github.com/user-attachments/assets/5b137451-0b1e-431a-9afd-425a9944e190" />

---

## Step 5: The Joiner Script

Copy **`Invoke-Joiner.ps1`** and **`IAMHelpers.ps1`** from this repo into the working folder and set `$domain` at the top of the joiner.

### 5.1 How it works

| Part | What it does | Key rule |
|---|---|---|
| **Guards** | Stops if not connected to Graph, or if the HR file has duplicate EmployeeIds | A duplicate ID means bad data, so nothing runs |
| **Pass 1: Accounts** | Skips anyone whose EmployeeId already exists → skips unknown departments → builds a username → creates the user with a temp password (must change at first sign-in) | **Fail closed:** no role-map entry means no account |
| **Pass 2: Managers** | Builds an EmployeeId → object Id lookup **once**, then links each user to their manager | Managers can only be linked after everyone exists, which is why this is a second pass. A missing manager is a **soft error**: the account is still created and HR is flagged. |
| **Pass 3: Groups** | Looks up group Ids and current members **once**, then adds missing memberships from the role map | **Disabled accounts are skipped**, so offboarded users never regain access |

**Username rules:** `first.last@domain`, lowercase, letters only (`I'teeya` → `iteeya.harris`). If the name is taken in Entra *or* earlier in the same run, a number is added (`michael.johnson2`).

### 5.2 Dry run (default, changes nothing)
```powershell
cd "C:\IAM Labs\Project1"
.\Invoke-Joiner.ps1
```
Maximize the window so the Status column isn't cut off. Expect **WOULD CREATE**, **WOULD SET**, and **WOULD ADD**.

<img width="1110" height="782" alt="Screenshot 2026-11-05 204438" src="https://github.com/user-attachments/assets/3eed37ad-4003-44b4-98a9-211fddbff044" />

### 5.3 Run for real
```powershell
.\Invoke-Joiner.ps1 -Commit
```

<img width="1072" height="572" alt="Screenshot 2026-11-05 204547" src="https://github.com/user-attachments/assets/bb3a3ef4-e1f2-4d82-94e3-3bbf84fb0e9c" />

<img width="1072" height="538" alt="manager_set_copy" src="https://github.com/user-attachments/assets/cd51247f-703c-4a76-9c28-d0f90bc7337b" />

<img width="874" height="702" alt="groups_added" src="https://github.com/user-attachments/assets/f57dabc2-3960-411f-8834-6afe67862dd2" />

### 5.4 Rerun to prove it's safe
```powershell
.\Invoke-Joiner.ps1
```
Expect **EXISTS - skipped**, **Already set**, and **Already member**. Nothing changes.

<img width="898" height="537" alt="already_exists" src="https://github.com/user-attachments/assets/6eb8b375-ba0c-47d9-bc43-7c115eb4be32" />

### 5.5 Verify
```powershell
# Users
Get-MgUser -All -Property DisplayName,UserPrincipalName,Department,EmployeeId |
    Where-Object EmployeeId | Sort-Object EmployeeId |
    Format-Table EmployeeId, DisplayName, UserPrincipalName, Department -AutoSize

# Managers
Get-MgUser -All -Property DisplayName,EmployeeId -ExpandProperty Manager | Where-Object EmployeeId |
    Select-Object DisplayName, @{n='Manager';e={$_.Manager.AdditionalProperties.displayName}}

# Group member counts
Get-MgGroup -Filter "startswith(displayName,'SG-')" | ForEach-Object {
    [PSCustomObject]@{ Group = $_.DisplayName; Members = (Get-MgGroupMember -GroupId $_.Id -All).Count }
}

# Licenses in use (wait ~5 minutes after adding members)
Get-MgSubscribedSku | Where-Object SkuPartNumber -eq "AAD_PREMIUM_P2" | Select-Object SkuPartNumber, ConsumedUnits
```

<img width="890" height="485" alt="entra_verfied" src="https://github.com/user-attachments/assets/f632cb68-ccf8-4d1d-99a4-7f01cc02bada" />

<img width="666" height="566" alt="manager_showing_foremplyees" src="https://github.com/user-attachments/assets/273d82f6-2194-44ea-af05-2b89c2827f38" />

<img width="892" height="318" alt="group_counted" src="https://github.com/user-attachments/assets/072634ae-d9ea-4ad1-a336-467a14ca5f1a" />

<img width="912" height="506" alt="user_licenses_showing_portal" src="https://github.com/user-attachments/assets/b7eeb3b8-3722-4db4-ac3b-ac5cddb71dbc" />

---

## Step 6: The Leaver Script

Copy **`Invoke-Leaver.ps1`** into the working folder.

### 6.1 How it works
| Order | Action | Why |
|---|---|---|
| 1 | **Disable** the account | Blocks new sign-ins |
| 2 | **Revoke sign-in sessions** | Cancels refresh tokens so devices that are already signed in get kicked out |
| 3 | **Remove all groups** | Removes access and frees the license |

Accounts are **disabled, not deleted**, so mail and files can be handed over, audit records are kept, and a mistake can be undone.

### 6.2 Create `hr_leavers.csv`
```csv
EmployeeId,LastDay,Reason
1514,2026-11-05,Resigned
1510,2026-11-05,Terminated
9999,2026-11-05,Test - does not exist
```
Row `9999` is a deliberate test: the script must report **NOT FOUND** and keep going.

### 6.3 Dry run → commit → rerun
```powershell
.\Invoke-Leaver.ps1            # WOULD DISABLE / WOULD REVOKE / WOULD REMOVE 2
.\Invoke-Leaver.ps1 -Commit    # DISABLED / REVOKED / 2
.\Invoke-Leaver.ps1            # Already disabled / 0 groups to remove
```

<img width="882" height="632" alt="leaver_dry_run" src="https://github.com/user-attachments/assets/ff62ae72-4bb2-42e7-9de4-07359d53db5e" />

<img width="890" height="366" alt="leaver_revoked_sessions" src="https://github.com/user-attachments/assets/7fc3db6e-6f05-4163-b8b8-3428dc6051d8" />

<img width="845" height="242" alt="leavers_disabled_revoked_agaion" src="https://github.com/user-attachments/assets/ef916db3-b728-4edf-8fac-8834fbcb2020" />

### 6.4 Prove it
```powershell
Get-MgSubscribedSku | Where-Object SkuPartNumber -eq "AAD_PREMIUM_P2" | Select-Object SkuPartNumber, ConsumedUnits
```
ConsumedUnits drops by one per leaver.

Then try signing in as an offboarded user at **myapps.microsoft.com** in a private window. Entra responds with *"Your account has been locked"* (AADSTS50057: account disabled).

<img width="866" height="520" alt="licenses_removed_leaver_script" src="https://github.com/user-attachments/assets/384f8590-65b3-455f-ab4d-29281bf262f3" />

<img width="956" height="530" alt="ga_signin_blocked" src="https://github.com/user-attachments/assets/2926ae11-507d-43e9-aa83-1005d6460bf6" />

### 6.5 The flaw this exposed (and the fix)
After offboarding, rerunning the joiner would have **added leavers back to their groups**, giving a disabled account its access and a license again. The joiner now records disabled accounts in Pass 2 and skips them in Pass 3 (`SKIP - account disabled`). This is already included in `Invoke-Joiner.ps1`.

<img width="778" height="725" alt="account_disabled_joinerscriptjpg" src="https://github.com/user-attachments/assets/cea1b729-5458-47dd-bb50-32eb593c1426" />

---

## Step 7: Error Handling and Audit Logging

### 7.1 How it works
- Every change sits in its **own `try/catch`**. If one step fails, the next steps still run. For a leaver, a failed disable still revokes sessions and removes access.
- `-ErrorAction Stop` turns soft errors into catchable ones. Without it, `catch` never fires.
- `Write-IamLog` (in `IAMHelpers.ps1`) appends one row per action to `logs\iam_actions.csv`:

| Column | Example |
|---|---|
| Timestamp | 2026-11-07 12:17:19 |
| RunBy | `admin@…` or `App: IAM-Provisioning-Automation` |
| Script | Joiner / Leaver |
| EmployeeId | 1518 |
| Action | Create user, Set manager, Add to SG-Dept-Finance, Disable account… |
| Target | User or UPN |
| Result | SUCCESS / ERROR / SKIP / WARN |
| Message | Error details or reason |

Changes, problems, and security decisions are logged. "Already done" results on reruns are **not** logged, so real events aren't buried in noise.

### 7.2 Offboard one more user and read the log
Add `1513,2026-11-06,Resigned` to `hr_leavers.csv`, then:
```powershell
.\Invoke-Leaver.ps1 -Commit
Import-Csv "C:\IAM Labs\Project1\logs\iam_actions.csv" | Format-Table -AutoSize
```

<img width="729" height="368" alt="leavers_log_ran" src="https://github.com/user-attachments/assets/ca114305-b066-43b3-a6cd-4afb82a42c30" />

<img width="882" height="516" alt="audi_log" src="https://github.com/user-attachments/assets/8f5aee83-0770-4c4a-941b-d1e7454a2b8c" />

### 7.3 Break it on purpose
Add a hire with an invalid country code:
```csv
1516,Test,Failure,IT,Test Account,1500,ZZ
```
```powershell
.\Invoke-Joiner.ps1 -Commit
Import-Csv "C:\IAM Labs\Project1\logs\iam_actions.csv" | Where-Object Result -eq "ERROR" | Format-List
```
Expected: Test Failure shows **ERROR**, everyone else is unaffected, and the error message is in the log. Change `ZZ` to `US` and rerun: only Test Failure gets created.

Clean up afterward:
```powershell
Remove-MgUser -UserId (Get-MgUser -Filter "employeeId eq '1516'").Id
```
Then delete the 1516 row from the CSV.

<img width="868" height="490" alt="joiner_error_caught" src="https://github.com/user-attachments/assets/8932d6fa-898f-48e0-b827-d594a0d293c5" />

<img width="666" height="344" alt="joiner_error_logged" src="https://github.com/user-attachments/assets/edc60c7e-e8ab-460b-b56c-6c4849e4aad5" />

<img width="882" height="502" alt="joiner_rerun__fixed_logged_corrected" src="https://github.com/user-attachments/assets/8c162093-35fa-4a08-bffb-da656afd6780" />

---

## Step 8: Unattended App-Only Automation

**Why:** delegated sign-in needs a person with a password and MFA. A scheduled job needs its **own identity**.

| | Delegated (Steps 1–7) | Application (Step 8) |
|---|---|---|
| Acts as | Signed-in admin | The app (service principal) |
| Limited by | The user's rights **and** the scopes | Only the granted permissions, tenant-wide |
| Credential | Password + MFA | Certificate |
| Approval | User or admin at sign-in | **Admin consent required** |

A **certificate** is used instead of a client secret. A secret is a password stored in a file. A certificate's private key stays locked in the Windows certificate store, and only the public half is uploaded.

### 8.1 Create the certificate
```powershell
$cert = New-SelfSignedCertificate `
    -Subject "CN=IAM-Provisioning-Automation" `
    -CertStoreLocation "Cert:\CurrentUser\My" `
    -KeyExportPolicy NonExportable `
    -KeySpec Signature `
    -KeyAlgorithm RSA -KeyLength 2048 -HashAlgorithm SHA256 `
    -NotAfter (Get-Date).AddMonths(12)

$cert | Select-Object Subject, Thumbprint, NotAfter
Export-Certificate -Cert $cert -FilePath "C:\IAM Labs\Project1\IAM-Provisioning-Automation.cer"
```
- **NonExportable:** the private key can't be copied off this machine.
- **12 months:** forces regular rotation.
- Check it in `certmgr.msc` → Personal → Certificates. It should say *"You have a private key that corresponds to this certificate."*

<img width="653" height="316" alt="cert_created" src="https://github.com/user-attachments/assets/7e88a84b-e1e0-49cf-ba26-dc049fe1cea2" />

<img width="408" height="522" alt="cert-private-key" src="https://github.com/user-attachments/assets/079133ae-bd10-479a-82d4-a31b219425ac" />
### 8.2 Register the app
**entra.microsoft.com → Entra ID → App registrations → + New registration**
- Name: `IAM-Provisioning-Automation`
- Supported account types: **Single tenant**
- Redirect URI: blank → **Register**
- Note the **Application (client) ID**

<img width="1137" height="680" alt="app_reg_overview" src="https://github.com/user-attachments/assets/4656022e-709b-496e-af88-3fb74cf7a165" />

### 8.3 Upload the public certificate
**Certificates & secrets → Certificates → Upload certificate** → choose the `.cer` → description `Provisioning automation - expires <month year>` → **Add**. Confirm the thumbprint matches the one on your PC.

<img width="1096" height="647" alt="cert_being_added" src="https://github.com/user-attachments/assets/7012c851-73f0-466c-b5f5-a5b107d9180e" />

### 8.4 Grant application permissions
**API permissions:**
1. Remove the default delegated **User.Read**, since the app signs no one in.
2. **+ Add a permission → Microsoft Graph → Application permissions**:
   - `User.ReadWrite.All` (create, disable, set managers, revoke sessions)
   - `Group.ReadWrite.All` (group membership)
   - `Organization.Read.All` (read licenses)
3. **Grant admin consent for <tenant>** → all three show ✅ **Granted**.

<img width="887" height="386" alt="granted_apipermission" src="https://github.com/user-attachments/assets/74dd1cc3-a695-40ac-8606-0123dac6f873" />

<img width="1145" height="478" alt="service_principle" src="https://github.com/user-attachments/assets/89b989a2-c701-41d2-9819-601518af518f" />

### 8.5 Create `app_config.json`
Copy `app_config.example.json` to `app_config.json`, then fill it in. The thumbprint is read **straight from the certificate store** to avoid copy/paste errors:
```powershell
cd "C:\IAM Labs\Project1"
Copy-Item .\app_config.example.json .\app_config.json
$cfg = Get-Content .\app_config.json -Raw | ConvertFrom-Json
$cfg.TenantId   = "<your-domain>.onmicrosoft.com"
$cfg.ClientId   = "<paste Application (client) ID>"
$cfg.Thumbprint = (Get-ChildItem Cert:\CurrentUser\My |
    Where-Object Subject -eq "CN=IAM-Provisioning-Automation").Thumbprint
$cfg | ConvertTo-Json | Set-Content .\app_config.json

$cfg.Thumbprint.Length   # must be 40
$cfg.ClientId.Length     # must be 36
```

### 8.6 Connect as the app
```powershell
Disconnect-MgGraph -ErrorAction SilentlyContinue
. .\IAMHelpers.ps1
Connect-IamGraph
Get-MgContext            # AuthType: AppOnly, AppName: IAM-Provisioning-Automation
```
There's no pop-up, no password, and no MFA. The scripts call `Connect-IamGraph` automatically when nobody is signed in, and **stop** if the connection fails.

<img width="807" height="521" alt="mgcontext_app_only" src="https://github.com/user-attachments/assets/acb6a53e-ef99-4fda-b239-94998b17db74" />

### 8.7 Test a provisioning run as the app
Add `1517,Kendra,Wallace,Sales,Sales Associate,1512,US` to `hr_new_hires.csv`, then:
```powershell
Disconnect-MgGraph -ErrorAction SilentlyContinue
.\Invoke-Joiner.ps1 -Commit
Import-Csv .\logs\iam_actions.csv | Where-Object EmployeeId -eq "1517" |
    Format-Table Timestamp, RunBy, Action, Result -AutoSize
```
RunBy shows **App: IAM-Provisioning-Automation**.

<img width="734" height="570" alt="joiner_run_as_app" src="https://github.com/user-attachments/assets/94042cba-c4e1-40ee-a9ae-b17fb707049f" />

<img width="846" height="272" alt="audit_log_showing_created" src="https://github.com/user-attachments/assets/5c2e062d-8381-4028-adf3-3a28137eb71f" />

### 8.8 Schedule it (daily, 6:00 AM)
The certificate is in the **CurrentUser** store, so the task runs as your Windows account. Look up where pwsh actually lives, since the Microsoft Store build is not in `Program Files`:
```powershell
$pwsh = (Get-Command pwsh).Source

$action = New-ScheduledTaskAction -Execute $pwsh `
    -Argument '-NoProfile -File "C:\IAM Labs\Project1\Invoke-Joiner.ps1" -Commit' `
    -WorkingDirectory "C:\IAM Labs\Project1"
$trigger  = New-ScheduledTaskTrigger -Daily -At 6:00AM
$settings = New-ScheduledTaskSettingsSet -StartWhenAvailable `
    -ExecutionTimeLimit (New-TimeSpan -Minutes 30)

Register-ScheduledTask -TaskName "IAM - Daily Joiner" `
    -Action $action -Trigger $trigger -Settings $settings `
    -Description "Provisions new hires from HR CSV using app-only Graph auth"
```
Test it now: add `1518,Andre,Lewis,Finance,Staff Accountant,1504,US` to the HR file, then:
```powershell
Start-ScheduledTask -TaskName "IAM - Daily Joiner"
# wait ~45 seconds
Get-ScheduledTaskInfo -TaskName "IAM - Daily Joiner" | Select-Object LastRunTime, LastTaskResult   # 0 = success
Import-Csv .\logs\iam_actions.csv | Where-Object EmployeeId -eq "1518" |
    Format-Table Timestamp, RunBy, Action, Result -AutoSize
```
Only the joiner is scheduled. Leavers are higher risk and usually run when HR sends a termination, often with a review step.

<img width="1330" height="502" alt="IAM_TaskScheduler" src="https://github.com/user-attachments/assets/940259da-4574-4269-85e5-c32ad3a47b8a" />

<img width="1162" height="298" alt="IAM_Task_ran_andre" src="https://github.com/user-attachments/assets/d690e522-fc95-4d64-b9ad-7c28a4a17297" />

---

## Resetting the Lab

```powershell
# Delete all lab users (only users with an EmployeeId; the admin has none)
$labUsers = Get-MgUser -All -Property Id,DisplayName,EmployeeId | Where-Object EmployeeId
$labUsers.Count                                   # check before deleting
$labUsers | ForEach-Object { Remove-MgUser -UserId $_.Id }

# Permanently purge them from the recycle bin (otherwise kept 30 days)
Get-MgDirectoryDeletedItemAsUser -All | ForEach-Object { Remove-MgDirectoryDeletedItem -DirectoryObjectId $_.Id }

# Pause the scheduled task
Disable-ScheduledTask -TaskName "IAM - Daily Joiner"
```
Wait 1–2 minutes before rerunning the joiner so Entra catches up.

---

## Troubleshooting

| # | Symptom | Root cause | Fix |
|---|---|---|---|
| 1 | `Install-Module` says "Administrator rights are required" even with `-Scope CurrentUser` | Controlled Folder Access blocked pwsh.exe from writing to Documents | Allowed pwsh.exe in Controlled Folder Access; used `Install-PSResource` |
| 2 | Sign-in loops / "InteractiveBrowserCredential authentication failed" | Windows broker (WAM) hand-off failing | `Set-MgGraphOption -DisableLoginByWAM $true` and `-ContextScope Process` |
| 3 | `Get-MgContext` returns nothing | Connection never finished in that window | Reconnect in the same window; wait for "Welcome to Microsoft Graph!" |
| 4 | Dry run planned `roy.james2` for the same person | Duplicated HR row (same EmployeeId) | Fixed data; added the duplicate-EmployeeId guard |
| 5 | A whole department showed "-" (skipped) | Department missing or misspelled in role_map.json | Fixed the map; fail-closed behavior worked as designed |
| 6 | Rerunning the joiner would re-add leavers to groups | Pass 3 didn't check account status | Skip disabled accounts in Pass 3 |
| 7 | Dozens of "Authentication needed" errors | Connection failed; `-ErrorAction SilentlyContinue` hid it in earlier passes | Added the "not connected" guard |
| 8 | "CertificateThumbprint must have a length of 40" | Config still held placeholder text (length 21) | Read the thumbprint from the cert store; check lengths 40/36 |
| 9 | Scheduled task failed, no visible error | Task pointed at `C:\Program Files\PowerShell\7\pwsh.exe`, but pwsh was the Store build | Used `(Get-Command pwsh).Source`; result 0x0 |

<img width="666" height="338" alt="gurad_stop" src="https://github.com/user-attachments/assets/849c2fb1-14cd-47ed-b862-4656849da307" />
---

## Key Concepts

| Term | Plain meaning |
|---|---|
| **JML** | Joiner / Mover / Leaver: the identity lifecycle |
| **Source of truth** | HR's data decides who gets an account; IT automates from it |
| **RBAC** | Access comes from your role (department), not individual grants |
| **Group-based licensing** | Licenses attach to groups, so membership decides who's licensed |
| **Idempotent** | Running it once or ten times gives the same result |
| **Fail closed** | When unsure, give no access rather than guess |
| **Hard stop vs. soft error** | Block when access is at risk; flag and continue when it isn't |
| **Revoke sessions** | Cancels refresh tokens so remembered sign-ins stop working |
| **Delegated vs. application permissions** | Acting as a signed-in user vs. acting as the app itself |
| **Service principal** | The app's actual account inside the tenant |
| **Admin consent** | An admin approving an app's tenant-wide permissions |

---

## Security Considerations

- **Least privilege:** three application permissions only; default `User.Read` removed; the app never deletes users.
- **Certificate, not secret:** non-exportable private key, public key only in Entra, 12-month expiry.
- **Nothing sensitive in Git:** `app_config.json`, temp passwords, live logs, and certificates are git-ignored.
- **Fail-closed guards:** unknown departments, duplicate IDs, and lost connections stop processing.
- **Accountability:** every change records whether a person or the app made it.
- **Known risk:** an app with `User.ReadWrite.All` can change any non-privileged user tenant-wide. In production it would need a dedicated automation host, certificate expiry monitoring, and sign-in alerts on the service principal.

---

## What I'd Do Differently

| Improvement | Why |
|---|---|
| **Temporary Access Pass** instead of temp passwords | No password ever written to a file |
| **Azure Automation + managed identity** | No certificate to manage; runs in the cloud, not on a workstation |
| **Mover workflow** | Department changes should swap groups (remove old, add new) |
| **Scramble password on leave** | Extra lock behind the disable |
| **Start-Transcript for scheduled runs** | Capture full console output when no one is watching |
| **Pester tests + PSScriptAnalyzer** | Automated testing and linting |
| **Logs to Log Analytics / Sentinel** | Central, tamper-resistant audit trail with alerts |
| **Certificate expiry alerts** | Warn before the 12-month cert breaks the automation |
| **HRIS API instead of CSV** | Real-time HR events instead of file drops |

---

## Skills Demonstrated

- Identity lifecycle automation (Joiner / Leaver) from an authoritative HR source
- RBAC design with a role map, plus group-based licensing
- Microsoft Graph PowerShell SDK: users, groups, managers, licenses, sessions
- Entra ID: security groups, app registrations, service principals, admin consent
- Authentication: delegated vs. application permissions, certificate-based app-only auth, session revocation
- Safe automation: dry runs, idempotency, fail-closed validation, guards, per-step error handling
- Audit logging supporting NIST 800-53 AC-2 (Account Management)
- Troubleshooting endpoint security, sign-in, data-quality, and scheduling issues

---

**Author:** Javier "Jay" Johnson · [LinkedIn](https://linkedin.com/in/javier-johnson-it)

