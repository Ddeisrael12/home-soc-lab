# Ticket 02: Domain Admins group changed

| Field | Value |
| --- | --- |
| Alert | Wazuh rule 60159, "Domain Admins Group Changed" |
| Severity | Level 12 |
| Host | DC001.BuildBetter.digital |
| Time | 2026-10-28, 00:47 CDT |
| Source | Windows Security Event ID 4728 (member added to a security-enabled global group) |
| Verdict | **Authorized**, planned lab test; change reverted |

## Timeline

| Time (CDT) | Event ID | Wazuh rule | What happened |
| --- | --- | --- | --- |
| 00:43:31 | 4720 | 60109 (level 8) | User account `tadmin3` (Test Admin 3) created in OU LabUsers |
| 00:43:31 | (see alert) | 60110 (level 8) | User account changed (password set, enabled) |
| 00:47:07 | 4728 | 60148 (level 5) | Security-enabled global group changed |
| 00:47:07 | 4728 | **60159 (level 12)** | **Member added to Domain Admins** |
| ~01:05 | 4729 | | Member removed from Domain Admins (cleanup; confirm the event in Wazuh) |

## Evidence

| Field | Value | Meaning |
| --- | --- | --- |
| subjectUserName | Administrator | Who made the change |
| subjectUserSid | ends in **-500** | The built-in Administrator account |
| memberName | CN=Test Admin 3, OU=LabUsers, DC=BuildBetter, DC=digital | Who was added |
| targetUserName | Domain Admins | Which group |
| targetSid | ends in **-512** | The Domain Admins group in any domain |

## Analysis

- A brand-new account was given the highest privilege in the domain 4 minutes after it was created.
- The change was made with the built-in Administrator account (SID -500), not a named admin account, so the action can't be tied to a specific person.
- In production, this pattern (new account, then immediate Domain Admins membership) is a common sign of privilege escalation or attacker persistence.

## Verification

In a real environment: look for a matching change request, or confirm with the administrator directly. Here, the change was a planned lab test by the analyst.

## Action

- Removed the test account from Domain Admins and confirmed membership is back to the approved accounts:
  ```powershell
  Remove-ADGroupMember -Identity "Domain Admins" -Members tadmin,tadmin3 -Confirm:$false
  Get-ADGroupMember "Domain Admins" | Select-Object SamAccountName
  ```
- Recommendation: admins should use named privileged accounts instead of the built-in Administrator, so every action is traceable.

## Root cause found along the way

The first attempt produced no alert. Auditing for Account Management showed "No Auditing" on the domain controller, because the audit settings had been put in the Default Domain Policy instead of the Default Domain Controllers Policy. Fixed by setting Advanced Audit Policy in the correct GPO.
