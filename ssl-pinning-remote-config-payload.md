# SSL Pinning Remote Config Payload Documentation

## Overview

This document defines the Firebase Remote Config payload structure for SSL pinning configurations across Flutter, Android (Kotlin), and iOS (Swift) implementations.

## Payload Structure

The Firebase Remote Config should contain a JSON payload with the following structure:

```json
{
  "configurations": [
    {
      "host": "apigee.kreditplus.com",
      "is_enabled": true,
      "is_android_enable": true,
      "is_ios_enable": true,
      "pins": {
        "primary": "sha256/AAAB3NzaC1yc2EAAAA...",
        "backup": "sha256/BBBF4OzaC1yc2EAAAA...",
        "emergency": "sha256/CCCG5PzaC1yc2EAAAA..."
      }
    },
    {
      "host": "api.kreditplus.com",
      "is_enabled": true,
      "is_android_enable": false,
      "is_ios_enable": true,
      "pins": {
        "primary": "sha256/DDDE6PzaC1yc2EAAAA...",
        "backup": "sha256/EEEF7QzaC1yc2EAAAA...",
        "emergency": "sha256/FFFG8RzaC1yc2EAAAA..."
      }
    }
  ]
}
```

## Field Definitions

### Root Level
- `configurations`: Array of SSL pinning configurations for different hosts

### Configuration Object
- `host`: String - The hostname for which this configuration applies (e.g., "apigee.kreditplus.com")
- `is_enabled`: Boolean - Master enable/disable flag for SSL pinning on this host
- `is_android_enable`: Boolean - Platform-specific flag to enable/disable SSL pinning on Android
- `is_ios_enable`: Boolean - Platform-specific flag to enable/disable SSL pinning on iOS
- `pins`: Object - Contains the public key hashes for certificate pinning

### Pins Object
- `primary`: String - Primary SHA-256 public key hash
- `backup`: String - Backup SHA-256 public key hash (used if primary fails)
- `emergency`: String - Emergency SHA-256 public key hash (used as last resort)

## Platform-Specific Behavior

### Flutter Implementation
Flutter apps will evaluate SSL pinning based on:
1. `is_enabled` must be `true`
2. Platform-specific flag must be `true`:
   - On Android: `is_android_enable` must be `true`
   - On iOS: `is_ios_enable` must be `true`

**Flutter Logic:**
```dart
bool isEnabledForHost(String host) {
  final config = getConfigForHost(host);
  if (config == null || !config.isEnabled) return false;
  
  // Platform-specific checks
  if (Platform.isAndroid && !config.isAndroidEnable) return false;
  if (Platform.isIOS && !config.isIosEnable) return false;
  
  return true;
}
```

### Android (Kotlin) Implementation
Android native apps will only check:
- `is_enabled` flag

**Android Logic:**
```kotlin
fun isEnabledForHost(host: String): Boolean {
    val config = getConfigForHost(host)
    return config?.isEnabled ?: false
}
```

### iOS (Swift) Implementation
iOS native apps will only check:
- `is_enabled` flag

**iOS Logic:**
```swift
func isEnabledForHost(_ host: String) -> Bool {
    guard let config = getConfigForHost(host) else { return false }
    return config.isEnabled
}
```

## Configuration Examples

### Example 1: Full Enable
```json
{
  "host": "apigee.kreditplus.com",
  "is_enabled": true,
  "is_android_enable": true,
  "is_ios_enable": true,
  "pins": { ... }
}
```
- **Result**: SSL pinning enabled on all platforms

### Example 2: Disable Android Only
```json
{
  "host": "api.kreditplus.com",
  "is_enabled": true,
  "is_android_enable": false,
  "is_ios_enable": true,
  "pins": { ... }
}
```
- **Result**: 
  - Flutter on Android: SSL pinning disabled
  - Flutter on iOS: SSL pinning enabled
  - Native Android: SSL pinning enabled
  - Native iOS: SSL pinning enabled

### Example 3: Master Disable
```json
{
  "host": "test.kreditplus.com",
  "is_enabled": false,
  "is_android_enable": true,
  "is_ios_enable": true,
  "pins": { ... }
}
```
- **Result**: SSL pinning disabled on all platforms

### Example 4: iOS Only
```json
{
  "host": "mobile.kreditplus.com",
  "is_enabled": true,
  "is_android_enable": false,
  "is_ios_enable": true,
  "pins": { ... }
}
```
- **Result**:
  - Flutter on Android: SSL pinning disabled
  - Flutter on iOS: SSL pinning enabled
  - Native Android: SSL pinning enabled
  - Native iOS: SSL pinning enabled

## Hash Format

All pin hashes should be in SHA-256 format with base64 encoding:
- Format: `sha256/[BASE64_ENCODED_HASH]`
- Example: `sha256/AAAB3NzaC1yc2EAAAA...`

## Default Configuration

When remote config is unavailable, implementations should use this fallback:

```json
{
  "configurations": [
    {
      "host": "apigee.kreditplus.com",
      "is_enabled": true,
      "is_android_enable": true,
      "is_ios_enable": true,
      "pins": {
        "primary": "sha256/[FALLBACK_HASH_1]",
        "backup": "sha256/[FALLBACK_HASH_2]",
        "emergency": "sha256/[FALLBACK_HASH_3]"
      }
    }
  ]
}
```

## Validation Rules

1. **host**: Must be a valid hostname/domain
2. **is_enabled**: Must be boolean
3. **is_android_enable**: Must be boolean
4. **is_ios_enable**: Must be boolean  
5. **pins**: Must contain all three hashes (primary, backup, emergency)
6. **Hash format**: Must start with "sha256/" followed by base64 encoded string

## Security Considerations

1. **Hash Encryption**: In production, consider encrypting the pin hashes
2. **Access Control**: Limit Firebase Remote Config access to authorized personnel
3. **Audit Trail**: Enable logging for configuration changes
4. **Testing**: Always test configuration changes in staging environment first
5. **Rollback Plan**: Keep previous working configurations for quick rollback

## Remote Config Setup

### Firebase Console Configuration
1. Navigate to Firebase Remote Config
2. Create parameter: `ssl_pinning_configurations`
3. Set the JSON payload as the default value
4. Configure conditions for different app versions if needed
5. Publish the configuration

### Fetch Settings
Recommended Remote Config settings:
- **Fetch Timeout**: 60 seconds
- **Minimum Fetch Interval**: 3600 seconds (1 hour)
- **Cache Expiration**: 43200 seconds (12 hours)

This ensures configurations are fetched regularly while minimizing network calls and providing offline support.
