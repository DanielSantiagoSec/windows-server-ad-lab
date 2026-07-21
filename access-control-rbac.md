# Access Control and RBAC

This document covers role-based access control implementation across Windows Server and Linux environments, including an account modification lab simulating a real-world role change, GPO password policy enforcement, and Linux server account creation.

---

## Lab Scenario: Pat's Role Change (Finance → HR)

**Background:**
Pat is an existing user who has transferred from the Finance department to HR. The access control task is to modify Pat's account to reflect the new role — removing Finance access and granting HR access — without creating a new account.

**Why This Matters:** In enterprise environments, creating a new account for a transfer rather than modifying the existing one creates orphaned accounts, duplicates audit history, and increases the risk of the old account remaining active. Proper role changes are performed on the existing account.

**Steps Performed:**

1. Located Pat's account in ADUC under `Company > Finance`
2. Updated display name and description to reflect HR role
3. Removed Pat from `Finance-Users` security group
4. Added Pat to `HR-Users` security group
5. Moved Pat's account object from `Company > Finance` OU to `Company > HR` OU
   - Right-clicked account → Move → selected HR OU
6. Updated Pat's manager attribute and department field in account properties
7. Confirmed Pat's access: Finance file share access removed (group-driven); HR file share access confirmed

**RBAC Principle Applied:** Access follows role, not the individual. By managing permissions through group membership rather than individual ACL entries, the role change is clean, auditable, and doesn't require touching every resource Pat had access to. The group membership change propagates automatically to all resources where that group has been granted access.

**Connection to M365 IAM Work:** This manual process is what the `Set-MoverAccount.ps1` script in the [`m365-iam-lifecycle-automation`](https://github.com/DanielSantiagoSec/m365-iam-lifecycle-automation) repo automates in Entra ID — department attribute updates, license reassignment, group changes, and manager updates, all triggered from a single script execution.

---

## GPO Password Policy Enforcement

**Objective:** Configure and enforce a password policy via Group Policy that applies to all domain users.

**Policy Settings Configured:**

| Setting | Value | Rationale |
|---------|-------|-----------|
| Minimum password length | 12 characters | Reduces brute-force feasibility |
| Password complexity | Enabled | Requires mixed character classes |
| Maximum password age | 90 days | Limits credential exposure window |
| Minimum password age | 1 day | Prevents immediate policy bypass |
| Password history | 10 passwords | Prevents rotation cycling |
| Account lockout threshold | 5 attempts | Mitigates online brute-force |
| Account lockout duration | 30 minutes | Balances security and helpdesk burden |

**Implementation:**

1. Opened Group Policy Management Console (GPMC)
2. Created a new GPO: `Domain Password Policy`
3. Navigated to Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy
4. Configured all settings per the table above
5. Linked the GPO to the domain root (applies to all users)
6. Ran `gpupdate /force` on a test machine and confirmed policy applied via `gpresult /r`

**Testing:** Created a test account and attempted to set a password that violated complexity requirements. Policy rejected the password. Confirmed lockout behavior by entering incorrect credentials five times.

---

## Linux Server Account Creation

**Objective:** Create a user account on a Linux server with appropriate group membership and access scope.

**Scenario:** A new member of the IT team needs access to the Linux server for log review. Access should be scoped to the minimum necessary — read access to log directories, no sudo rights.

**Commands Used:**

```bash
# Create the new user
sudo useradd -m -s /bin/bash ituser

# Set initial password
sudo passwd ituser

# Create a log-readers group if it doesn't exist
sudo groupadd log-readers

# Add user to log-readers group
sudo usermod -aG log-readers ituser

# Set group ownership and permissions on log directory
sudo chown root:log-readers /var/log/applogs
sudo chmod 750 /var/log/applogs
```

**Result:** `ituser` can read files in `/var/log/applogs` but cannot write to them or access other restricted directories. No sudo rights granted.

**Least Privilege Applied:** The account was created with only the access necessary for the stated job function. No shell history of elevated commands, no group memberships beyond what the role requires. This mirrors the principle applied in the Windows environment — access follows role, scoped to minimum necessary.
