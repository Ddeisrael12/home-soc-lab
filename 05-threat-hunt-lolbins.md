# Ticket 05: Threat hunt, built-in admin tools on WS001

| Field | Value |
| --- | --- |
| Type | Proactive threat hunt (no alert fired) |
| SIEM | Splunk Enterprise (Sysmon via Universal Forwarder) |
| Host | WS001 |
| Data searched | All Sysmon process-creation events (Event ID 1) since the forwarder was installed on 2026-10-30 |
| Hunt performed | 2026-10-30 23:48 to 2026-10-31 00:05 CDT |
| Hypothesis | Built-in Windows tools that attackers use to hide activity (`auditpol`, `wevtutil`, `secedit`) are being misused |
| ATT&CK | T1070.001 Clear Windows Event Logs; T1562.002 Impair Defenses: Disable Windows Event Logging |
| Verdict | **No malicious activity.** All runs explained (Wazuh SCA checks and a Microsoft Defender update) |

## Why hunt for this

Attackers often use tools already built into Windows ("living off the land", or LOLBins) because they don't look like malware:

- `wevtutil cl Security` clears the Security log.
- `auditpol /set ... /success:disable` or `auditpol /clear` turns auditing off.
- `secedit` can change security policy.

A baseline of the programs running on WS001 showed these tools near the top of the list: `auditpol.exe` was the **most frequently run program** on the machine.

## Step 1: Baseline, what runs on WS001

```
index=* host=WS001 source="*Sysmon*" EventCode=1
| stats count by Image
| sort - count
```

Top results included `auditpol.exe` (324), `svchost.exe` (317), `cmd.exe` (163), `wevtutil.exe` (158), `powershell.exe` (120) and `SecEdit.exe` (108).

## Step 2: Who started them (parent process)

```
index=* host=WS001 source="*Sysmon*" EventCode=1 (Image="*auditpol.exe" OR Image="*wevtutil.exe" OR Image="*SecEdit.exe")
| stats count by ParentImage, Image
```

| Parent | Program | Count | Verdict |
| --- | --- | --- | --- |
| `wazuh-agent.exe` | auditpol | 189 | Expected: Wazuh agent |
| `MsMpEng.exe` (Microsoft Defender) | wevtutil | 16 + 74 | Expected: Defender |
| `svchost.exe` | wevtutil | 8 | Expected: Windows scheduled task |
| `powershell.exe` (32-bit) | SecEdit | 175 | Checked in step 4 |
| **blank (`-`)** | auditpol | 324 | Checked in step 3 |
| **blank (`-`)** | wevtutil | 72 | Checked in step 3 |

A blank parent usually means the parent process exited before Sysmon recorded it. That is common, but not proof of anything, so the command lines were checked.

## Step 3: What the blank-parent runs did (command line)

```
index=* host=WS001 source="*Sysmon*" EventCode=1 (Image="*auditpol.exe" OR Image="*wevtutil.exe") ParentImage="-"
| eval User=mvindex(User,-1)
| stats count by Image, CommandLine, User
```

- **auditpol (324):** every run was `auditpol.exe /get /subcategory:"..."`, run by `NT AUTHORITY\SYSTEM`, 12 times per subcategory in alphabetical order. `/get` only **reads** a setting. The steady, alphabetical pattern matches Wazuh's Security Configuration Assessment (SCA).
- **wevtutil (72):** 12 different commands, 6 runs each. All were `install-manifest` / `uninstall-manifest` for files in `Windows Defender\Platform\4.18.26100.4-0\`. That is a **Defender platform update** re-registering its event logs. **No `cl` (clear log) command anywhere.**

## Step 4: Who started the 32-bit PowerShell

```
index=* host=WS001 source="*Sysmon*" EventCode=1 Image="*SysWOW64*powershell.exe"
| eval User=mvindex(User,-1)
| stats count by ParentImage, User
```

| Parent | User | Count |
| --- | --- | --- |
| `wazuh-agent.exe` | SYSTEM | 72 |
| blank (`-`) | SYSTEM | 125 |

Command lines for the 125 blank-parent runs:

```
index=* host=WS001 source="*Sysmon*" EventCode=1 Image="*SysWOW64*powershell.exe" ParentImage="-"
| stats count by CommandLine
```

| Command | What it does |
| --- | --- |
| `secedit /export ... Select-String "LockoutBadCount"` (and `LockoutDuration`, `ResetLockoutCount`, `MaximumPasswordAge`, `MinimumPasswordAge`, `MinimumPasswordLength`, `PasswordHistorySize`) | Exports security policy, reads **one** setting, deletes the temp file |
| `Get-CimInstance ... Win32_UserAccount ... SID LIKE '...-501'` | Checks whether the built-in **Guest** account (SID -501) is disabled |

These are CIS benchmark checks run by Wazuh SCA. Counts add up to 125. No encoded commands (`-enc`), no downloads, no changes.

## Conclusion

Every run of `auditpol`, `wevtutil` and `secedit` on WS001 was explained:

- Wazuh SCA reading audit and password policy settings (read-only, as SYSTEM)
- A Microsoft Defender platform update registering its event logs
- A Windows scheduled task

No evidence of log clearing or audit tampering. The hunt also produced a **baseline** of normal process activity on WS001, so future runs with different parents or command lines will stand out.

## Recommendations

- Don't allowlist these tools by program name alone. Allowlist the **parent + command line** pattern (for example, `wazuh-agent.exe` + `auditpol /get`), so a malicious `auditpol /clear` still alerts.
- Add a detection for the dangerous uses, whatever the parent:

```
index=* source="*Sysmon*" EventCode=1
  ((Image="*wevtutil.exe" (CommandLine="* cl *" OR CommandLine="*clear-log*"))
   OR (Image="*auditpol.exe" (CommandLine="*/clear*" OR CommandLine="*disable*" OR CommandLine="*/remove*")))
| table _time, host, User, ParentImage, CommandLine
```

- Pair it with Security Event ID **1102** (Security log cleared) from every host.

## Lessons learned

- **Start with a baseline.** You can't spot "unusual" until you know what "normal" looks like.
- **Program name is not enough; check the parent and the command line.** The same `wevtutil.exe` can be a Defender update or an attacker erasing evidence.
- **A blank parent is a question, not an answer.** It was resolved by reading what the process actually did.
- **Splunk `User` field duplicates:** each event carried two `User` values (`NOT_TRANSLATED` from the event header and the real account from Sysmon), so `stats` counted every event twice. `eval User=mvindex(User,-1)` keeps only the last value.
- **Most hunts find nothing malicious, and that's a valid result.** Proving activity is normal is useful work.
