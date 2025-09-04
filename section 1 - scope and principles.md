# Section 1: Scope & Principles

## Overview
This section defines the fundamental scope and core principles for SSL certificate pinning implementation across Kredit Plus environments.

## Content

### Pinning Scope
- Pinning applies only to:
  - `kbfinansia.com` (staging)
  - `api.kreditplus.com` (pilot)
  - `kreditplus.com` (production)

### Technical Method
- Method: Pin the **SHA-256 SPKI hash** of the TLS public key.

### Key Principles
- Always ship at least **two pins**:
  - Current key
  - Backup/future key
- Fail **closed** in production, fail **open** in staging/debug.

## Security Rationale
The dual-pin approach ensures continuity during certificate rotations while maintaining strong security posture. The fail-closed policy in production prevents potential man-in-the-middle attacks, while fail-open in staging allows for development flexibility.

## Compliance Notes
- Aligns with OWASP mobile security guidelines
- Supports certificate transparency requirements
- Enables emergency response capabilities
