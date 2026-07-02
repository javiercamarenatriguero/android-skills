---
name: review-owasp-security
description: Reviews code against OWASP MASVS (Mobile Application Security Verification Standard) controls. Use when user says "security review", "check for vulnerabilities", "OWASP audit", or when code handles sensitive data, authentication, networking, local storage, or deep links. Reports findings as PASS / WARN / FAIL.
allowed-tools: Read, Grep, Glob
---

# OWASP MASVS Security Review

## When to Activate

Trigger this review when code touches any of the following areas:
- **Sensitive data handling** — tokens, passwords, PII, health data
- **Authentication & session management** — login, logout, token refresh, biometrics
- **Networking** — API calls, certificate configuration, TLS settings
- **Local storage** — databases, preferences, file system, caching
- **Deep links & WebViews** — URL schemes, intent filters, embedded web content
- **Permissions & exported components** — manifest declarations, runtime permissions

## Review Checklist

### 1. Data Storage

| Control | Expected |
|---------|----------|
| Sensitive data (tokens, passwords, PII) stored in Android Keystore / iOS Keychain | ✅ |
| No SharedPreferences / UserDefaults for sensitive data | ✅ |
| No PII logged in any environment (debug or release) | ✅ |
| Backups exclude sensitive data (`android:allowBackup="false"` or backup rules) | ✅ |
| Temporary files containing sensitive data are deleted after use | ✅ |

### 2. Networking

| Control | Expected |
|---------|----------|
| TLS 1.3 enforced | ✅ |
| Certificate pinning enabled for API endpoints | ✅ |
| No cleartext traffic (`network_security_config.xml` / ATS) | ✅ |
| Certificate validation not disabled (no `TrustAllCerts`, no `SSLSocketFactory` overrides) | ✅ |
| No sensitive data in URL query parameters (use request body or headers) | ✅ |

### 3. Authentication & Session

| Control | Expected |
|---------|----------|
| Tokens stored securely (Keystore / Keychain) | ✅ |
| Refresh token rotation implemented | ✅ |
| Session expires after inactivity | ✅ |
| Biometric auth uses system APIs (`BiometricPrompt` / `LAContext`), not custom implementations | ✅ |
| Failed auth attempts are rate-limited | ✅ |

### 4. Deep Links & WebViews

| Control | Expected |
|---------|----------|
| Deep links validated with App Links (Android) / Universal Links (iOS) | ✅ |
| WebViews: JavaScript disabled by default, enabled only when strictly necessary | ✅ |
| No `file://` access in WebViews | ✅ |
| Input validation on all deep link parameters | ✅ |
| WebViews do not load arbitrary URLs from external input | ✅ |

### 5. Permissions & Exported Components

| Control | Expected |
|---------|----------|
| Minimum necessary permissions requested, at point of use | ✅ |
| Activities / ContentProviders / BroadcastReceivers set `android:exported=false` by default | ✅ |
| ProGuard / R8 enabled for release builds (Android) | ✅ |
| No unused permissions in manifest | ✅ |

### 6. Credentials & Secrets

| Control | Expected |
|---------|----------|
| No API keys hardcoded in source code | ✅ |
| Build-time injection via `BuildConfig` (Android) / `xcconfig` (iOS) | ✅ |
| CI/CD secrets stored in GitHub Secrets (never in repo) | ✅ |
| `.gitignore` excludes local config files with secrets (`local.properties`, `.env`) | ✅ |

## How to Perform the Review

1. **Identify scope** — Determine which files and modules are affected.
2. **Run checklist** — Evaluate each applicable control against the code under review.
3. **Classify findings**:
   - **FAIL** — Immediate security risk. Blocks merge.
   - **WARN** — Potential risk or deviation from best practice. Requires justification to proceed.
   - **PASS** — Control verified and satisfied.
4. **Generate report** — Output findings in the format below.

## Expected Output

```markdown
## OWASP MASVS Audit Report

### FAIL (block merge)
- [FAIL] {Description of finding} — {file}:{line}
- [FAIL] {Description of finding} — {file}:{line}

### WARN (require justification)
- [WARN] {Description of finding} — {file}:{line}

### PASS
- [PASS] {Description of verified control}
- [PASS] {Description of verified control}

### Summary
- **FAIL**: {count}
- **WARN**: {count}
- **PASS**: {count}
- **Verdict**: {APPROVED / BLOCKED}
```

## Severity Guidelines

| Severity | Criteria | Action |
|----------|----------|--------|
| **FAIL** | Hardcoded secrets, disabled certificate validation, PII in logs, cleartext traffic | Block merge, fix immediately |
| **WARN** | Missing certificate pinning, broad permissions, no session timeout configured | Document justification or fix before release |
| **PASS** | Control is correctly implemented and verified | No action needed |

## References

- [OWASP MASVS](https://mas.owasp.org/MASVS/)
- [OWASP MASTG](https://mas.owasp.org/MASTG/)
- [Android Security Best Practices](https://developer.android.com/privacy-and-security/security-tips)
- [Apple Platform Security](https://support.apple.com/guide/security/)
