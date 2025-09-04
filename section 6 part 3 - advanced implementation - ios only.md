# Section 6 Part 3: Advanced Implementation - Gradual Rollout with Remote Pin Storage (Appendix)

## Overview
This appendix demonstrates an advanced SSL pinning implementation that includes gradual rollout capabilities and remote pin storage using Firebase Remote Config. This approach provides maximum flexibility but comes with increased complexity and security considerations.

**⚠️ Warning**: This implementation stores certificate pins in remote configuration, which may have security implications. Use only if your security requirements allow remote pin management and you have proper security controls in place.

## When to Use This Approach

### Use Cases
- **Dynamic Pin Rotation**: Need to update pins without app releases
- **Emergency Pin Updates**: Certificate issues require immediate pin changes
- **A/B Testing**: Test new pins with subset of users before full rollout
- **Gradual Migrations**: Slowly migrate from old to new certificates

### Security Considerations
- **Remote Pin Storage**: Pins are stored in Firebase, increasing attack surface
- **Network Dependency**: Pin updates require network connectivity
- **Cache Invalidation**: Stale pin caches could cause failures
- **Key Management**: Firebase access controls become critical

## Enhanced Configuration Schema

### Complex Configuration Structure
```json
{
  "ssl_pinning_config": {
    "enabled": true,
    "rollout_percentage": 25,
    "min_app_version": "1.5.0",
    "hosts": {
      "apigee.kreditplus.com": {
        "enabled": true,
        "pins": {
          "primary": "sha256/AAAB3NzaC1yc2EAAAA...",
          "backup": "sha256/BBBF4OzaC1yc2EAAAA...",
          "emergency": "sha256/CCCG5PzaC1yc2EAAAA..."
        },
        "rollout_percentage": 50
      },
      "apigee.kbfinansia.com": {
        "enabled": true,
        "pins": {
          "primary": "sha256/DDDE6QzaC1yc2EAAAA...",
          "backup": "sha256/EEEF7RzaC1yc2EAAAA..."
        },
        "rollout_percentage": 100
      }
    },
    "rollout_strategy": {
      "method": "percentage",
      "seed": "device_id",
      "sticky": true
    },
    "fallback": {
      "use_builtin_pins": true,
      "builtin_pin_hosts": ["apigee.kreditplus.com"]
    },
    "monitoring": {
      "enabled": true,
      "failure_threshold": 5.0,
      "auto_disable": true
    }
  }
}
```

### Configuration Parameters Reference

| Parameter | Type | Description | Logic Impact |
|-----------|------|-------------|--------------|
| `enabled` | Boolean | Global SSL pinning toggle | Master kill switch for all pinning |
| `rollout_percentage` | Integer (0-100) | Global rollout percentage | Controls what % of users get pinning |
| `min_app_version` | String | Minimum app version for pinning | Version gate for safety |
| `hosts[hostname].enabled` | Boolean | Per-host pinning toggle | Individual host control |
| `hosts[hostname].pins` | Object | Remote pin storage | Pins fetched from remote config |
| `hosts[hostname].rollout_percentage` | Integer (0-100) | Per-host rollout control | Host-specific rollout |
| `rollout_strategy.method` | String | How rollout is calculated | "percentage", "whitelist", "geography" |
| `rollout_strategy.seed` | String | Randomization seed | "device_id", "user_id", "install_id" |
| `rollout_strategy.sticky` | Boolean | Consistent user experience | Same user always gets same rollout result |
| `fallback.use_builtin_pins` | Boolean | Fallback to build-time pins | Safety net if remote fails |
| `monitoring.failure_threshold` | Float | Auto-disable threshold (%) | Automatically disable if failure rate > threshold |

## Swift Implementation Example

### Advanced Remote Config Manager
```swift
import Firebase
import Foundation
import CryptoKit

struct AdvancedSSLPinningConfig: Codable {
    let enabled: Bool
    let rolloutPercentage: Int
    let minAppVersion: String
    let hosts: [String: HostConfig]
    let rolloutStrategy: RolloutStrategy
    let fallback: FallbackConfig
    let monitoring: MonitoringConfig
    
    struct HostConfig: Codable {
        let enabled: Bool
        let pins: PinConfig
        let rolloutPercentage: Int
        
        struct PinConfig: Codable {
            let primary: String
            let backup: String
            let emergency: String?
        }
    }
    
    struct RolloutStrategy: Codable {
        let method: String
        let seed: String
        let sticky: Bool
    }
    
    struct FallbackConfig: Codable {
        let useBuiltinPins: Bool
        let builtinPinHosts: [String]
        
        enum CodingKeys: String, CodingKey {
            case useBuiltinPins = "use_builtin_pins"
            case builtinPinHosts = "builtin_pin_hosts"
        }
    }
    
    struct MonitoringConfig: Codable {
        let enabled: Bool
        let failureThreshold: Double
        let autoDisable: Bool
        
        enum CodingKeys: String, CodingKey {
            case enabled
            case failureThreshold = "failure_threshold"
            case autoDisable = "auto_disable"
        }
    }
    
    enum CodingKeys: String, CodingKey {
        case enabled
        case rolloutPercentage = "rollout_percentage"
        case minAppVersion = "min_app_version"
        case hosts
        case rolloutStrategy = "rollout_strategy"
        case fallback
        case monitoring
    }
}

class AdvancedRemoteConfigManager {
    private let remoteConfig = RemoteConfig.remoteConfig()
    private let sslConfigKey = "ssl_pinning_config"
    private let analytics = Analytics.self
    
    // Build-time fallback pins (from Part 1)
    private let builtinPins: [String: [String]] = [
        "apigee.kreditplus.com": [
            "sha256/builtin_primary_pin_hash",
            "sha256/builtin_backup_pin_hash"
        ],
        "apigee.kbfinansia.com": [
            "sha256/builtin_staging_pin_hash"
        ]
    ]
    
    init() {
        setupRemoteConfig()
    }
    
    private func setupRemoteConfig() {
        let settings = RemoteConfigSettings()
        settings.minimumFetchInterval = 300 // 5 minutes for faster updates
        #if DEBUG
        settings.minimumFetchInterval = 0
        #endif
        remoteConfig.configSettings = settings
        
        setDefaults()
    }
    
    func fetchAndActivate() async -> Bool {
        do {
            let status = try await remoteConfig.fetchAndActivate()
            return status == .successFetchedFromRemote || status == .successUsingPreFetchedData
        } catch {
            print("Remote config fetch failed: \(error)")
            return false
        }
    }
    
    func getSSLPinningConfig() -> AdvancedSSLPinningConfig {
        let configString = remoteConfig[sslConfigKey].stringValue ?? ""
        
        do {
            let configData = configString.data(using: .utf8) ?? Data()
            let config = try JSONDecoder().decode(AdvancedSSLPinningConfig.self, from: configData)
            return config
        } catch {
            print("Failed to decode advanced SSL pinning config: \(error)")
            return getDefaultConfig()
        }
    }
    
    func shouldEnablePinning() -> Bool {
        let config = getSSLPinningConfig()
        
        // Global enable check
        guard config.enabled else {
            reportTelemetry(event: "pinning_disabled_global")
            return false
        }
        
        // Version gate
        guard isAppVersionSupported(minVersion: config.minAppVersion) else {
            reportTelemetry(event: "pinning_disabled_version")
            return false
        }
        
        // Global rollout check
        guard isInRollout(percentage: config.rolloutPercentage, strategy: config.rolloutStrategy) else {
            reportTelemetry(event: "pinning_disabled_rollout_global")
            return false
        }
        
        reportTelemetry(event: "pinning_enabled_global")
        return true
    }
    
    func shouldPinHost(_ hostname: String) -> (shouldPin: Bool, pins: [String]) {
        guard shouldEnablePinning() else {
            return (false, [])
        }
        
        let config = getSSLPinningConfig()
        
        // Check if host is configured
        guard let hostConfig = config.hosts[hostname], hostConfig.enabled else {
            reportTelemetry(event: "pinning_disabled_host_not_configured", parameters: ["hostname": hostname])
            return checkFallbackPins(hostname: hostname, config: config)
        }
        
        // Host-specific rollout check
        guard isInRollout(percentage: hostConfig.rolloutPercentage, strategy: config.rolloutStrategy) else {
            reportTelemetry(event: "pinning_disabled_rollout_host", parameters: ["hostname": hostname])
            return checkFallbackPins(hostname: hostname, config: config)
        }
        
        // Extract remote pins
        var pins: [String] = []
        pins.append(hostConfig.pins.primary)
        pins.append(hostConfig.pins.backup)
        if let emergency = hostConfig.pins.emergency {
            pins.append(emergency)
        }
        
        // Remove sha256/ prefix if present for internal processing
        let cleanPins = pins.map { pin in
            pin.hasPrefix("sha256/") ? String(pin.dropFirst(7)) : pin
        }
        
        reportTelemetry(event: "pinning_enabled_host_remote", parameters: [
            "hostname": hostname,
            "pin_count": cleanPins.count
        ])
        
        return (true, cleanPins)
    }
    
    private func checkFallbackPins(hostname: String, config: AdvancedSSLPinningConfig) -> (shouldPin: Bool, pins: [String]) {
        guard config.fallback.useBuiltinPins,
              config.fallback.builtinPinHosts.contains(hostname),
              let fallbackPins = builtinPins[hostname] else {
            return (false, [])
        }
        
        reportTelemetry(event: "pinning_enabled_host_fallback", parameters: ["hostname": hostname])
        return (true, fallbackPins)
    }
    
    private func isInRollout(percentage: Int, strategy: AdvancedSSLPinningConfig.RolloutStrategy) -> Bool {
        if percentage >= 100 { return true }
        if percentage <= 0 { return false }
        
        let seed = getSeedValue(for: strategy.seed)
        let hash = calculateStableHash(seed: seed, sticky: strategy.sticky)
        let userPercentile = hash % 100
        
        return userPercentile < percentage
    }
    
    private func getSeedValue(for seedType: String) -> String {
        switch seedType {
        case "device_id":
            return UIDevice.current.identifierForVendor?.uuidString ?? "unknown"
        case "user_id":
            // Return user ID if available from your auth system
            return getCurrentUserId() ?? "anonymous"
        case "install_id":
            // Return installation ID
            return getInstallationId()
        default:
            return "default_seed"
        }
    }
    
    private func calculateStableHash(seed: String, sticky: Bool) -> Int {
        if sticky {
            // Use persistent hash for consistent user experience
            let key = "ssl_pinning_rollout_hash_\(seed)"
            if let stored = UserDefaults.standard.object(forKey: key) as? Int {
                return stored
            }
            
            let hash = abs(seed.hashValue) % 100
            UserDefaults.standard.set(hash, forKey: key)
            return hash
        } else {
            // Use session-based hash
            return abs(seed.hashValue) % 100
        }
    }
    
    private func isAppVersionSupported(minVersion: String) -> Bool {
        guard let currentVersion = Bundle.main.infoDictionary?["CFBundleShortVersionString"] as? String else {
            return false
        }
        return compareVersions(currentVersion, minVersion) >= 0
    }
    
    private func compareVersions(_ version1: String, _ version2: String) -> Int {
        return version1.compare(version2, options: .numeric)
    }
    
    private func getCurrentUserId() -> String? {
        // Implement based on your authentication system
        return nil
    }
    
    private fun getInstallationId() -> String {
        let key = "ssl_pinning_installation_id"
        if let stored = UserDefaults.standard.string(forKey: key) {
            return stored
        }
        
        let newId = UUID().uuidString
        UserDefaults.standard.set(newId, forKey: key)
        return newId
    }
    
    private func setDefaults() {
        let defaultConfig = getDefaultConfig()
        if let data = try? JSONEncoder().encode(defaultConfig),
           let jsonString = String(data: data, encoding: .utf8) {
            remoteConfig.setDefaults([sslConfigKey: jsonString as NSObject])
        }
    }
    
    private func getDefaultConfig() -> AdvancedSSLPinningConfig {
        return AdvancedSSLPinningConfig(
            enabled: false,
            rolloutPercentage: 0,
            minAppVersion: "1.0.0",
            hosts: [:],
            rolloutStrategy: AdvancedSSLPinningConfig.RolloutStrategy(
                method: "percentage",
                seed: "device_id",
                sticky: true
            ),
            fallback: AdvancedSSLPinningConfig.FallbackConfig(
                useBuiltinPins: true,
                builtinPinHosts: ["apigee.kreditplus.com"]
            ),
            monitoring: AdvancedSSLPinningConfig.MonitoringConfig(
                enabled: true,
                failureThreshold: 5.0,
                autoDisable: true
            )
        )
    }
    
    private func reportTelemetry(event: String, parameters: [String: Any] = [:]) {
        analytics.logEvent(event, parameters: parameters)
    }
}
```

### Advanced Certificate Pinner
```swift
class AdvancedCertificatePinner: NSObject {
    private let remoteConfigManager = AdvancedRemoteConfigManager()
    private var monitoringData: [String: MonitoringData] = [:]
    
    struct MonitoringData {
        var successCount: Int = 0
        var failureCount: Int = 0
        var lastUpdate: Date = Date()
        
        var failureRate: Double {
            let total = successCount + failureCount
            return total > 0 ? Double(failureCount) / Double(total) * 100.0 : 0.0
        }
    }
    
    override init() {
        super.init()
        Task {
            await updateConfiguration()
        }
    }
    
    func createURLSession() async -> URLSession {
        await updateConfiguration()
        
        let configuration = URLSessionConfiguration.default
        return URLSession(
            configuration: configuration,
            delegate: self,
            delegateQueue: nil
        )
    }
    
    private func updateConfiguration() async {
        let _ = await remoteConfigManager.fetchAndActivate()
    }
    
    private func updateMonitoring(hostname: String, success: Bool) {
        if monitoringData[hostname] == nil {
            monitoringData[hostname] = MonitoringData()
        }
        
        if success {
            monitoringData[hostname]?.successCount += 1
        } else {
            monitoringData[hostname]?.failureCount += 1
        }
        
        monitoringData[hostname]?.lastUpdate = Date()
        
        // Check auto-disable threshold
        let config = remoteConfigManager.getSSLPinningConfig()
        if config.monitoring.enabled && config.monitoring.autoDisable {
            let failureRate = monitoringData[hostname]?.failureRate ?? 0.0
            if failureRate > config.monitoring.failureThreshold {
                // Log critical failure rate
                Analytics.logEvent("ssl_pinning_auto_disable_triggered", parameters: [
                    "hostname": hostname,
                    "failure_rate": failureRate,
                    "threshold": config.monitoring.failureThreshold
                ])
            }
        }
    }
}

extension AdvancedCertificatePinner: URLSessionDelegate {
    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        
        let hostname = challenge.protectionSpace.host
        let (shouldPin, pins) = remoteConfigManager.shouldPinHost(hostname)
        
        guard shouldPin else {
            // Not pinning this host - use default validation
            completionHandler(.performDefaultHandling, nil)
            return
        }
        
        // Perform advanced pinning validation
        performAdvancedPinningValidation(
            challenge: challenge,
            hostname: hostname,
            expectedPins: pins,
            completionHandler: completionHandler
        )
    }
    
    private func performAdvancedPinningValidation(
        challenge: URLAuthenticationChallenge,
        hostname: String,
        expectedPins: [String],
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard let serverTrust = challenge.protectionSpace.serverTrust else {
            updateMonitoring(hostname: hostname, success: false)
            Analytics.logEvent("ssl_pinning_failure", parameters: [
                "hostname": hostname,
                "reason": "no_server_trust"
            ])
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        // Extract server public key
        guard let serverCert = SecTrustGetCertificateAtIndex(serverTrust, 0),
              let serverPublicKey = SecCertificateCopyKey(serverCert),
              let serverPublicKeyData = SecKeyCopyExternalRepresentation(serverPublicKey, nil) else {
            updateMonitoring(hostname: hostname, success: false)
            Analytics.logEvent("ssl_pinning_failure", parameters: [
                "hostname": hostname,
                "reason": "key_extraction_failed"
            ])
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        // Calculate server key hash
        let serverKeyHash = sha256(data: serverPublicKeyData as Data)
        
        // Check against expected pins
        let pinMatched = expectedPins.contains(serverKeyHash)
        
        if pinMatched {
            updateMonitoring(hostname: hostname, success: true)
            Analytics.logEvent("ssl_pinning_success", parameters: [
                "hostname": hostname,
                "pin_source": expectedPins.count > 2 ? "remote" : "builtin"
            ])
            let credential = URLCredential(trust: serverTrust)
            completionHandler(.useCredential, credential)
        } else {
            updateMonitoring(hostname: hostname, success: false)
            Analytics.logEvent("ssl_pinning_failure", parameters: [
                "hostname": hostname,
                "reason": "pin_mismatch",
                "server_hash": serverKeyHash,
                "expected_pins": expectedPins.count
            ])
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
    
    private func sha256(data: Data) -> String {
        let digest = SHA256.hash(data: data)
        return Data(digest).base64EncodedString()
    }
}
```

## Configuration Examples and Rollout Scenarios

### Scenario 1: Conservative Rollout (25% of users)
```json
{
  "ssl_pinning_config": {
    "enabled": true,
    "rollout_percentage": 25,
    "hosts": {
      "apigee.kreditplus.com": {
        "enabled": true,
        "pins": {
          "primary": "sha256/newPrimaryPin...",
          "backup": "sha256/newBackupPin..."
        },
        "rollout_percentage": 100
      }
    },
    "fallback": {
      "use_builtin_pins": true,
      "builtin_pin_hosts": ["apigee.kreditplus.com"]
    }
  }
}
```
**Result**: 25% of users get new remote pins, 75% use builtin pins

### Scenario 2: Emergency Pin Update (100% immediate)
```json
{
  "ssl_pinning_config": {
    "enabled": true,
    "rollout_percentage": 100,
    "hosts": {
      "apigee.kreditplus.com": {
        "enabled": true,
        "pins": {
          "primary": "sha256/emergencyPin...",
          "backup": "sha256/backupPin...",
          "emergency": "sha256/temporaryPin..."
        },
        "rollout_percentage": 100
      }
    },
    "fallback": {
      "use_builtin_pins": false
    }
  }
}
```
**Result**: All users immediately get new emergency pins

### Scenario 3: A/B Testing Different Pins
```json
{
  "ssl_pinning_config": {
    "enabled": true,
    "rollout_percentage": 100,
    "hosts": {
      "apigee.kreditplus.com": {
        "enabled": true,
        "pins": {
          "primary": "sha256/newAlgorithmPin...",
          "backup": "sha256/standardPin..."
        },
        "rollout_percentage": 50
      }
    },
    "rollout_strategy": {
      "method": "percentage",
      "seed": "user_id",
      "sticky": true
    },
    "fallback": {
      "use_builtin_pins": true,
      "builtin_pin_hosts": ["apigee.kreditplus.com"]
    }
  }
}
```
**Result**: 50% get new pins consistently, 50% use builtin pins

## Trade-offs and Considerations

### Advantages
- **Maximum Flexibility**: Can update pins without app releases
- **Gradual Testing**: Test changes with small user groups
- **Emergency Response**: Instant pin updates for certificate issues
- **Data-Driven**: Monitor rollout success rates

### Disadvantages
- **Security Risk**: Pins stored remotely increase attack surface
- **Complexity**: Much more complex code and configuration
- **Network Dependency**: Requires connectivity for pin updates
- **Debugging**: Harder to debug with dynamic pin sources

### When NOT to Use
- **High Security Requirements**: If pins must never leave the app
- **Simple Deployments**: Basic kill switch (Part 2) is sufficient
- **Limited Resources**: Team cannot manage complex rollout logic
- **Regulatory Constraints**: Compliance requires embedded pins

This advanced implementation provides maximum control and flexibility but should only be used when the additional complexity and security trade-offs are justified by your specific requirements.
