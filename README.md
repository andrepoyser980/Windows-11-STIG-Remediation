**WN11-AU-000560**

<img width="500" height="300" alt="image" src="https://github.com/user-attachments/assets/91af652b-6d05-4ad7-8d45-a1f4b4697ef6" />



**“Other Logon/Logoff Events” Must Be Audited (Success)**

**Author:** Andre Poyser

**Version:** 1.0

**Last Updated:** 2025-11-29

**STIG Overview**

**Control ID:** WN11-AU-000560
**Requirement:**

“Windows 11 systems must audit **Other Logon/Logoff Events** with **Success** enabled.”

This is part of the Advanced Audit Policy under:

```sql
Audit Policy → Advanced Audit Policy Configuration
 ↳ System Audit Policies
    ↳ Logon/Logoff
       ↳ Other Logon/Logoff Events
```

This category logs events related to:

* Remote connections

* Scheduled task logon events

* Token creation

* Interactive/remote session details

These logs support:

* Threat hunting

* Incident response

* Credential misuse detection

* Compliance reporting

* Forensics and chain-of-custody documentation

**Why Passing This STIG Can Be Difficult**

This STIG is notorious for failing even when the setting appears correct.

Common issues:

**1. Legacy Audit Policy overrides advanced audit settings**

If any of these are enabled under:

```sql
secpol.msc → Local Policies → Audit Policy
```

(e.g., Audit logon events)
Windows will block all subcategory settings and show:

```sql
No auditing

```
**2. SCENoApplyLegacyAuditPolicy not enabled**

This registry value controls audit policy mode:

```sql
HKLM\SYSTEM\CurrentControlSet\Control\Lsa\SCENoApplyLegacyAuditPolicy

```
0 or missing → legacy mode (subcategory settings ignored)

1 → advanced audit override enabled

**3. Nessus validates only Success, not Success+Failure**

Nessus mirrors DISA check logic:
```sql
auditpol /get /subcategory:"Other Logon/Logoff Events"

```
It checks strictly for Success.

**Troubleshooting Process**

Below is the actual sequence I executed. 


**Step 1** Verified Legacy Audit Policy

**I checked:**
```sql
secpol.msc → Local Policies → Audit Policy
```

I identified that one or more legacy policies were enabled—this blocks advanced auditing globally.

Fixed by setting **ALL** entries to **No auditing.**

<img width="1177" height="528" alt="image" src="https://github.com/user-attachments/assets/719efb42-3959-47db-be3d-4c02bb31c3ee" />


**Step 2** Enabled Advanced Audit Override

**Checked registry:**
```sql
reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v SCENoApplyLegacyAuditPolicy

```
If missing or 0, audit subcategories are ignored.

Set to 1 (REG_DWORD)
Rebooted to activate

<img width="1333" height="646" alt="image" src="https://github.com/user-attachments/assets/f6d90465-0ce9-4233-a455-1f3a8d51bf93" />


**Step 3** Applied Audit Policy Using auditpol.exe

Tested initial setting:
```sql
auditpol /set /subcategory:"Other Logon/Logoff Events" /success:enable /failure:enable
```

This is the correct STIG-aligned config (success + failure)

**Step 4** Observed Failure in Nessus

Nessus still failed the STIG even though auditpol showed:
```sql
Success and Failure
```

This proved that Nessus was checking only Success state.

**Step 5** Identified the Fix for Nessus

Running this command manually resolved the finding:
```sql
auditpol /set /subcategory:"Other Logon/Logoff Events" /success:enable
```
<img width="1634" height="560" alt="image" src="https://github.com/user-attachments/assets/9af5481d-3b02-4082-a4db-f3a1264e0315" />

Nessus passed immediately.
This confirmed that **some environments require enabling Success separately** even when Success+Failure is already on.

<img width="1645" height="944" alt="image" src="https://github.com/user-attachments/assets/90feb4c3-c0c3-487e-b584-30ff6a0a17e7" />


**Step 6** Integrated Backup Logic into Script

To ensure automation always passes:

**1.** Apply Success and Failure (primary path)

**2.** Verify Success

**3.** If not present → apply Success only (backup path)

**4.** Re-verify

**5.** Exit with STIG compliance status

This hybrid approach ensures compatibility with:

* Windows 11

* STIG Viewer

* Nessus STIG audit checks

* DoD ACAS audit logic

**Final Remediation Script (with Backup–Path Logic)**

[WN11-AU-000560.ps1](https://github.com/andrepoyser980/andrepoyser980/blob/main/STIGS/WN11-AU-000560.ps1)

**Verification Commands**
**Check Effective Setting**

```sql
auditpol /get /subcategory:"Other Logon/Logoff Events"
```

Expected:

```sql
Inclusion Setting : Success      OR
Inclusion Setting : Success and Failure
```

**Check Registry Override**
```sql
reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v SCENoApplyLegacyAuditPolicy
```

Expected:
```sql
0x1
```

<img width="1511" height="419" alt="image" src="https://github.com/user-attachments/assets/965873e4-9eda-48e9-8b52-0e2b5f796843" />

**Check for Legacy Audit Conflict**
```sql
secpol.msc → Local Policies → Audit Policy
```

Expected:
**All entries = No auditing**

**Nessus STIG Audit Validation**

After running the script and verifying:

* Success enabled
* SCENoApplyLegacyAuditPolicy = 1
* Legacy audit disabled

Nessus passed the STIG check:
```sql
WN11-AU-000560 : PASS
```
