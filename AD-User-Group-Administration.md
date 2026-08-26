<h1 align="center">🧑‍💼 Active Directory Administration</h1>
<h3 align="center">Users, Organizational Units, Groups & Account Lifecycle Management</h3>

<p align="center">
  <img src="https://img.shields.io/badge/Focus-AD%20User%20%26%20Group%20Management-1E293B?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Platform-Windows%20Server%20AD%20DS-334155?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Status-Hands--On%20Practice-475569?style=for-the-badge"/>
</p>

---

## 📋 Purpose

This document covers the core identity-management skills practiced on the Domain Controller in my home SOC lab: creating and organizing users, structuring Organizational Units (OUs), managing group membership, and handling account lifecycle actions like disabling and re-enabling accounts. This is foundational AD administration — the kind of work that underpins almost every identity-related security event a SOC investigates (account lockouts, privilege changes, suspicious group membership).

---

## 👤 Users

**What a user object represents:** an individual identity in the domain, used for authentication, resource access, and permission assignment.

### Creating a user
Via **Active Directory Users and Computers (ADUC)**:
1. Right-click the target OU → **New → User**
2. Fill in First name, Last name, and a logon name (`sAMAccountName`) — this becomes the user's domain login
3. Set an initial password, and choose whether the user must change it at next logon (recommended for real onboarding scenarios)

Via **PowerShell** (the more scalable, scriptable approach):
```powershell
New-ADUser -Name "Jane Doe" `
  -GivenName "Jane" -Surname "Doe" `
  -SamAccountName "jdoe" `
  -UserPrincipalName "jdoe@soc.local" `
  -Path "OU=Employees,DC=soc,DC=local" `
  -AccountPassword (ConvertTo-SecureString "TempPass123!" -AsPlainText -Force) `
  -Enabled $true `
  -ChangePasswordAtLogon $true
```

### Key attributes worth knowing
| Attribute | Purpose |
|---|---|
| `sAMAccountName` | Legacy logon name, still used domain-wide (e.g. `SOC\jdoe`) |
| `userPrincipalName` | Modern logon format (`jdoe@soc.local`) |
| `distinguishedName` | Full LDAP path identifying the object's exact location in the directory |
| `memberOf` | Groups the user belongs to |

---

## 🗂️ Organizational Units (OUs)

**What an OU is:** a container used to organize objects (users, computers, groups) within a domain — primarily for applying **Group Policy** and delegating administrative control, not for security permissions directly (that's what groups are for).

### Why OU structure matters
A well-planned OU hierarchy lets you:
- Apply different Group Policy Objects (GPOs) to different parts of the org (e.g. stricter password policy for an IT OU vs. general staff)
- Delegate limited admin rights (e.g. a helpdesk group that can reset passwords only within a specific OU)
- Keep the directory navigable as it scales

### Example structure practiced in this lab
```
SOC (domain root)
├── Employees
│   ├── IT
│   └── Security
├── Servers
├── Workstations
└── Service Accounts
```

### Creating an OU
```powershell
New-ADOrganizationalUnit -Name "Security" -Path "OU=Employees,DC=soc,DC=local"
```

---

## 👥 Groups

**What a group is:** a container of members (users, computers, or other groups) used to assign permissions or apply policy collectively, rather than one object at a time.

### Group types
| Type | Purpose |
|---|---|
| **Security groups** | Used to assign permissions to resources (file shares, applications, etc.) |
| **Distribution groups** | Used for email distribution lists only — no security function |

### Group scopes
| Scope | Can contain | Typically used for |
|---|---|---|
| **Domain Local** | Members from any domain | Assigning access to resources in the local domain |
| **Global** | Members from the same domain only | Organizing users with similar job functions |
| **Universal** | Members from any domain in the forest | Cross-domain resource access in multi-domain environments |

### Creating a group and adding members
```powershell
New-ADGroup -Name "SOC-Analysts" -GroupScope Global -GroupCategory Security -Path "OU=Security,OU=Employees,DC=soc,DC=local"

Add-ADGroupMember -Identity "SOC-Analysts" -Members "jdoe"
```

### Checking group membership
```powershell
Get-ADGroupMember -Identity "SOC-Analysts"
```

---

## ⏸️ Account Lifecycle: Disabling, Suspending & Re-Enabling

**Why this matters for security, not just IT admin:** a disabled-but-not-deleted account is a standard practice for offboarding — it preserves the object (and its history/permissions for auditing) while immediately cutting off access. Suspended accounts unexpectedly *re-enabling*, or disabled accounts still showing successful logon attempts, are both meaningful signals a SOC would investigate.

### Disabling a user account
```powershell
Disable-ADAccount -Identity "jdoe"
```
Via ADUC: right-click the user → **Disable Account**

### Re-enabling
```powershell
Enable-ADAccount -Identity "jdoe"
```

### Locking vs. Disabling — an important distinction
| Action | Meaning | Who can undo it |
|---|---|---|
| **Locked out** | Automatic, triggered by repeated failed login attempts (a security control against brute-force) | Auto-unlocks after a timeout, or an admin can force it via `Unlock-ADAccount` |
| **Disabled** | Deliberate admin action — account cannot authenticate at all until re-enabled | Only an admin, explicitly |

```powershell
Unlock-ADAccount -Identity "jdoe"
```

### Checking account status
```powershell
Get-ADUser -Identity "jdoe" -Properties Enabled, LockedOut, PasswordExpired
```

---

## 🔍 Why This Matters for SOC Work

Every action documented above generates a corresponding **Windows Security event**, now flowing into this lab's SIEM via the DC's Advanced Audit Policy configuration:

| Action | Event ID |
|---|---|
| User account created | 4720 |
| User account enabled | 4722 |
| User account disabled | 4725 |
| Account locked out | 4740 |
| Account unlocked | 4767 |
| Group membership added | 4728 / 4732 / 4756 (depending on group type) |
| Password reset | 4724 |

Understanding what "normal" AD administration looks like — and what generates which event — is the foundation for later recognizing what *abnormal* looks like: unexpected account creation outside business hours, a disabled account suddenly re-enabled, or unusual group membership changes (e.g. a standard user added to Domain Admins).

---

## 📚 Resources

- [Microsoft — Active Directory Users and Computers Overview](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/adac/active-directory-administrative-center)
- [Microsoft — AD PowerShell Module Reference](https://learn.microsoft.com/en-us/powershell/module/activedirectory/)
- [Microsoft — Group Scope Explained](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups)
- [Ultimate Windows Security — 4720-4767 Event ID Reference](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/)

---

<p align="center"><i>Part of the SOC Home Lab documentation series — companion to the Active Directory & SIEM integration write-up.</i></p>
