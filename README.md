# antigravity-windows-bug-report
Technical investigation of Windows updater installation path mismatch and TLS certificate expiration issues in Google Antigravity IDE.

# Bug Report: Antigravity IDE Windows Updater Path Mismatch and Language Server TLS Certificate Expiration

**Product:** Google Antigravity IDE  
**Platform:** Windows  
**Installation directory:** `D:\Antigravity IDE`  
**Registry-reported version:** 2.6.0  
**Date observed:** November 8, 2026  
**Status:** Unresolved at the time of publication

## Summary

I encountered two independent issues while updating and launching Google Antigravity IDE on Windows.

The first issue involves an installation path mismatch between the actual application directory and the path used by the Windows updater.

The second issue involves an expired TLS certificate reported by the Antigravity Language Server, preventing successful secure communication.

I investigated both issues using PowerShell, Windows registry inspection, application logs, and executable-level testing.

## Issue 1: Windows updater uses an incorrect installation path

### Environment

My Antigravity installation is located at:

`D:\Antigravity IDE`

The executable exists and is accessible:

`D:\Antigravity IDE\Antigravity IDE.exe`

PowerShell verification:

```powershell
Test-Path "D:\Antigravity IDE\Antigravity IDE.exe"

# Output: True
```

However, the Windows uninstall registry contains the following installation location:

`C:\Users\shubh\AppData\Local\Programs\Antigravity IDE\`

The executable does not exist at that location:

```powershell
Test-Path "$env:LOCALAPPDATA\Programs\Antigravity IDE\Antigravity IDE.exe"

# Output: False
```

### Updater log

The updater log reports:

```text
Nov 08 10:25:48.611 INFO Starting:
C:\Users\shubh\AppData\Local\Programs\Antigravity IDE\Antigravity IDE.exe

Nov 08 10:25:48.611 ERRO Bad arguments:
Code path doesn't seem to exist:
C:\Users\shubh\AppData\Local\Programs\Antigravity IDE\Antigravity IDE.exe
```

### Expected behavior

The updater should correctly identify and use the existing installation directory, including custom installation locations such as D drive.

### Actual behavior

The updater attempts to access an executable at a stale C-drive location, even though the application executable exists on D drive.

### Impact

- Application update fails.
- The updater cannot locate the expected executable.
- Users with custom installation directories may experience update failures.

## Issue 2: Language Server TLS certificate expiration

After launching the application directly from D drive:

```powershell
Start-Process "D:\Antigravity IDE\Antigravity IDE.exe"
```

The application starts its browser onboarding service, but subsequent Language Server communication fails.

### Observed logs

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

The stack trace points to the application's internal Node.js TLS connection handling.

### Expected behavior

The Language Server should establish secure communication using a valid, trusted TLS certificate.

### Actual behavior

TLS certificate validation fails with `CERT_HAS_EXPIRED`, resulting in repeated connection and heartbeat errors.

### Impact

Language Server communication is disrupted. The extent of functionality affected requires further verification.

## Investigation performed

The following checks were performed:

1. Verified that the D-drive executable exists.
2. Inspected the Windows uninstall registry.
3. Confirmed that the registry points to the C-drive installation.
4. Verified that the C-drive executable is missing.
5. Examined the Windows updater log.
6. Launched Antigravity directly from D drive.
7. Captured the Language Server TLS certificate validation errors.

## Communication with the vendor

I contacted the Google Antigravity team through X to report the findings and ask whether an official bug bounty or responsible disclosure program was available.

I also communicated that I would wait an additional day for a response before publishing my findings.

As of November 10, 2026, I have not received a response.

I am publishing this report to document the reproducible behavior, share the technical investigation, and help other developers experiencing similar problems.

## Questions for the Antigravity team

1. Is custom installation path handling on Windows a known updater issue?
2. Why is the updater resolving an installation path different from the actual application location?
3. Is the Language Server TLS certificate expiration a known issue?
4. Is there an official bug reporting, responsible disclosure, or bug bounty program?

**Note:** These findings describe software reliability and TLS validation errors. I am not claiming an exploitable security vulnerability or a confirmed security impact.

I welcome technical feedback, investigation, and acknowledgement from the engineering team.

**Reporter:** Shubham Setia  
Senior Software Engineer | Frontend Technical Lead
