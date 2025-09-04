# Section 10: Remote Configuration Management

## Overview
This section defines the remote configuration system used to control SSL pinning behavior across applications, enabling dynamic control without requiring application updates. This configuration strategy supports the kill switch implementation detailed in Section 6 Part 2.

## Configuration Strategy: Simple MVP Approach

### Design Principles
After evaluating complex configuration options, the recommended approach uses a **simple, MVP configuration** that meets core goals:

1. **Kill Switch Capability**: Instant enable/disable without app updates
2. **Host Control**: Manage which hosts get pinned  
3. **Version Safety**: Ensure pinning only runs on tested app versions
4. **Operational Simplicity**: Easy to understand and manage

### Why Simple Over Complex

| Approach | Pros | Cons | Recommendation |
|----------|------|------|----------------|
| **Simple (MVP)** | Easy to understand<br/>Low complexity<br/>Reliable operation<br/>Fast implementation | Limited flexibility<br/>No gradual rollout | ✅ **Recommended** |
| **Complex** | Maximum flexibility<br/>Gradual rollout<br/>A/B testing | High complexity<br/>More failure points<br/>Security risks | ❌ Only if absolutely needed |

**Decision**: Use simple configuration for initial implementation. Complex features can be added later if business requirements justify the added complexity.

## Approved Configuration Schema

### Standard Configuration Structure
```json
{
  "ssl_pinning_config": {
    "enabled": true,
    "target_hosts": [
      "apigee.kreditplus.com",
      "apigee.kbfinansia.com"
    ],
    "min_app_version": "1.5.0"
  }
}
```

### Alternative Format (Comma-Separated)
```json
{
  "ssl_pinning_config": {
    "enabled": true,
    "target_hosts": "apigee.kreditplus.com,apigee.kbfinansia.com",
    "min_app_version": "1.5.0"
  }
}
```

### Configuration Parameters Reference

| Parameter | Type | Required | Default | Purpose | Business Logic |
|-----------|------|----------|---------|---------|----------------|
| `enabled` | Boolean | Yes | `false` | Master kill switch for all SSL pinning | **Emergency Control**: Set to `false` to instantly disable all pinning across all apps |
| `target_hosts` | Array or String | Yes | `[]` | Hostnames that should have SSL pinning enforced | **Selective Pinning**: Only listed hosts get certificate validation. Empty array = no pinning |
| `min_app_version` | String | Yes | `"1.0.0"` | Minimum app version required for pinning | **Safety Gate**: Prevents pinning on untested app versions |

### Application Logic Flow

#### 1. Master Switch Control (`enabled`)
```
Application Startup:
    fetch ssl_pinning_config from remote
    
    if (config.enabled == false) {
        → DISABLE ALL PINNING
        → Use normal TLS for all connections
        → Log: "SSL pinning disabled by remote config"
    }
    
    if (config.enabled == true) {
        → Continue to next validation steps
    }
```

#### 2. Version Safety Gate (`min_app_version`)
```
Version Check:
    currentVersion = getAppVersion()  // e.g., "1.6.0"
    requiredVersion = config.min_app_version  // e.g., "1.5.0"
    
    if (currentVersion < requiredVersion) {
        → DISABLE PINNING for this app
        → Use normal TLS for all connections  
        → Log: "App version too old for SSL pinning"
        → Return
    }
    
    → Continue to host validation
```

#### 3. Host-Level Control (`target_hosts`)
```
For Each Network Request:
    hostname = extractHostname(request.url)
    
    if (hostname NOT in config.target_hosts) {
        → Use NORMAL TLS VALIDATION
        → Standard CA certificate validation
    } else {
        → Use SSL PINNING VALIDATION  
        → Apply build-time certificate pins
        → Reject if pin doesn't match
    }
```

## Configuration Examples and Use Cases

### Kill Switch Scenarios

#### Emergency Kill Switch (Complete Disable)
```json
{
  "ssl_pinning_config": {
    "enabled": false,
    "target_hosts": [],
    "min_app_version": "1.0.0"
  }
}
```
**Use Case**: Certificate rotation went wrong, need immediate disable
**Result**: All SSL pinning disabled across all apps and hosts

#### Production Environment Only
```json
{
  "ssl_pinning_config": {
    "enabled": true,
    "target_hosts": ["apigee.kreditplus.com"],
    "min_app_version": "1.5.0"
  }
}
```
**Use Case**: Enable pinning for production API only
**Result**: 
- Production API (`apigee.kreditplus.com`) gets pinned
- Staging API (`apigee.kbfinansia.com`) uses normal TLS
- Platform API (`platform.kreditplus.com`) uses normal TLS

#### Staging Environment Only  
```json
{
  "ssl_pinning_config": {
    "enabled": true,
    "target_hosts": ["apigee.kbfinansia.com"],
    "min_app_version": "1.0.0"
  }
}
```
**Use Case**: Test pinning in staging before production
**Result**: Only staging environment gets pinned, production uses normal TLS

#### Multiple Environments
```json
{
  "ssl_pinning_config": {
    "enabled": true,
    "target_hosts": [
      "apigee.kreditplus.com",
      "apigee.kbfinansia.com"
    ],
    "min_app_version": "1.5.0"
  }
}
```
**Use Case**: Enable pinning for both production and staging
**Result**: Both environments get pinned, other hosts use normal TLS

### Version Control Scenarios

#### Block Old App Versions
```json
{
  "ssl_pinning_config": {
    "enabled": true,
    "target_hosts": ["apigee.kreditplus.com"],
    "min_app_version": "2.0.0"
  }
}
```
**Result**: Only apps version 2.0.0+ apply pinning, older versions use normal TLS

### Host Matching Examples

| Request URL | Target Hosts | Pinning Applied? | Explanation |
|-------------|-------------|------------------|-------------|
| `https://apigee.kreditplus.com/api/data` | `["apigee.kreditplus.com"]` | ✅ Yes | Exact match in target list |
| `https://platform.kreditplus.com/auth` | `["apigee.kreditplus.com"]` | ❌ No | Not in target list |
| `https://api.tokopedia.com/external` | `["apigee.kreditplus.com"]` | ❌ No | External host, not in list |
| `https://apigee.kbfinansia.com/test` | `["apigee.kreditplus.com", "apigee.kbfinansia.com"]` | ✅ Yes | Match found in target list |

## Configuration Distribution and Management

### Firebase Remote Config Implementation

#### Setup Requirements
- **Firebase Project**: Dedicated project or existing project with proper IAM
- **Environment Separation**: Separate configurations for staging/production
- **Access Control**: Role-based access for configuration updates
- **Backup Strategy**: Configuration version history and rollback capability

#### Distribution Timeline
- **Fetch Interval**: 1 hour for production, immediate for debug builds
- **Emergency Updates**: < 1 minute propagation globally
- **Standard Updates**: < 5 minutes propagation globally
- **Cache Strategy**: Local fallback with last-known-good configuration

#### Configuration Conditions (Firebase Console)
```json
{
  "conditions": [
    {
      "name": "Production Apps",
      "expression": "app.build == 'release' && app.version >= '1.5.0'",
      "parameters": {
        "ssl_pinning_config": {
          "enabled": true,
          "target_hosts": ["apigee.kreditplus.com"],
          "min_app_version": "1.5.0"
        }
      }
    },
    {
      "name": "Staging Apps", 
      "expression": "app.build == 'debug'",
      "parameters": {
        "ssl_pinning_config": {
          "enabled": true,
          "target_hosts": ["apigee.kbfinansia.com"],
          "min_app_version": "1.0.0"
        }
      }
    },
    {
      "name": "Emergency Kill Switch",
      "expression": "true",
      "parameters": {
        "ssl_pinning_config": {
          "enabled": false,
          "target_hosts": [],
          "min_app_version": "1.0.0"
        }
      }
    }
  ]
}
```

### Emergency Response Procedures

#### Kill Switch Activation (Emergency)

**Step 1: Immediate Response**
1. Access Firebase Remote Config console
2. Activate "Emergency Kill Switch" condition
3. Verify configuration shows: `"enabled": false`
4. Publish changes immediately

**Step 2: Verification**
```json
{
  "ssl_pinning_config": {
    "enabled": false,
    "target_hosts": [],
    "min_app_version": "1.0.0"
  }
}
```

**Step 3: Monitoring**
- Monitor app telemetry for pinning bypass confirmation
- Verify apps switch to normal TLS validation
- Document activation time and reason

#### Selective Host Disable

**Disable Specific Host Only:**
```json
{
  "ssl_pinning_config": {
    "enabled": true,
    "target_hosts": ["apigee.kbfinansia.com"],
    "min_app_version": "1.5.0"
  }
}
```
**Result**: Disables pinning for `apigee.kreditplus.com`, keeps staging pinned

#### Re-enablement After Issue Resolution

**Gradual Re-enablement:**
```json
// Step 1: Enable staging only
{
  "ssl_pinning_config": {
    "enabled": true,
    "target_hosts": ["apigee.kbfinansia.com"],
    "min_app_version": "1.5.0"
  }
}

// Step 2: Add production after validation
{
  "ssl_pinning_config": {
    "enabled": true,
    "target_hosts": [
      "apigee.kbfinansia.com",
      "apigee.kreditplus.com"
    ],
    "min_app_version": "1.5.0"
  }
}
```

### Operational Procedures

#### Configuration Change Process

**Standard Changes (Non-Emergency):**
1. **Test in Staging**: Update staging configuration first
2. **Validate**: Confirm staging apps behave correctly
3. **Production Update**: Apply same configuration to production
4. **Monitor**: Watch for 15 minutes post-deployment
5. **Document**: Record changes in configuration log

**Emergency Changes:**
1. **Immediate Update**: Apply kill switch without staging test
2. **Incident Response**: Follow security incident procedures
3. **Communication**: Notify security and development teams
4. **Post-Incident**: Document lessons learned

#### Configuration Validation

**Pre-Deployment Checks:**
- [ ] JSON syntax is valid
- [ ] All required parameters present
- [ ] Host lists contain valid hostnames
- [ ] Version strings follow semantic versioning
- [ ] Staging configuration tested successfully

**Post-Deployment Monitoring:**
- [ ] Configuration distributed to majority of apps within 5 minutes
- [ ] No increase in SSL/TLS connection failures
- [ ] App telemetry confirms expected behavior
- [ ] No security alerts triggered

### Configuration Security

#### Access Control
- **Firebase Console Access**: Limited to security and DevOps teams
- **Two-Factor Authentication**: Required for all configuration access
- **Audit Logging**: All changes logged with user and timestamp
- **Review Process**: Non-emergency changes require peer review

#### Configuration Integrity
- **Syntax Validation**: JSON validation before deployment
- **Schema Validation**: Ensure all required fields present
- **Value Validation**: Check hostnames and versions are valid
- **Rollback Testing**: Verify rollback capability monthly

#### Monitoring and Alerting

**Configuration Change Alerts:**
- Immediate notification when kill switch activated
- Daily summary of configuration changes
- Alert on unexpected configuration distribution failures

**Application Behavior Monitoring:**
- SSL pinning success/failure rates
- TLS handshake performance impact
- Certificate validation error rates
- App crash rates related to network connections

### Best Practices

#### Configuration Management
1. **Keep It Simple**: Use minimal configuration for easier operation
2. **Document Changes**: Maintain log of why changes were made
3. **Test First**: Always validate in staging before production
4. **Monitor Impact**: Watch for unintended consequences
5. **Have Rollback Plan**: Know how to quickly revert changes

#### Emergency Response
1. **Know the Process**: Train teams on kill switch activation
2. **Have Contacts Ready**: Maintain list of who to notify
3. **Document Everything**: Record timeline and decisions
4. **Learn from Incidents**: Update procedures based on experience

#### Version Management
1. **Conservative Minimums**: Don't enable pinning on untested versions
2. **Gradual Increases**: Raise minimum version gradually
3. **Support Older Apps**: Ensure graceful degradation for old versions
4. **Plan for EOL**: Have strategy for unsupported app versions

This configuration strategy provides the essential kill switch capability while maintaining operational simplicity and reliability.
