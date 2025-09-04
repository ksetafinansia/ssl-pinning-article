# Section 5: Pinning Scope

## Overview
This section defines the precise scope of SSL pinning enforcement, detailing which requests are subject to pinning validation and which follow standard TLS procedures.

## Content

### 5.1 Requests to Pinned Domains
**Example:** `api.kreditplus.com`
- ✅ **Pinning enforced** - SPKI hash validation required
- ❌ **If mismatch** → Request fails immediately
- 🔒 **Security Level:** Maximum (only accepts specific certificate)

**Behavior:**
- Client validates server certificate against stored SPKI hashes
- Connection rejected if certificate doesn't match pinned values
- No fallback to standard CA validation

### 5.2 Requests to Other Subdomains
**Example:** `apigee.kreditplus.com`
- **If only `api.kreditplus.com` pinned** → Normal TLS validation only
- **If wildcard `*.kreditplus.com` pinned** → Risk of mismatch failures
- ⚠️ **Caution:** Wildcard pinning can cause unexpected failures

**Behavior:**
- Pinning scope determined by exact hostname matching
- Subdomain inheritance only with explicit wildcard configuration
- Standard TLS/CA validation for non-pinned subdomains

### 5.3 Requests to External Domains
**Example:** `api.tokopedia.com`
- ✅ **Normal TLS only** - Standard CA validation
- 🔄 **No pinning applied** - Full certificate chain validation
- 📋 **Standard Security:** CA-based trust model

**Behavior:**
- No SPKI hash validation performed
- Relies on operating system's CA trust store
- Standard TLS handshake and validation procedures

## Best Practices

### Hostname Precision
- **Always pin exact hostnames only**
- Avoid wildcard pinning unless absolutely necessary
- Document all pinned domains in central registry
- Regular audit of pinning scope

### Implementation Guidelines
- Pin specific services, not entire domains
- Consider load balancer and CDN implications
- Plan for certificate rotation across pinned services
- Test scope changes in staging first

## Risk Assessment

### Over-Pinning Risks
- Service disruption during certificate changes
- Increased operational complexity
- Higher failure rates from scope mismatches

### Under-Pinning Risks
- Reduced security for critical APIs
- Potential man-in-the-middle vulnerabilities
- Inconsistent security posture
