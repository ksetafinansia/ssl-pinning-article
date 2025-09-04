# Section 2: Audience & Responsibilities

## Overview
This section outlines the roles and responsibilities for each team involved in SSL pinning implementation and operations.

## Content

### Developers
**Primary Responsibilities:**
- Implement pinning in client apps (Android, iOS, Flutter, KMM)
- Add kill-switch hooks (remote config)
- Ensure telemetry/logs are emitted
- Validate staging and pilot domains before prod rollout

**Key Deliverables:**
- Client-side pinning implementation
- Remote configuration integration
- Telemetry and logging systems
- Pre-production validation reports

### DevOps
**Primary Responsibilities:**
- Generate TLS keypairs, CSRs, and manage cert issuance
- Deploy and rotate certificates in gateways/CDNs
- Maintain backup keypairs and document SPKI pins
- Provide observability dashboards (pin failure rate, cert validity)

**Key Deliverables:**
- Certificate lifecycle management
- Infrastructure deployment automation
- SPKI pin documentation and registry
- Monitoring and alerting systems

### SecOps
**Primary Responsibilities:**
- Approve rollout plans and rotation schedules
- Monitor alerts for pinning failures
- Approve kill-switch usage in emergencies
- Audit pin registry and incident logs

**Key Deliverables:**
- Security approval workflows
- Incident response procedures
- Compliance audits and reports
- Emergency response protocols

## Cross-Team Coordination
- Weekly sync meetings during implementation phases
- Shared responsibility matrix for incident response
- Escalation procedures for emergency scenarios
