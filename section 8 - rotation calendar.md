# Section 8: Rotation Calendar

## Overview
This section provides a detailed timeline and responsibility matrix for certificate rotation operations, ensuring coordinated execution across all teams involved in SSL pinning management.

## Content

### Rotation Timeline Matrix

| Time Window       | Developers                            | DevOps                              | SecOps                     |
|-------------------|---------------------------------------|-------------------------------------|----------------------------|
| **T-30 days**     | Prepare code with kill-switch         | Generate current + backup keys      | Approve rotation plan      |
| **T-14 days**     | Release app with pins (flag OFF)     | Issue cert with current key         | Validate SPKI registry     |
| **T-7 days**      | Validate staging environment          | Deploy cert to staging              | Approve pilot enable       |
| **T-5 days**      | Enable small cohort on pilot         | Monitor cert usage                  | Watch for alerts           |
| **T-3 days**      | Ramp cohort to 50%                   | Ensure CDN/API gateway stable       | Security validation        |
| **T-0 (cutover)** | Confirm apps accept both pins        | Deploy cert with backup key         | Monitor for pin failures   |
| **T+7 days**      | Update code with new backup pin      | Remove old key from registry        | Confirm low error rates    |
| **T+14 days**     | Plan next backup key                 | Generate next backup keypair        | Approve next cycle         |

## Detailed Phase Descriptions

### Pre-Rotation Phase (T-30 to T-14)
**Objective:** Prepare all systems and teams for certificate rotation

**Developer Activities:**
- Code review for kill-switch implementation
- Update remote configuration schemas
- Prepare telemetry collection enhancements

**DevOps Activities:**
- Certificate authority coordination
- Backup key generation and secure storage
- Infrastructure readiness validation

**SecOps Activities:**
- Risk assessment for rotation impact
- Approve rotation schedule and procedures
- Validate security controls and monitoring

### Staging Validation Phase (T-14 to T-7)
**Objective:** Validate new certificates in controlled environment

**Key Activities:**
- Deploy new certificates to staging
- Test application compatibility
- Validate pin extraction procedures
- Confirm monitoring and alerting systems

### Pilot Rollout Phase (T-7 to T-0)
**Objective:** Limited production testing with gradual user expansion

**Rollout Strategy:**
- 1% user cohort initial activation
- Monitor success rates and performance
- Gradual expansion to 50% of pilot users
- Final validation before production cutover

### Production Cutover (T-0)
**Objective:** Deploy new certificate to production environment

**Critical Activities:**
- Coordinate deployment across all infrastructure
- Activate monitoring for both old and new pins
- Validate application acceptance of new certificate
- Maintain rollback capability

### Post-Rotation Phase (T+1 to T+14)
**Objective:** Stabilize new certificate and prepare for next cycle

**Cleanup Activities:**
- Remove old pins from application code
- Update documentation and registries
- Generate next rotation backup keys
- Conduct post-rotation security review

## Emergency Procedures

### Unplanned Rotation
- Expedited timeline (T-7 to T-0 compressed to 24-48 hours)
- Enhanced monitoring and team availability
- Streamlined approval processes for security incidents

### Rotation Failure Scenarios
- Immediate kill-switch activation procedures
- Rollback to previous certificate version
- Incident response team activation
- Communication protocols with stakeholders
