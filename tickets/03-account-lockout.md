# Ticket 03: Failed logons and account lockout

| Field | Value |
| --- | --- |
| Alert | Wazuh rule 60115, "User account locked out (multiple login errors)" |
| Severity | Level 9 |
| Host | DC001.BuildBetter.digital |
| Target user | BUILDBETTER\jdoe (SID ending -1112) |
| Source workstation | WS001 |
| Time | 2026-10-29, 23:24:40 CDT |
| Source | Windows Security Events 4625 (failed logon, WS001), 4740 (account locked out, DC001) |
| Verdict | **Benign**, planned lab test; account unlocked |

## Setup

- Account lockout policy (Default Domain Policy): lockout after **5** invalid attempts, 15-minute duration and reset.
- Wrong password entered repeatedly for `jdoe` at the WS001 sign-in screen.

## Query used

```
agent.name:DC001 and data.win.system.eventID:(4625 or 4740 or 4771)
```

Results: 16 hits in the 24-hour window.

## Timeline

| Time (CDT) | Host | Wazuh rule | Level | What happened |
| --- | --- | --- | --- | --- |
| 23:24:18 | WS001 + DC001 | 60122 / 60104 | 5 | Failed logon for jdoe; the domain controller rejects it |
| 23:24:25 | WS001 + DC001 | 60122 / 60104 | 5 | Failed logon |
| 23:24:33 | WS001 + DC001 | 60122 / 60104 | 5 | Failed logon |
| 23:24:40 | WS001 + DC001 | 60122 / 60104 | 5 | Failed logon that hit the threshold |
| **23:24:40** | **DC001** | **60115** | **9** | **jdoe locked out (event 4740)** |
| 23:24:48 | DC001 | 60104 | 5 | Attempt after the lockout |
| 23:31:38, 23:31:56 | DC001 | 60104 | 5 | More attempts after the lockout |

## Evidence (event 4740)

| Field | Value | Meaning |
| --- | --- | --- |
| Account Name | jdoe | The account that was locked |
| Security ID | S-1-5-21-...-1112 | Matches jdoe's SID from account creation, so it's the right account |
| Caller Computer Name | WS001 | Where the failed attempts came from |

## Analysis

- All failures targeted one account (jdoe) from one internal machine (WS001), about 7 seconds apart, at the interactive sign-in screen.
- Each attempt appears twice: WS001 logs the bad password locally (4625, rule 60122), and DC001 logs the domain refusing it (rule 60104). Correlating both shows where the attempts came from.
- Severity rises from 5 to 9 at the lockout. A single failure is routine; repeated failures ending in a lockout warrant a look.
- Attempts continued after the lockout (23:24:48 and 23:31). Those are expected to be rejected because the account was locked.
- Pattern fits a user mistyping a password, not a password-spraying attack. Spraying usually shows **many accounts** tried with a few passwords each, often from one source or over the network.

## Action

- Unlocked the account:
  ```powershell
  Unlock-ADAccount -Identity jdoe
  ```
- In production: contact the user to confirm it was them before unlocking, check whether other accounts failed from the same source, and reset the password if the user didn't make the attempts.
