# Ticket 04: Sensitive file modified in a monitored Finance folder

| Field | Value |
| --- | --- |
| Alert | Wazuh rule 550, "Integrity checksum changed" (plus 554 file added, 553 file deleted) |
| Host | WS001.BuildBetter.digital |
| Monitored path | `C:\Finance` (real-time File Integrity Monitoring with content tracking) |
| Time | 2026-10-30, 01:09:38 to 01:10:06 CDT |
| Source | Wazuh syscheck (File Integrity Monitoring) |
| Verdict | **Benign**, planned lab test |

## Setup

`C:\Finance` was added to the Wazuh agent's FIM configuration on WS001 to simulate a folder of sensitive financial records:

```xml
<syscheck>
  <directories realtime="yes" report_changes="yes">C:\Finance</directories>
</syscheck>
```

- `realtime="yes"`: alerts within seconds instead of waiting for the scheduled scan (every 12 hours by default)
- `report_changes="yes"`: records **what** changed inside text files, not just that the file changed

The agent log confirmed monitoring: `Monitoring path: 'c:\finance', with options '... report_changes | realtime'`.

## Query used

```
agent.name:WS001 and rule.id:(550 or 553 or 554)
```

## Timeline

| Time (CDT) | Rule | Event | File |
| --- | --- | --- | --- |
| 01:09:38 | 550 | Integrity checksum changed (file modified) | `c:\finance\payroll.txt` |
| 01:09:52 | 554 | File added to the system | `c:\finance\inv.txt` |
| 01:10:06 | 553 | File deleted | `c:\finance\inv.txt` |

## Evidence (rule 550)

| Field | Value |
| --- | --- |
| syscheck.path | `c:\finance\payroll.txt` |
| syscheck.diff | `> Bonus 50000` (one line added) |

The original file contained only `Q4 payroll: approved`. The diff shows exactly one new line was appended.

## Analysis

- A payroll record was edited to add a bonus amount, and a second file was briefly created and then deleted in the same folder within 30 seconds.
- In production, this pattern deserves attention: an unexpected edit to financial data, plus a short-lived file (which could be data staging or covering tracks), is a common sign of fraud or insider misuse.
- FIM tells you **what** changed and **when**, but not **who**. To attribute the change, correlate with Windows logon events (4624) on WS001 at that time, or turn on `whodata` auditing for the folder.

## Verification (production)

- Check with the payroll owner whether a bonus was approved.
- Identify the user session active on WS001 at 01:09.
- Compare the file against the last backup.

## Action

Closed as a lab test. Recommended improvement: add `whodata="yes"` to the FIM rule for sensitive folders, so alerts include the user and process that made each change.

## Lesson

FIM is only as good as its scope. Wazuh's defaults watch system files on a 12-hour schedule; sensitive business folders must be added explicitly, with real-time monitoring and change reporting turned on.
