# Ticket 01: Executable dropped in a folder commonly used by malware

| Field | Value |
| --- | --- |
| Alert | Wazuh rule 92213, "Executable file dropped in folder commonly used by malware" |
| Severity | Level 15 |
| Host | WS001.BuildBetter.digital |
| User | BUILDBETTER\jdoe |
| Time | 2026-10-24, 16:46 CDT (21:46 UTC) |
| Source | Sysmon Event ID 11 (File created) |
| Verdict | **False positive**, expected browser activity |

## Evidence

| Field | Value |
| --- | --- |
| Image (process) | `C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe` |
| TargetFilename | `C:\Users\jdoe\AppData\Local\Temp\msedge_chrome_Unpacker_BeginUnzipping...\well_known_domains.dll` |
| RuleName (Sysmon) | DLL |

## Analysis

- The file was created by Microsoft Edge running from its standard install path.
- The folder name (`msedge_chrome_Unpacker_BeginUnzipping`) matches how Chromium-based browsers unpack component updates into the user's Temp folder.
- The rule fires on any executable or DLL written to Temp, because malware often stages there. That makes it high severity but noisy.

## Verification steps

- Confirm the parent process is signed by Microsoft:
  ```powershell
  Get-AuthenticodeSignature "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" | Select-Object Status, SignerCertificate
  ```
- Use "View surrounding documents" in Wazuh to check for anything else loading or running the DLL.

## Action

Closed as a false positive. Recommended tuning: exclude `msedge.exe` writing to `msedge_chrome_Unpacker_*` paths, so analysts aren't paged for routine browser updates.

## Lesson

Separate what the log shows (process, file, user, path) from what you conclude. Only write the conclusion after verifying it.
