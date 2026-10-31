# Ticket 03: Failed logons and account lockout (in progress)

| Field | Value |
| --- | --- |
| Alert | Fill in: Wazuh rule ID and description |
| Severity | Fill in: level |
| Host | DC001.BuildBetter.digital |
| Target user | BUILDBETTER\jdoe |
| Source workstation | WS001 |
| Time | Fill in |
| Source | Windows Security Events 4625 / 4771 (failed logon), 4740 (account locked out) |
| Verdict | Fill in |

## Setup

- Account lockout policy (Default Domain Policy): lockout after **5** invalid attempts, 15-minute duration and reset.
- Wrong password entered 6 times for `jdoe` on WS001.

## Query used

```
agent.name:DC001 and data.win.system.eventID:(4625 or 4740 or 4771)
```

## Timeline

| Time | Event ID | Wazuh rule | What happened |
| --- | --- | --- | --- |
| | | | |

## Evidence

| Field | Value |
| --- | --- |
| TargetUserName | |
| Workstation / IpAddress | |
| Failure reason or status code | |
| Number of failures before lockout | |

## Analysis

Fill in: how many failures, from where, over what time span. Does it look like a user mistyping, or a password-guessing pattern (many accounts, many attempts, odd source)?

## Action

- Unlock the account: `Unlock-ADAccount -Identity jdoe`
- Fill in: next steps you'd take in production (contact the user, check for other targeted accounts, block the source)
