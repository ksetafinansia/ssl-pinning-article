# SSL Pinning iOS Swift Simple Implementation Guide

## Overview

This guide provides essential concepts and implementation patterns for SSL pinning in iOS applications using Firebase Remote Config. SSL pinning enhances security by validating server certificates against known public key hashes.

## Core Architecture

```
iOS App → Firebase Remote Config → SSL Pinning Logic → URLSession → Secure API Calls
```

## 1. Dependencies Setup

Add Firebase to your iOS project using CocoaPods. Add to your `Podfile`:

```ruby
# Podfile
platform :ios, '11.0'
use_frameworks!

target 'YourApp' do
  # Firebase
  pod 'Firebase/Core'
  pod 'Firebase/RemoteConfig'
  
  # Networking (if using Alamofire)
  pod 'Alamofire', '~> 5.8'
  
  # JSON parsing (if needed)
  pod 'SwiftyJSON', '~> 5.0'
end
```

Or using Swift Package Manager:
```
https://github.com/firebase/firebase-ios-sdk
```

## 2. Configuration Payload Structure

Firebase Remote Config stores SSL pinning configurations with this JSON structure:

```json
{
  "configurations": [
    {
      "host": "apigee.kreditplus.com",
      "is_android_enable": true,
      "is_ios_enable": true,
      "pins": {
        "primary": "encrypted_hash_1",
        "backup": "encrypted_hash_2", 
        "emergency": "encrypted_hash_3"
      }
    }
  ]
}
```

**Note**: iOS implementation checks the `is_ios_enable` flag for platform-specific control.

## 3. Core Implementation Components

### A. Data Models

```swift
struct RemoteSSLConfig: Codable {
    let host: String
    let isAndroidEnable: Bool? // Not used in iOS logic
    let isIosEnable: Bool
    let pins: SSLPins
    
    enum CodingKeys: String, CodingKey {
        case host
        case isAndroidEnable = "is_android_enable"
        case isIosEnable = "is_ios_enable"
        case pins
    }
}

struct SSLPins: Codable {
    let primary: String
    let backup: String
    let emergency: String
    
    var allHashes: [String] {
        return [primary, backup, emergency]
    }
}

struct RemoteConfigResponse: Codable {
    let configurations: [RemoteSSLConfig]
}
```

### B. SSL Pinning Manager

```swift
class SSLPinningManager {
    static let shared = SSLPinningManager()
    
    private let remoteConfigKey = "ssl_pinning_configurations"
    private let configCacheKey = "ssl_pinning_remote_config"
    
    private var configurations: [RemoteSSLConfig] = []
    private var remoteConfig: RemoteConfig!
    
    private init() {}
    
    // Key methods to implement:
    // - initialize() - Setup Firebase and load config
    // - setupFirebaseRemoteConfig() - Configure Firebase Remote Config
    // - fetchFirebaseRemoteConfiguration() - Fetch from Firebase
    // - getConfigForHost(String host) - Get config for specific host
    // - isEnabledForHost(String host) - Check if pinning enabled (checks is_ios_enable)
    // - createSecureURLSession() - Create URLSession with SSL pinning
}
```

## 4. Implementation Steps

### Step 1: Firebase Setup

1. Add Firebase to your iOS project
2. Configure Firebase Remote Config with default values
3. Set fetch intervals and timeout settings

```swift
private func setupFirebaseRemoteConfig() {
    remoteConfig = RemoteConfig.remoteConfig()
    
    let settings = RemoteConfigSettings()
    settings.minimumFetchInterval = 3600 // 1 hour
    settings.fetchTimeout = 60
    remoteConfig.configSettings = settings
    
    // Set default values
    let defaultConfig = getDefaultConfigJSON()
    remoteConfig.setDefaults([remoteConfigKey: defaultConfig as NSObject])
}
```

### Step 2: Certificate Validation Logic

iOS implementation checks the `is_ios_enable` flag:

```swift
func isEnabledForHost(_ host: String) -> Bool {
    guard let config = getConfigForHost(host) else { return false }
    return config.isIosEnable
}
```

### Step 3: URLSession Integration

Create secure URLSession with certificate pinning:

```swift
func createSecureURLSession() -> URLSession {
    let configuration = URLSessionConfiguration.default
    configuration.timeoutIntervalForRequest = 30
    configuration.timeoutIntervalForResource = 60
    
    let session = URLSession(
        configuration: configuration,
        delegate: self,
        delegateQueue: nil
    )
    
    return session
}

// Implement URLSessionDelegate for certificate pinning
extension SSLPinningManager: URLSessionDelegate {
    func urlSession(_ session: URLSession, 
                   didReceive challenge: URLAuthenticationChallenge, 
                   completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) {
        
        guard let hostname = challenge.protectionSpace.host,
              isEnabledForHost(hostname) else {
            // Use default handling for non-pinned hosts
            completionHandler(.performDefaultHandling, nil)
            return
        }
        
        // Perform certificate validation
        if validateCertificate(challenge: challenge, hostname: hostname) {
            completionHandler(.useCredential, URLCredential(trust: challenge.protectionSpace.serverTrust!))
        } else {
            completionHandler(.rejectProtectionSpace, nil)
        }
    }
}
```

### Step 4: Alamofire Integration (Optional)

If using Alamofire, create a custom ServerTrustManager:

```swift
func createSecureAlamofireSession() -> Session {
    let serverTrustManager = ServerTrustManager(
        allHostsMustBeEvaluated: false,
        evaluators: createServerTrustEvaluators()
    )
    
    let configuration = URLSessionConfiguration.default
    configuration.timeoutIntervalForRequest = 30
    
    return Session(
        configuration: configuration,
        serverTrustManager: serverTrustManager
    )
}

private func createServerTrustEvaluators() -> [String: ServerTrustEvaluating] {
    var evaluators: [String: ServerTrustEvaluating] = [:]
    
    configurations.forEach { config in
        if config.isIosEnable {
            let decryptedHashes = decryptPins(config.pins.allHashes)
            let publicKeys = decryptedHashes.compactMap { convertHashToPublicKey($0) }
            
            evaluators[config.host] = PublicKeysTrustEvaluator(
                keys: publicKeys,
                performDefaultValidation: true,
                validateHost: true
            )
        }
    }
    
    return evaluators
}
```

## 5. Key Security Considerations

### A. Hash Management
- Store encrypted pin hashes in Firebase Remote Config
- Implement secure decryption logic using iOS Keychain
- Use multiple backup hashes (primary, backup, emergency)

### B. Kill Switch
- Implement remote kill switch through `is_enabled` flag
- Cache configurations locally using UserDefaults or Keychain
- Provide fallback configurations for offline scenarios

### C. Error Handling
- Distinguish SSL pinning errors from network errors
- Implement retry mechanisms with exponential backoff
- Log security events appropriately (without sensitive data)

## 6. Configuration Management

### Local Caching
```swift
private func cacheConfiguration(_ configJSON: String) {
    UserDefaults.standard.set(configJSON, forKey: configCacheKey)
    UserDefaults.standard.set(Date(), forKey: "\(configCacheKey)_timestamp")
}

private func loadCachedConfiguration() -> RemoteConfigResponse? {
    guard let cachedJSON = UserDefaults.standard.string(forKey: configCacheKey),
          let data = cachedJSON.data(using: .utf8) else {
        return nil
    }
    
    return try? JSONDecoder().decode(RemoteConfigResponse.self, from: data)
}
```

### Fallback Configuration
```swift
private func loadFallbackConfiguration() {
    configurations = [
        RemoteSSLConfig(
            host: "apigee.kreditplus.com",
            isAndroidEnable: true,
            isIosEnable: true,
            pins: SSLPins(
                primary: "sha256/[FALLBACK_HASH_1]",
                backup: "sha256/[FALLBACK_HASH_2]",
                emergency: "sha256/[FALLBACK_HASH_3]"
            )
        )
    ]
}
```

## 7. Testing Strategy

### A. Unit Tests
- Test configuration parsing and validation
- Test hash decryption logic
- Test enable/disable logic (only `is_ios_enable` flag)

### B. Integration Tests
- Test Firebase Remote Config integration
- Test certificate validation with real certificates
- Test offline scenarios with cached configuration

### C. Security Tests
- Test with invalid certificates
- Test kill switch functionality
- Test network failure scenarios

## 8. Error Handling Patterns

### SSL Pinning Exceptions
```swift
enum SSLPinningError: Error, LocalizedError {
    case certificateValidationFailed(hostname: String)
    case configurationLoadFailed(Error)
    case networkError(String)
    case invalidConfiguration
    
    var errorDescription: String? {
        switch self {
        case .certificateValidationFailed(let hostname):
            return "Certificate validation failed for \(hostname)"
        case .configurationLoadFailed(let error):
            return "Failed to load SSL pinning configuration: \(error.localizedDescription)"
        case .networkError(let message):
            return "Network error: \(message)"
        case .invalidConfiguration:
            return "Invalid SSL pinning configuration"
        }
    }
}
```

### User-Friendly Error Handling
```swift
func handleSSLPinningError(_ error: Error) -> String {
    if let sslError = error as? SSLPinningError {
        switch sslError {
        case .certificateValidationFailed:
            return "Security error: Certificate validation failed. Please check your connection."
        case .networkError:
            return "Network error: Please check your internet connection and try again."
        default:
            return "An unexpected security error occurred. Please try again later."
        }
    }
    return "An unexpected error occurred. Please try again later."
}
```

## 9. Best Practices

### A. Configuration Management
- Use Firebase Remote Config for centralized management
- Implement local caching with UserDefaults or Keychain
- Set appropriate fetch intervals (recommended: 1 hour)

### B. Certificate Management
- Pin to public key hashes, not certificate hashes
- Maintain multiple backup pins for redundancy
- Plan for certificate rotation scenarios

### C. Performance Optimization
- Cache validation results to avoid repeated computations
- Use URLSession connection pooling
- Minimize Firebase Remote Config fetch frequency

### D. Security Best Practices
- Never log sensitive certificate or pin data
- Use iOS Keychain for storing sensitive configuration
- Implement certificate transparency validation if needed

## 10. Production Considerations

### A. Monitoring and Logging
```swift
class SSLPinningLogger {
    static let shared = SSLPinningLogger()
    
    func logSSLPinningEnabled(hostname: String) {
        os_log("SSL Pinning enabled for %@", log: .default, type: .info, hostname)
    }
    
    func logCertificateValidationSuccess(hostname: String) {
        os_log("Certificate validation successful for %@", log: .default, type: .debug, hostname)
    }
    
    func logCertificateValidationFailure(hostname: String) {
        os_log("Certificate validation failed for %@", log: .default, type: .error, hostname)
    }
}
```

### B. Gradual Rollout
- Start with kill switch enabled (`is_ios_enable: false`)
- Gradually enable for user segments using Firebase Remote Config conditions
- Monitor crash reports and error rates

### C. Emergency Response
- Implement quick kill switch activation through Firebase console
- Monitor certificate validation failure rates
- Set up alerts for unusual SSL pinning patterns

## Implementation Checklist

- [ ] Firebase project setup and Remote Config enabled
- [ ] SSL pinning data models implemented
- [ ] Firebase Remote Config integration completed
- [ ] Certificate validation logic implemented (checks only `is_ios_enable`)
- [ ] URLSession/Alamofire integration with certificate pinning
- [ ] Local caching and offline support (UserDefaults/Keychain)
- [ ] Fallback configuration for network failures
- [ ] Error handling and user-friendly messages
- [ ] Unit and integration tests
- [ ] Security testing with invalid certificates
- [ ] Production monitoring and logging setup
- [ ] App Store submission considerations

## Common Pitfalls to Avoid

1. **Hardcoding Certificates**: Always use Firebase Remote Config
2. **Platform Flag Confusion**: iOS only checks `is_ios_enable` flag
3. **Poor Error Handling**: Distinguish between security and network errors
4. **No Offline Support**: Always implement local caching and fallback
5. **Insufficient Testing**: Test with various certificate and network scenarios
6. **Logging Sensitive Data**: Never log certificate hashes or pin data
7. **No Kill Switch**: Always implement remote disable capability
8. **Blocking Main Thread**: Use background queues for network and configuration operations
9. **App Store Rejection**: Ensure SSL pinning doesn't interfere with App Store review process
10. **Certificate Expiry**: Plan for certificate rotation and renewal

## App Store Considerations

- Ensure SSL pinning doesn't prevent App Store review process
- Consider implementing reviewer bypass mechanism
- Document SSL pinning implementation for App Store review
- Test thoroughly on different network configurations

This guide provides the foundation for implementing robust SSL pinning in iOS applications using Firebase Remote Config while maintaining security, flexibility, and App Store compliance.
