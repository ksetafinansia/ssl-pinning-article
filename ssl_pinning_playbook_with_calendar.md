# 🔐 SSL Pinning Playbook

**Pilot Domain:** `api.kreditplus.com`  
**Staging Test Domain:** `kbfinansia.com`  
**Production Domain:** `kreditplus.com`  

This playbook is the official guideline for **developers, DevOps, and SecOps** to implement and operate **SSL Public Key Pinning (SPKI hash)**.

---

## 1. Scope & Principles
- Pinning applies only to:
  - `kbfinansia.com` (staging)
  - `api.kreditplus.com` (pilot)
  - `kreditplus.com` (production)
- Method: Pin the **SHA-256 SPKI hash** of the TLS public key.
- Always ship at least **two pins**:
  - Current key
  - Backup/future key
- Fail **closed** in production, fail **open** in staging/debug.

---

## 2. Audience & Responsibilities

### Developers
- Implement pinning in client apps (Android, iOS, Flutter, KMM).  
- Add kill-switch hooks (remote config).  
- Ensure telemetry/logs are emitted.  
- Validate staging and pilot domains before prod rollout.

### DevOps
- Generate TLS keypairs, CSRs, and manage cert issuance.  
- Deploy and rotate certificates in gateways/CDNs.  
- Maintain backup keypairs and document SPKI pins.  
- Provide observability dashboards (pin failure rate, cert validity).

### SecOps
- Approve rollout plans and rotation schedules.  
- Monitor alerts for pinning failures.  
- Approve kill-switch usage in emergencies.  
- Audit pin registry and incident logs.

---

## 3. Workflow
1. Test in **staging** (`kbfinansia.com`).  
2. Pilot rollout in **`api.kreditplus.com`**.  
3. Enforce in **production** (`kreditplus.com`).  

---

## 4. Pin Extraction Commands

**From live host**
```bash
HOST=api.kreditplus.com
echo | openssl s_client -connect ${HOST}:443 -servername ${HOST} 2>/dev/null  | openssl x509 -pubkey -noout  | openssl pkey -pubin -outform der  | openssl dgst -sha256 -binary  | openssl base64
```

**From backup keypair**
```bash
openssl rsa -in backup_key.pem -pubout -outform der  | openssl dgst -sha256 -binary | openssl base64
```

---

## 5. Pinning Scope

### 5.1 Requests to the pinned domain
- Example: `api.kreditplus.com`  
- ✅ Pinning enforced.  
- ❌ If mismatch → request fails.

### 5.2 Requests to other subdomains
- Example: `apigee.kreditplus.com`  
- If only `api.kreditplus.com` pinned → normal TLS only.  
- If wildcard `*.kreditplus.com` pinned → risk of mismatch → fail.

### 5.3 Requests to other domains
- Example: `api.tokopedia.com`  
- ✅ Normal TLS, no pinning applied.

**Best practice:** Always pin **exact hostnames** only.

---

## 6. Platform Implementation (Summary)

- **Android (Kotlin)** → OkHttp `CertificatePinner` or Network Security Config.  
- **iOS (Swift)** → URLSessionDelegate or TrustKit.  
- **Flutter** → `dio_certificate_pin` or custom adapter.  
- **KMM** → Android uses OkHttp pinner; iOS uses Swift delegate.  
- **Go (server)** → `tls.Config.VerifyPeerCertificate`.  
- **Nuxt (SSR)** → custom `https.Agent` with `checkServerIdentity`.

---

## 7. Rollout & Contingency

### 7.1 Rollout Timeline

**T-30 to T-21 (Prep)**
- Generate current + backup pins.  
- Implement remote kill-switch in clients.  
- Add telemetry for pin success/failure.

**T-20 to T-14 (Publish “latent”)**
- Release app with pinning code + kill-switch.  
- Default flag OFF.  
- Staged rollout in app stores.

**T-13 to T-7 (Pilot enable small cohort)**
- Enable pinning ON (via remote flag) for 1–5% of users on `api.kreditplus.com`.  
- Observe telemetry; ramp gradually.

**T-6 to T-3 (Ramp pilot)**
- Increase rollout to 25–50% users.  
- Validate no spike in failures.

**T-2 to T-0 (Pilot full)**
- Enable 100% for pilot domain.  
- Keep kill-switch available.

**T+7 onward (Production)**
- Enable pinning for `kreditplus.com` gradually (5% → 25% → 100%).  
- Maintain kill-switch for 2–4 weeks.

### 7.2 Contingency (Kill-Switch)

**How**
- **Mobile** → remote config disables pinning check, TLS still validated.  
- **Server** → set `SSL_PINNING_ENABLED=false` to bypass pinned agent.

**Security Concern**
- With pinning **ON** → client only trusts our exact server key.  
- With pinning **OFF** → client trusts any CA-issued cert.  
- Still encrypted, but less secure → possible rogue CA impersonation.

**Rule of Thumb**
- Always ship with kill-switch.  
- Never hard-enforce pinning without it.

---

## 8. Rotation Calendar

| Time Window       | Devs                                  | DevOps                              | SecOps                     |
|-------------------|---------------------------------------|-------------------------------------|----------------------------|
| **T-30 days**     | Prepare code with kill-switch.        | Generate current + backup keys.     | Approve rotation plan.     |
| **T-14 days**     | Release app with pins (flag OFF).     | Issue cert with current key.        | Validate SPKI registry.    |
| **T-7 days**      | Validate staging env.                 | Deploy cert to staging.             | Approve pilot enable.      |
| **T-5 days**      | Enable small cohort on pilot.         | Monitor cert usage.                 | Watch for alerts.          |
| **T-3 days**      | Ramp cohort to 50%.                   | Ensure CDN/API gateway stable.      | Security validation.       |
| **T-0 (cutover)** | Confirm apps accept both pins.        | Deploy cert with backup key.        | Monitor for pin failures.  |
| **T+7 days**      | Update code with new backup pin.      | Remove old key from registry.       | Confirm low error rates.   |
| **T+14 days**     | Plan next backup key.                 | Generate next backup keypair.       | Approve next cycle.        |

---

## 9. Testing & Validation
- Staging: simulate wrong cert → expect pin failure.  
- Pilot: confirm clients accept both current + backup pins.  
- Prod: strict enforcement with rollback available.

---

## 10. Monitoring & Alerts
- Log pin failures by host and app version.  
- New Relic / ELK dashboards.  
- Alert if failure rate exceeds baseline.  
- Audit pin registry quarterly.

---

## 11. Example Remote Config JSON

```json
{
  "ssl_pinning_enabled": true,
  "ssl_pinning_hosts": ["api.kreditplus.com"],
  "min_app_version_enforced": "5.3.0"
}
```

- Enable per-host, per-version.  
- Avoid breaking old versions.

---

## 12. Operational Checklist

- [ ] Current + backup pins generated.  
- [ ] Clients shipped with kill-switch.  
- [ ] Staged rollout in app stores.  
- [ ] Staging validated with wrong cert.  
- [ ] Pilot enabled via remote flag.  
- [ ] Production enabled via ramp.  
- [ ] Monitoring live.  
- [ ] Contingency tested.  

---
