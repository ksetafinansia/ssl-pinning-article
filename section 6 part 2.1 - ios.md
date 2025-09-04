# iOS Kill Switch Implementation (Swift)

This guide shows how to implement Firebase Remote Config kill switch for SSL pinning in iOS applications using Swift and URLSession.

## Prerequisites
- Existing SSL pinning implementation from **Section 6 Part 1**
- Firebase project with Remote Config enabled
- Firebase iOS SDK integrated

## Step 1: Firebase Remote Config Manager

### Remote Config Manager
```swift
// RemoteConfigManager.swift
import Firebase
import Foundation

struct SSLPinningConfig: Codable {
    let enabled: Bool
    let targetHosts: [String]
    let minAppVersion: String
    
    enum CodingKeys: String, CodingKey {
        case enabled
        case targetHosts = "target_hosts"
        case minAppVersion = "min_app_version"
    }
}

class RemoteConfigManager {
    private let remoteConfig = RemoteConfig.remoteConfig()
    private let sslConfigKey = "ssl_pinning_config"
    
    init() {
        let settings = RemoteConfigSettings()
        settings.minimumFetchInterval = 3600 // 1 hour for production
        #if DEBUG
        settings.minimumFetchInterval = 0
        #endif
        remoteConfig.configSettings = settings
        
        // Set defaults
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
    
    func getSSLPinningConfig() -> SSLPinningConfig {
        let configString = remoteConfig[sslConfigKey].stringValue ?? ""
        
        do {
            let configData = configString.data(using: .utf8) ?? Data()
            var config = try JSONDecoder().decode(SSLPinningConfig.self, from: configData)
            
            // Handle comma-separated string format
            if config.targetHosts.isEmpty && configString.contains("target_hosts") {
                if let jsonObject = try JSONSerialization.jsonObject(with: configData) as? [String: Any],
                   let hostsString = jsonObject["target_hosts"] as? String {
                    let hosts = hostsString.split(separator: ",").map { $0.trimmingCharacters(in: .whitespaces) }
                    config = SSLPinningConfig(
                        enabled: config.enabled,
                        targetHosts: hosts,
                        minAppVersion: config.minAppVersion
                    )
                }
            }
            
            return config
        } catch {
            print("Failed to decode SSL pinning config: \(error)")
            return getDefaultConfig()
        }
    }
    
    func shouldEnablePinning() -> Bool {
        let config = getSSLPinningConfig()
        
        // Global enable check
        if !config.enabled {
            return false
        }
        
        // App version check
        if !isAppVersionSupported(minVersion: config.minAppVersion) {
            return false
        }
        
        // Check if we have any target hosts
        if config.targetHosts.isEmpty {
            return false
        }
        
        return true
    }
    
    func shouldPinHost(_ hostname: String) -> Bool {
        if !shouldEnablePinning() { return false }
        
        let config = getSSLPinningConfig()
        return config.targetHosts.contains(hostname)
    }
    
    private func setDefaults() {
        let defaultConfig = SSLPinningConfig(
            enabled: false,
            targetHosts: [],
            minAppVersion: "1.0.0"
        )
        
        if let data = try? JSONEncoder().encode(defaultConfig),
           let jsonString = String(data: data, encoding: .utf8) {
            remoteConfig.setDefaults([sslConfigKey: jsonString as NSObject])
        }
    }
    
    private func getDefaultConfig() -> SSLPinningConfig {
        return SSLPinningConfig(
            enabled: false,
            targetHosts: [],
            minAppVersion: "1.0.0"
        )
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
}
```

## Step 2: Updated Certificate Pinner with Remote Config

### Enhanced Certificate Pinner
```swift
// UpdatedCertificatePinner.swift
import Foundation
import CommonCrypto

class CertificatePinner: NSObject {
    private let remoteConfigManager = RemoteConfigManager()
    private var shouldPin = false
    
    // Build-time pins (from Section 6 Part 1)
    private let buildTimePins: [String: [String]] = [
        "apigee.kreditplus.com": ["your_build_time_primary_pin", "your_build_time_backup_pin"],
        "apigee.kbfinansia.com": ["your_staging_pin"]
    ]
    
    override init() {
        super.init()
        Task {
            await updateConfiguration()
        }
    }
    
    func createURLSession() async -> URLSession {
        await updateConfiguration()
        
        let configuration = URLSessionConfiguration.default
        configuration.timeoutIntervalForRequest = 30
        configuration.timeoutIntervalForResource = 30
        
        return URLSession(
            configuration: configuration,
            delegate: self,
            delegateQueue: nil
        )
    }
    
    private func updateConfiguration() async {
        let _ = await remoteConfigManager.fetchAndActivate()
        shouldPin = remoteConfigManager.shouldEnablePinning()
    }
    
    // Force refresh configuration
    func refreshConfiguration() async {
        await updateConfiguration()
    }
}

extension CertificatePinner: URLSessionDelegate {
    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        
        let hostname = challenge.protectionSpace.host
        
        // Check if this host should be pinned according to remote config
        if !remoteConfigManager.shouldPinHost(hostname) {
            // Not in target host list or pinning disabled, use default handling
            completionHandler(.performDefaultHandling, nil)
            return
        }
        
        // Perform pinning validation using build-time pins
        performPinningValidation(
            challenge: challenge, 
            hostname: hostname, 
            completionHandler: completionHandler
        )
    }
    
    private func performPinningValidation(
        challenge: URLAuthenticationChallenge,
        hostname: String,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard let serverTrust = challenge.protectionSpace.serverTrust,
              let pins = buildTimePins[hostname] else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        // Extract public key and validate against build-time pins
        guard let serverCert = SecTrustGetCertificateAtIndex(serverTrust, 0),
              let serverPublicKey = SecCertificateCopyKey(serverCert),
              let serverPublicKeyData = SecKeyCopyExternalRepresentation(serverPublicKey, nil) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        let serverKeyHash = sha256(data: serverPublicKeyData as Data)
        
        // Check if server key hash matches any of our build-time pins
        if pins.contains(serverKeyHash) {
            let credential = URLCredential(trust: serverTrust)
            completionHandler(.useCredential, credential)
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
    
    private func sha256(data: Data) -> String {
        var hash = [UInt8](repeating: 0, count: Int(CC_SHA256_DIGEST_LENGTH))
        data.withUnsafeBytes {
            _ = CC_SHA256($0.baseAddress, CC_LONG(data.count), &hash)
        }
        return Data(hash).base64EncodedString()
    }
}
```

## Step 3: Network Manager Integration

### Network Manager
```swift
// NetworkManager.swift
import Foundation

class NetworkManager {
    private let certificatePinner = CertificatePinner()
    private var urlSession: URLSession?
    
    func getURLSession() async -> URLSession {
        if urlSession == nil {
            urlSession = await certificatePinner.createURLSession()
        }
        return urlSession!
    }
    
    // Call this when you need to refresh configuration
    func refreshConfiguration() async {
        await certificatePinner.refreshConfiguration()
        urlSession = await certificatePinner.createURLSession()
    }
}

// Usage in your API calls
class APIService {
    private let networkManager = NetworkManager()
    
    func makeAPICall() async throws -> Data {
        let session = await networkManager.getURLSession()
        let url = URL(string: "https://apigee.kreditplus.com/api/v1/data")!
        
        let (data, response) = try await session.data(from: url)
        
        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw URLError(.badServerResponse)
        }
        
        return data
    }
}
```

## Step 4: Migration from Part 1 to Part 2

### Before (Part 1 - Basic Implementation)
```swift
class BasicCertificatePinner: NSObject, URLSessionDelegate {
    func createURLSession() -> URLSession {
        return URLSession(
            configuration: .default,
            delegate: self,
            delegateQueue: nil
        )
    }
    
    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        // Always perform pinning validation
        performPinningValidation(challenge: challenge, completionHandler: completionHandler)
    }
}
```

### After (Part 2 - Enhanced Implementation)
```swift
class EnhancedCertificatePinner: NSObject, URLSessionDelegate {
    private let remoteConfigManager = RemoteConfigManager()
    
    func createURLSession() async -> URLSession {
        // NEW: Check remote config before creating session
        await remoteConfigManager.fetchAndActivate()
        
        return URLSession(
            configuration: .default,
            delegate: self,
            delegateQueue: nil
        )
    }
    
    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        let hostname = challenge.protectionSpace.host
        
        // NEW: Check if this host should be pinned according to remote config
        if remoteConfigManager.shouldPinHost(hostname) {
            // Use existing Part 1 pinning logic
            performPinningValidation(challenge: challenge, completionHandler: completionHandler)
        } else {
            // NEW: Bypass pinning for this host
            completionHandler(.performDefaultHandling, nil)
        }
    }
}
```

## Step 5: App Delegate Setup

### App Delegate Configuration
```swift
// AppDelegate.swift
import UIKit
import Firebase

@main
class AppDelegate: UIResponder, UIApplicationDelegate {
    
    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        
        // Configure Firebase
        FirebaseApp.configure()
        
        return true
    }
}
```

## Dependencies

### Add to Package.swift or Podfile

#### Using Swift Package Manager
```swift
dependencies: [
    .package(url: "https://github.com/firebase/firebase-ios-sdk", from: "10.15.0")
]
```

#### Using CocoaPods
```ruby
# Podfile
target 'YourApp' do
  use_frameworks!
  
  pod 'Firebase/RemoteConfig'
  pod 'Firebase/Analytics' # Optional
end
```

## Testing

### Unit Test Example
```swift
// RemoteConfigManagerTests.swift
import XCTest
@testable import YourApp

class RemoteConfigManagerTests: XCTestCase {
    
    func testShouldDisablePinningWhenConfigDisabled() {
        // Mock Firebase Remote Config to return disabled config
        let manager = RemoteConfigManager()
        // ... test implementation
    }
    
    func testShouldDisablePinningWhenAppVersionBelowMinimum() {
        // Test version comparison logic
        let manager = RemoteConfigManager()
        // ... test implementation
    }
    
    func testVersionComparison() {
        let manager = RemoteConfigManager()
        
        // Test various version scenarios
        XCTAssertTrue(manager.compareVersions("1.5.0", "1.4.9") > 0)
        XCTAssertTrue(manager.compareVersions("1.5.0", "1.5.0") == 0)
        XCTAssertTrue(manager.compareVersions("1.4.9", "1.5.0") < 0)
    }
}
```

### Integration Test
```swift
// IntegrationTests.swift
import XCTest
@testable import YourApp

class IntegrationTests: XCTestCase {
    
    func testKillSwitchFunctionality() async {
        // Setup: Enable kill switch in Firebase console
        let certificatePinner = CertificatePinner()
        let session = await certificatePinner.createURLSession()
        
        // Test: Make request to pinned host
        let url = URL(string: "https://apigee.kreditplus.com/api/test")!
        
        do {
            let (_, response) = try await session.data(from: url)
            // Verify: Request should succeed with normal TLS
            XCTAssertNotNil(response)
        } catch {
            XCTFail("Request should succeed when kill switch is enabled")
        }
    }
}
```

## Best Practices

### 1. **Error Handling**
- Always provide safe defaults when Remote Config fails
- Don't crash the app if configuration parsing fails
- Log configuration errors for debugging

### 2. **Performance**
- Cache the URLSession instance to avoid repeated configuration fetches
- Use appropriate fetch intervals (1 hour for production, immediate for debug)
- Consider background configuration refresh

### 3. **Security**
- Keep certificate pins embedded in the app build
- Use Firebase Remote Config only for control logic
- Validate configuration before applying changes

### 4. **SwiftUI Integration**
```swift
// SwiftUI Integration Example
@StateObject private var networkManager = NetworkManager()

var body: some View {
    ContentView()
        .task {
            await networkManager.refreshConfiguration()
        }
}
```

**📋 For configuration management, emergency procedures, and operational guidance, see Section 10.**
