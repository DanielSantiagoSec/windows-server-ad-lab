# Active Directory Configuration

This document covers Active Directory configuration tasks performed in the lab environment, including automated backup scheduling, event-triggered task creation, OU design, and user provisioning.

---

## Task 1: Automated Backup with Task Scheduler

**Objective:** Configure a scheduled task to run an automated backup of a critical directory on a recurring schedule.

**Configuration:**

- **Tool:** Windows Task Scheduler
- **Trigger:** Daily at 02:00 AM
- **Action:** Run `robocopy` to copy the target directory to a backup destination
- **Run As:** Local System (or a dedicated service account with minimum necessary permissions)
- **Settings:** Run task as soon as possible after a missed schedule; do not run concurrent instances

**Command used:**
```
robocopy "C:\CriticalData" "D:\Backups\CriticalData" /MIR /LOG:"C:\Logs\backup.log"
```

**Result:** Task created and verified. Triggered manually to confirm execution. Log file confirmed successful copy of all target files to backup destination.

**Security Note:** The backup destination (`D:\Backups`) was configured with restricted NTFS permissions — read/write for the backup service account only. This prevents standard users from accessing or tampering with backup data.

---

## Task 2: Event-Triggered Task for Logon Monitoring (Event ID 4624)

**Objective:** Configure a Task Scheduler task that triggers on Windows Security Event ID 4624 (successful logon) to demonstrate event-driven automation.

**Configuration:**

- **Tool:** Windows Task Scheduler → New Trigger → On an Event
- **Event Log:** Security
- **Source:** Microsoft-Windows-Security-Auditing
- **Event ID:** 4624
- **Action:** Write entry to a custom log file (`C:\Logs\logon-events.log`) with timestamp

**Steps:**
1. Opened Task Scheduler and created a new task
2. Set trigger to "On an event" — Log: Security, Source: Microsoft-Windows-Security-Auditing, Event ID: 4624
3. Set action to run a PowerShell script that appends a timestamped entry to the log file
4. Configured the task to run whether or not the user is logged in

**Result:** Task triggered successfully on next logon event. Logon entries with timestamps confirmed in the log file.

**Why This Matters:** Demonstrates the ability to operationalize Windows audit events beyond passive log review. Event-triggered automation enables real-time response workflows — the same mechanism used in enterprise environments to trigger alerts or automated containment actions on security-relevant events.

---

## Task 3: Organizational Unit (OU) Creation

**Objective:** Design and create an OU structure in Active Directory to support departmental user management and scoped Group Policy application.

**OU Structure Created:**

```
domain.local
└── Company
    ├── IT
    ├── Finance
    ├── HR
    ├── Sales
    └── ServiceAccounts
```

**Steps:**
1. Opened Active Directory Users and Computers (ADUC)
2. Right-clicked the domain root → New → Organizational Unit
3. Created the `Company` parent OU
4. Created child OUs for each department under `Company`
5. Created a separate `ServiceAccounts` OU to isolate non-human accounts from user account GPO scope

**Design Rationale:**
- Department-level OUs allow GPOs to be applied at the department level without affecting other departments
- `ServiceAccounts` isolation prevents user-targeted GPOs (password complexity, logon restrictions) from inadvertently applying to service accounts, which have different operational requirements

---

## Task 4: User Provisioning into New OU

**Objective:** Provision a new user account and place it in the correct OU.

**Steps:**
1. In ADUC, navigated to the target OU (e.g., `Company > HR`)
2. Right-clicked → New → User
3. Completed user creation form: first name, last name, UPN (user@domain.local), initial password
4. Set "User must change password at next logon" — enforces credential ownership from day one
5. Confirmed account appeared in the correct OU

**Group Membership:**
- Added the new user to the `HR-Users` security group
- Group membership drives access to HR file shares and HR-specific application permissions

**Connection to M365 IAM Work:** This manual provisioning process is the direct predecessor of the automated joiner workflow built in the [`m365-iam-lifecycle-automation`](https://github.com/DanielSantiagoSec/m365-iam-lifecycle-automation) repo. The `New-JoinerAccount.ps1` script automates every step performed manually here — OU placement, group assignment, UPN construction, and initial password handling — using Microsoft Graph PowerShell against an Entra ID tenant.
