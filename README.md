# Antigravity Windows Bug Report

Technical investigation of Windows updater installation path mismatch and Language Server TLS certificate validation errors in Google Antigravity IDE.

---

# Bug Report: Windows Updater Installation Path Mismatch and Language Server TLS Certificate Expiration in Google Antigravity IDE

**Product:** Google Antigravity IDE  
**Platform:** Windows  
**Actual Installation Directory:** `D:\Antigravity IDE`  
**Registry-Reported Version:** `2.6.0`  
**Date Observed:** November 8, 2026  
**Report Updated:** November 10, 2026  
**Status:** Unresolved at the time of publication

## 1. Summary

I encountered two separate issues while updating and launching Google Antigravity IDE on Windows.

1. **Windows updater installation path mismatch:** The updater attempted to locate and launch Antigravity from a C-drive directory that had never been used for the installation, instead of the actual D-drive installation directory.
2. **Language Server TLS certificate validation failure:** After launching the application directly from its actual installation directory, the application repeatedly reported `CERT_HAS_EXPIRED` errors during Language Server communication.

I investigated these behaviors through PowerShell executable checks, Windows Registry inspection, updater logs, and application-level log analysis.

This report documents the observed behavior, supporting evidence, reproduction steps, and questions for the Antigravity engineering team.

---

## 2. Issue 1: Windows Updater References an Incorrect Installation Directory

### 2.1 Environment

My Antigravity IDE installation is located at:

```text
D:\Antigravity IDE
```

The application executable exists at:

```text
D:\Antigravity IDE\Antigravity IDE.exe
```

**Important clarification:** Antigravity IDE was installed on the D: drive. It was never installed in the following C-drive directory:

```text
C:\Users\shubh\AppData\Local\Programs\Antigravity IDE
```

### 2.2 Executable verification

I verified the actual installation executable using PowerShell:

```powershell
Test-Path "D:\Antigravity IDE\Antigravity IDE.exe"
```

**Output:**

```text
True
```

I then checked the C-drive path referenced by the updater:

```powershell
Test-Path "$env:LOCALAPPDATA\Programs\Antigravity IDE\Antigravity IDE.exe"
```

**Output:**

```text
False
```

These checks confirmed that the executable existed in the D-drive installation directory but not at the C-drive location referenced by the updater.

### 2.3 Windows Registry investigation

I inspected the Antigravity uninstall registration using:

```powershell
Get-ItemProperty `
"HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*" |
Where-Object {$_.DisplayName -like "*Antigravity*"} |
Select-Object DisplayName, InstallLocation, DisplayIcon, UninstallString
```

The registry returned the following installation location:

```text
C:\Users\shubh\AppData\Local\Programs\Antigravity IDE\
```

The `DisplayIcon` and `UninstallString` entries also referenced the same C-drive installation directory.

The registry-reported application version was:

```text
2.6.0
```

This indicates that the Windows uninstall registration did not reflect the actual D-drive installation location at the time of investigation.

### 2.4 Updater log evidence

The updater log was located at:

```text
%LOCALAPPDATA%\Temp\vscode-inno-updater-1794133548.log
```

The relevant log entries were:

```text
Nov 08 10:25:48.611 INFO Starting:
C:\Users\shubh\AppData\Local\Programs\Antigravity IDE\Antigravity IDE.exe

Nov 08 10:25:48.611 ERRO Bad arguments:
Code path doesn't seem to exist:
C:\Users\shubh\AppData\Local\Programs\Antigravity IDE\Antigravity IDE.exe
```

The updater attempted to use the C-drive executable path, which did not exist.

### 2.5 Expected behavior

The updater should correctly identify and use the application's actual installation directory.

Custom installation locations, including installations on another drive, should be handled correctly throughout the update process.

### 2.6 Actual behavior

The updater attempted to access:

```text
C:\Users\shubh\AppData\Local\Programs\Antigravity IDE\Antigravity IDE.exe
```

Instead of the actual executable:

```text
D:\Antigravity IDE\Antigravity IDE.exe
```

The resulting error indicated that the expected application executable could not be found.

### 2.7 Observed impact

- The Windows update process failed.
- The updater could not locate the expected executable.
- The application installation path recorded in Windows Registry did not match the actual installation directory.
- Users with custom installation locations may encounter similar update failures if the same behavior affects their installations.

---

## 3. Issue 2: Language Server TLS Certificate Expiration Error

### 3.1 Reproduction

To isolate the updater issue from application startup, I launched Antigravity directly from its actual D-drive installation:

```powershell
Start-Process "D:\Antigravity IDE\Antigravity IDE.exe"
```

The application process started and initialized its browser onboarding service.

However, subsequent Language Server communication produced repeated TLS certificate validation errors.

### 3.2 Observed application logs

The relevant log output included:

```text
[main] update#setState idle

[main] [uncaught exception in main]:
ConnectError: [internal] certificate has expired

[LS Main] Heartbeat error:
ConnectError: [internal] certificate has expired

code: 13

cause:
Error: certificate has expired

code: CERT_HAS_EXPIRED
```

The associated Node.js stack trace included TLS connection handling, specifically `TLSSocket.onConnectSecure`.

### 3.3 Expected behavior

The Language Server should establish secure communication using a valid certificate chain and successfully maintain its required connections and heartbeat operations.

### 3.4 Actual behavior

The application reported:

```text
CERT_HAS_EXPIRED
```

This resulted in repeated connection and heartbeat errors during Language Server communication.

### 3.5 Observed impact

- Language Server communication encountered repeated TLS validation failures.
- Heartbeat operations reported errors.
- The extent of the impact on code intelligence and other IDE functionality requires further verification.

**Note:** The observed `CERT_HAS_EXPIRED` error establishes a certificate validation failure. The exact certificate involved and its validity dates require further verification before attributing the issue to a specific bundled certificate.

---

## 4. Additional Observation: VS Code-Related Updater Components

During the update attempt, the application displayed the following message:

```text
Failed to install Visual Studio Code update.
```

The temporary updater log was named:

```text
vscode-inno-updater-1794133548.log
```

Its location was:

```text
%LOCALAPPDATA%\Temp\
```

The log contained the incorrect C-drive executable path described in Issue 1.

### Why this observation matters

The error message and temporary log filename contain Visual Studio Code-related terminology, even though the application being updated is Google Antigravity IDE.

This suggests that Antigravity may use or share VS Code-derived updater components or infrastructure.

However, these observations alone do not establish that Antigravity uses an identical updater implementation or that the two products share the same update mechanism.

The confirmed issue remains the updater's attempt to resolve an executable path that did not correspond to the actual installation directory.

---

## 5. Investigation Performed

The following checks were performed during the investigation:

1. Inspected the Antigravity installation directory on D:.
2. Verified that `Antigravity IDE.exe` existed in the D-drive installation directory.
3. Checked whether the corresponding executable existed in the C-drive Programs directory.
4. Inspected the Windows uninstall Registry entries for Antigravity.
5. Identified the mismatch between the actual installation directory and the registry-reported installation location.
6. Examined the Windows updater log and identified the incorrect executable path.
7. Launched Antigravity directly from the D-drive installation.
8. Examined the application logs and captured the Language Server TLS certificate validation errors.
9. Identified the VS Code-related updater error message and temporary log filename.

---

## 6. Communication with the Vendor

I contacted the Google Antigravity team through X to report my findings and ask whether an official bug bounty or responsible disclosure program was available.

I also communicated that I would wait an additional day for a response before publishing the technical findings.

As of November 10, 2026, I had not received a response.

I am publishing this report to document the observed behavior, provide reproducible technical evidence, and help other developers who may encounter similar issues.

---

## 7. Questions for the Antigravity Engineering Team

1. Is the Windows updater installation path mismatch a known issue?
2. Why does the updater resolve a C-drive installation path when the application was installed on D:?
3. Why do the Windows uninstall registration entries reference a directory that was never used for this installation?
4. Is the Language Server `CERT_HAS_EXPIRED` error a known issue?
5. Which certificate is involved in the Language Server TLS validation failure, and are there known certificate lifecycle or renewal issues?
6. Is there an official bug reporting, responsible disclosure, or bug bounty program for Google Antigravity?

---

## 8. Scope and Disclosure Note

This report documents two observed software issues:

- A Windows updater installation path mismatch.
- Language Server TLS certificate validation errors.

These findings currently demonstrate application update and communication reliability problems.

I am **not claiming a confirmed exploitable security vulnerability, unauthorized access, data exposure, or system compromise**. The security implications, if any, require further investigation by the Antigravity engineering team.

All observations are based on local testing and the logs available during the investigation.

I welcome technical feedback, investigation, and acknowledgement from the Antigravity engineering team.

---

**Reporter:** Shubham Setia  
**Role:** Senior Software Engineer | Frontend Technical Lead  
**Platform:** Windows  
**Report Date:** November 10, 2026
