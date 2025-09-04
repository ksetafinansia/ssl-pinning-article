# Section 5: Pinning Scope - Practical Implementation Guide

## Overview
This section explains exactly which servers to pin and how to implement SSL pinning in a real-world scenario. We'll use a practical example to show you how to scope pinning correctly for your API gateway architecture.

## Real-World Example: API Gateway Pinning

### Understanding Your Architecture
Let's say you have this setup:
```
Client Applications:
├── Mobile Apps → apigee.kreditplus.com → {
│                                        ├── los.kreditplus.com (internal service)
│                                        ├── api.tokopedia.com (external service)
│                                        └── other internal/external services
│                                        }
├── Mobile Apps → platform.kreditplus.com (direct connection)
└── Web Apps → apigee.kreditplus.com → {same backend services as above}
```

**Key Question:** Where should you apply SSL pinning?

**Answer:** Pin ONLY the connection from clients to `apigee.kreditplus.com`

**Important:** Do NOT pin `platform.kreditplus.com` even though clients connect to it directly

### Why Pin Only the API Gateway?

#### ✅ What You Should Pin
**Target:** `apigee.kreditplus.com` (your API gateway)
- **Reason:** This is the critical entry point you control
- **Security:** Protects the most important connection (clients → gateway)
- **Manageability:** You control this certificate and its rotation
- **Impact:** Single point of pinning reduces complexity
- **Clients:** Both mobile apps and web apps should pin this

#### ❌ What You Should NOT Pin
**Platform Service** (`platform.kreditplus.com`):
- **Reason:** Different service with separate certificate management
- **Risk:** Creates multiple pinning points to manage
- **Alternative:** Use normal TLS validation for platform connections
- **Flexibility:** Allows independent certificate rotation

**Internal Services** (like `los.kreditplus.com`):
- **Reason:** These are server-to-server connections within your infrastructure
- **Security:** Already protected by internal network security
- **Complexity:** Would require managing multiple pins

**External Services** (like `api.tokopedia.com`):
- **Reason:** You don't control these certificates
- **Risk:** External providers can change certificates without notice
- **Alternative:** Use normal TLS validation for these connections

## Step-by-Step Implementation Plan

### Phase 1: Staging Environment Setup

#### Step 1: Prepare Staging Infrastructure
1. **Set up staging API gateway:** `apigee.kbfinansia.com`
2. **Ensure staging mirrors production architecture:**
   ```
   Client Applications:
   ├── Mobile Apps (Test) → apigee.kbfinansia.com → {
   │                                               ├── los-staging.kreditplus.com
   │                                               ├── api.tokopedia.com (same external)
   │                                               └── other staging services
   │                                               }
   ├── Mobile Apps (Test) → platform.kbfinansia.com (normal TLS, no pinning)
   └── Web Apps (Test) → apigee.kbfinansia.com → {same backend services as above}
   ```

#### Step 2: Extract Staging Certificate Pin
```bash
# Extract pin for staging gateway ONLY
HOST=apigee.kbfinansia.com
echo | openssl s_client -connect ${HOST}:443 -servername ${HOST} 2>/dev/null | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | openssl base64
```

#### Step 3: Configure Client Applications for Staging
**Important:** Configure both mobile and web apps to pin ONLY `apigee.kbfinansia.com`

*Note: Detailed implementation code examples are provided in Section 6 - Platform Implementation Summary*

#### Step 4: Test Staging Implementation
1. **Deploy test applications** with staging pinning (mobile and web)
2. **Verify connections work:**
   - Mobile/Web → `apigee.kbfinansia.com` ✅ (should work with pinning)
   - Mobile → `platform.kbfinansia.com` ✅ (normal TLS, no pinning)
   - Gateway → `los-staging.kreditplus.com` ✅ (normal TLS, no pinning)
   - Gateway → `api.tokopedia.com` ✅ (normal TLS, no pinning)
3. **Test certificate mismatch scenarios**
4. **Monitor for any connection failures**

### Phase 2: Production Environment Setup

#### Step 1: Extract Production Certificate Pin
```bash
# Extract pin for production gateway
HOST=apigee.kreditplus.com
echo | openssl s_client -connect ${HOST}:443 -servername ${HOST} 2>/dev/null | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | openssl base64
```

#### Step 2: Prepare Backup Pin
```bash
# Extract pin from backup certificate (if available)
openssl rsa -in apigee_backup_key.pem -pubout -outform der | openssl dgst -sha256 -binary | openssl base64
```

#### Step 3: Configure Production Client Applications
**Important:** Configure both mobile and web apps to pin ONLY `apigee.kreditplus.com`

*Note: Detailed implementation code examples for Android, iOS, and Web applications are provided in Section 6 - Platform Implementation Summary*

## What Happens in Practice

### When Client Applications Connect to apigee.kreditplus.com
1. **Mobile/Web app initiates HTTPS connection**
2. **SSL pinning validates certificate against stored pin**
3. **If pin matches:** Connection proceeds ✅
4. **If pin doesn't match:** Connection fails immediately ❌
5. **No fallback to normal certificate validation**

### When Client Applications Connect to platform.kreditplus.com
1. **Mobile app initiates HTTPS connection**
2. **Normal TLS validation is used (no pinning)**
3. **Standard CA certificate validation applies**
4. **Connection works as long as certificate is valid**

### When apigee.kreditplus.com Connects to Backend Services
1. **API gateway makes HTTPS calls to backend services**
2. **Normal TLS validation is used (no pinning)**
3. **Standard CA certificate validation applies**
4. **Connections work as long as certificates are valid**

## Testing and Validation Strategy

### Staging Environment Tests
1. **Positive Test:** Verify mobile/web apps connect successfully to staging gateway
2. **Negative Test:** Change staging certificate and verify connection fails
3. **Platform Test:** Confirm mobile apps can still reach platform.kbfinansia.com without pinning
4. **Backend Test:** Confirm gateway can still reach all backend services
5. **Performance Test:** Measure any latency impact from pinning validation

### Production Rollout Strategy
1. **Gradual Rollout:** Deploy to small percentage of users first
2. **Monitor Metrics:** Watch for connection failure rates
3. **Rollback Plan:** Prepare app version without pinning if issues arise
4. **Full Deployment:** Expand to all users once validated

## Common Pitfalls and How to Avoid Them

### ❌ Don't Pin Everything
**Wrong Approach:**
```
// This is TOO MUCH pinning - DON'T DO THIS
Pin apigee.kreditplus.com ❌ Too many pins
Pin platform.kreditplus.com ❌ Unnecessary complexity  
Pin los.kreditplus.com ❌ Internal service
Pin api.tokopedia.com ❌ External service you don't control
```

**Correct Approach:**
```
// Pin ONLY what you control and need to secure
Pin ONLY: apigee.kreditplus.com ✅
Use normal TLS for everything else ✅
```

**Key Principle:** Even though mobile apps connect to both `apigee.kreditplus.com` and `platform.kreditplus.com`, only pin the API gateway to keep complexity manageable.

### ❌ Don't Forget Backup Pins
**Wrong:** Only pin current certificate
**Right:** Always include backup certificate pin

### ❌ Don't Skip Staging Testing
**Wrong:** Deploy pinning directly to production
**Right:** Always test thoroughly in staging environment first

## Summary

**Simple Rule:** Pin only the servers you directly control and that serve as the primary entry point to your system.

**For Your Case:**
- ✅ Pin: `apigee.kreditplus.com` (your controlled API gateway)
- ❌ Don't Pin: `platform.kreditplus.com` (separate service, adds complexity)
- ❌ Don't Pin: Internal services like `los.kreditplus.com`
- ❌ Don't Pin: External services like `api.tokopedia.com`

**Implementation Path:**
1. Test in staging (`apigee.kbfinansia.com`)
2. Deploy to production (`apigee.kreditplus.com`)
3. Monitor and maintain

**Client Applications:** Both mobile and web apps should follow the same pinning strategy - pin only the API gateway.

*For detailed platform-specific implementation code examples, see Section 6 - Platform Implementation Summary.*
