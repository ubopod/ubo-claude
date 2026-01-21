---
name: security-review
description: Security review skill for ubo_app on Raspberry Pi: device secrets, service isolation, RPC/web UI hardening, filesystem safety, privilege boundaries, and dependency hygiene.
---

# Security Review (ubo_app / Raspberry Pi)

This skill is for reviewing changes to `ubo_app`, an embedded-ish Raspberry Pi application with:

- multiple services (often threaded, each with its own event loop)
- local hardware access (GPIO/camera/audio)
- network-facing components (WiFi config, web UI, gRPC/RPC)
- system-level operations (install/update scripts, privileged helpers)

Security here is **not** just “web app security” — you must consider **physical access**, **local network threats**, **service isolation**, and **privilege separation**.

---

## Threat model (assume these are true)

- The device may be on an untrusted LAN.
- Attackers may have partial access:
  - local network access to exposed ports
  - QR inputs / keyboard inputs / UI inputs
  - filesystem access via misconfig (world-readable secrets) or stolen SD card
  - optional remote access services (SSH, VS Code tunnel, RPi Connect)
- Hardware / OS is not a strong sandbox: treat every service boundary as a security boundary.

---

## Security checklist (prioritized)

### 1) Secrets & credentials (CRITICAL)

**Never commit**:

- WiFi credentials
- API keys (cloud services, AI providers)
- tokens (web UI auth tokens, pairing tokens)
- private keys / certs

Device-friendly practices:

- Store secrets in **environment variables** or **device-local secret storage** (file permissions locked down).
- Ensure secrets are **not logged** and **not included in snapshots**.
- Redact secrets in debug dumps and exceptions.

Verification:

- [ ] No hardcoded secrets in code or docs
- [ ] No secrets in logs
- [ ] Secrets files (if any) are `0600` and owned by the correct user
- [ ] Update/deploy scripts don’t echo secrets

---

### 2) Network surface area (CRITICAL)

Treat every exposed port as hostile input.

If you add or modify:

- web UI endpoints
- gRPC/RPC services
- HTTP calls / websocket endpoints
- hotspot / onboarding flows

Verify:

- [ ] Default bind is **localhost** unless explicitly required
- [ ] If binding to LAN: authentication is present and strong
- [ ] Rate limiting / backpressure for expensive endpoints
- [ ] Clear firewall expectations (document ports and intended exposure)
- [ ] No debug endpoints left enabled

---

### 3) Authentication & authorization (CRITICAL)

For embedded devices, auth often looks like:

- local pairing code / QR-based pairing
- device-local admin credentials
- short-lived tokens for web UI
- optional mTLS for RPC (if applicable)

Verify:

- [ ] Sensitive actions require auth (WiFi changes, user management, remote access toggles, filesystem ops)
- [ ] Auth tokens are stored safely (not world-readable, not in logs)
- [ ] Authorization exists (admin vs user, local vs remote)
- [ ] “First boot” / “setup mode” is time-bounded and clearly indicated

---

### 4) Privilege boundaries (CRITICAL)

Anything under `system_manager/` or scripts that require root is high risk.

Verify:

- [ ] Least privilege: only the minimum commands run as root
- [ ] No arbitrary command execution from user input
- [ ] Sudoers rules are narrow (no `NOPASSWD: ALL`)
- [ ] File permissions and ownership are correct for created files/dirs
- [ ] Root-only operations are audited/logged (without secrets)

---

### 5) Filesystem safety (HIGH)

Common embedded vulnerabilities:

- path traversal (`../../etc/shadow`)
- unsafe deletes/overwrites
- symlink attacks

Verify:

- [ ] All user-controlled paths are validated / normalized
- [ ] Writes are restricted to known safe directories
- [ ] Avoid following symlinks for sensitive operations
- [ ] File uploads (if any) are validated (type/size) and stored safely

---

### 6) Command / shell injection (HIGH)

If any code calls subprocesses:

- prefer argument lists, not shell strings
- avoid `shell=True`

Verify:

- [ ] No `shell=True` with user-controlled input
- [ ] Commands use explicit argv arrays
- [ ] Output parsing is robust and bounded

---

### 7) Input validation at boundaries (HIGH)

Boundaries include:

- QR payloads
- web UI requests
- RPC/gRPC messages
- config files / persistent store
- hardware-reported values (treat as untrusted; drivers can glitch)

Verify:

- [ ] Inputs are validated with explicit schemas/types
- [ ] Invalid inputs fail safely (no crashes, no partial writes)
- [ ] Errors don’t leak secrets

For protobuf/gRPC:

- validate message fields (length/ranges/enums) before use
- guard expensive operations (rate limit / auth)

---

### 8) Service isolation & cross-service calls (MEDIUM)

Even though it’s one codebase, keep boundaries strict:

- services should communicate via actions/events
- don’t share device handles across services
- keep “dangerous” capabilities (filesystem ops, remote access toggles) isolated and audited

Verify:

- [ ] No direct cross-service business logic calls added
- [ ] Hardware access remains single-owner

---

### 9) Web UI security (MEDIUM but important)

If `090-web-ui` exists, you still need classic web security:

- XSS: sanitize untrusted HTML; avoid unsafe templating
- CSRF: if cookies are used, enforce CSRF protections and SameSite
- CORS: lock down origins

Verify:

- [ ] No unsafe HTML rendering of user-controlled content
- [ ] Cookies (if used) are `HttpOnly`, `Secure`, appropriate `SameSite`
- [ ] CORS is explicit (no `*` with credentials)

---

### 10) Dependency hygiene (MEDIUM)

Verify:

- [ ] Dependencies pinned/locked (uv lockfile / constraints)
- [ ] Known vuln scan is part of the workflow (where possible)
- [ ] High-risk deps reviewed (crypto, auth, parsers)

---

## Review output format

When reviewing a PR/change set, report issues as:

```
[SEVERITY] Title
File: path:line
Risk: what an attacker gains
Fix: specific change recommendation
```

Severity guidance:

- **CRITICAL**: auth bypass, RCE, secrets exposure, root escalation
- **HIGH**: path traversal, injection, unsafe default exposure
- **MEDIUM**: missing hardening, weak rate limits, risky defaults
- **LOW**: defense-in-depth improvements

---

## Quick “go/no-go” checks

- [ ] No secrets committed or logged
- [ ] No new unauthenticated network endpoints
- [ ] No root-level code paths accept untrusted input
- [ ] Filesystem operations are safe against traversal/symlinks
- [ ] Subprocess usage is injection-safe

