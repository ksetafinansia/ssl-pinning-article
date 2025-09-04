# Android Kill Switch Implementation (Kotlin)

This guide shows how to implement Firebase Remote Config kill switch for SSL pinning in Android applications using Kotlin and OkHttp.

## Prerequisites
- Existing SSL pinning implementation from **Section 6 Part 1**
- Firebase project with Remote Config enabled
- Firebase Android SDK integrated

## Step 1: Firebase Remote Config Setup

### Remote Config Manager
```kotlin
// RemoteConfigManager.kt
import com.google.firebase.remoteconfig.FirebaseRemoteConfig
import com.google.firebase.remoteconfig.FirebaseRemoteConfigSettings
import kotlinx.coroutines.tasks.await
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json

@Serializable
data class SSLPinningConfig(
    val enabled: Boolean = false,
    val targetHosts: List<String> = emptyList(),
    val minAppVersion: String = "1.0.0"
)

class RemoteConfigManager {
    private val remoteConfig = FirebaseRemoteConfig.getInstance()
    private val json = Json { ignoreUnknownKeys = true }
    
    companion object {
        private const val SSL_PINNING_CONFIG_KEY = "ssl_pinning_config"
        private const val FETCH_INTERVAL_SECONDS = 3600L // 1 hour
    }
    
    init {
        val configSettings = FirebaseRemoteConfigSettings.Builder()
            .setMinimumFetchIntervalInSeconds(if (BuildConfig.DEBUG) 0 else FETCH_INTERVAL_SECONDS)
            .build()
        remoteConfig.setConfigSettingsAsync(configSettings)
        
        // Set default values
        remoteConfig.setDefaultsAsync(
            mapOf(
                SSL_PINNING_CONFIG_KEY to getDefaultConfigJson()
            )
        )
    }
    
    suspend fun fetchAndActivate(): Boolean {
        return try {
            remoteConfig.fetchAndActivate().await()
        } catch (e: Exception) {
            // Log error but don't crash
            false
        }
    }
    
    fun getSSLPinningConfig(): SSLPinningConfig {
        return try {
            val configJson = remoteConfig.getString(SSL_PINNING_CONFIG_KEY)
            
            // Handle both array and comma-separated string formats
            val parsedConfig = json.decodeFromString<Map<String, Any>>(configJson)
            val targetHosts = when (val hosts = parsedConfig["target_hosts"]) {
                is List<*> -> hosts.mapNotNull { it as? String }
                is String -> if (hosts.isBlank()) emptyList() else hosts.split(",").map { it.trim() }
                else -> emptyList()
            }
            
            SSLPinningConfig(
                enabled = parsedConfig["enabled"] as? Boolean ?: false,
                targetHosts = targetHosts,
                minAppVersion = parsedConfig["min_app_version"] as? String ?: "1.0.0"
            )
        } catch (e: Exception) {
            // Return safe defaults if parsing fails
            SSLPinningConfig()
        }
    }
    
    private fun getDefaultConfigJson(): String {
        val defaultConfig = SSLPinningConfig(
            enabled = false, // Safe default - pinning disabled
            targetHosts = emptyList(),
            minAppVersion = "1.0.0"
        )
        return json.encodeToString(SSLPinningConfig.serializer(), defaultConfig)
    }
    
    // Check if current user should have pinning enabled
    fun shouldEnablePinning(): Boolean {
        val config = getSSLPinningConfig()
        
        // Check if pinning is globally enabled
        if (!config.enabled) {
            return false
        }
        
        // Check app version requirement
        if (!isAppVersionSupported(config.minAppVersion)) {
            return false
        }
        
        // Check if we have any target hosts
        if (config.targetHosts.isEmpty()) {
            return false
        }
        
        return true
    }
    
    // Check if a specific host should be pinned
    fun shouldPinHost(hostname: String): Boolean {
        if (!shouldEnablePinning()) return false
        
        val config = getSSLPinningConfig()
        return config.targetHosts.contains(hostname)
    }
    
    private fun isAppVersionSupported(minVersion: String): Boolean {
        return try {
            val currentVersion = BuildConfig.VERSION_NAME
            compareVersions(currentVersion, minVersion) >= 0
        } catch (e: Exception) {
            false
        }
    }
    
    private fun compareVersions(version1: String, version2: String): Int {
        val v1Parts = version1.split(".").map { it.toIntOrNull() ?: 0 }
        val v2Parts = version2.split(".").map { it.toIntOrNull() ?: 0 }
        val maxLength = maxOf(v1Parts.size, v2Parts.size)
        
        for (i in 0 until maxLength) {
            val v1Part = v1Parts.getOrNull(i) ?: 0
            val v2Part = v2Parts.getOrNull(i) ?: 0
            val result = v1Part.compareTo(v2Part)
            if (result != 0) return result
        }
        return 0
    }
}
```

## Step 2: Enhanced HTTP Client from Part 1

### Updated OkHttp Client
```kotlin
// EnhancedPinnedHttpClient.kt
import okhttp3.*
import okhttp3.CertificatePinner

class EnhancedPinnedHttpClient {
    private val remoteConfigManager = RemoteConfigManager()
    
    // Part 1 pins - these remain in the build for security
    companion object {
        private const val APIGEE_PROD = "apigee.kreditplus.com"
        private const val APIGEE_STAGING = "apigee.kbfinansia.com"
        private const val PRIMARY_PIN = "sha256/your_build_time_primary_pin"
        private const val BACKUP_PIN = "sha256/your_build_time_backup_pin"
        private const val STAGING_PIN = "sha256/your_staging_pin"
    }
    
    suspend fun createClient(): OkHttpClient {
        // NEW: Check remote config before creating client
        remoteConfigManager.fetchAndActivate()
        
        return if (remoteConfigManager.shouldEnablePinning()) {
            createPinnedClient() // Use Part 1 pinning logic
        } else {
            createNormalClient() // Bypass pinning
        }
    }
    
    private fun createPinnedClient(): OkHttpClient {
        // SAME: Use Part 1 pinning configuration with build-time pins
        val certificatePinner = CertificatePinner.Builder()
            .add(APIGEE_PROD, PRIMARY_PIN)
            .add(APIGEE_PROD, BACKUP_PIN)
            .add(APIGEE_STAGING, STAGING_PIN)
            .build()
        
        return OkHttpClient.Builder()
            .certificatePinner(certificatePinner)
            .addInterceptor(HostCheckInterceptor(remoteConfigManager)) // NEW: Remote host control
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .build()
    }
    
    private fun createNormalClient(): OkHttpClient {
        // NEW: Option to bypass pinning completely
        return OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .build()
    }
}

// NEW: Interceptor that checks remote config for each request
class HostCheckInterceptor(
    private val remoteConfigManager: RemoteConfigManager
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val hostname = request.url.host
        
        // Check if this specific host should be pinned according to remote config
        if (!remoteConfigManager.shouldPinHost(hostname)) {
            // Host not in remote config target list
            // The CertificatePinner will still try to pin it, but we could handle this differently
            // For this implementation, we let it proceed and pin if configured in Part 1
        }
        
        return chain.proceed(request)
    }
}
```

## Step 3: Integration in Application

### Usage Example
```kotlin
// MainActivity.kt or Application class
class NetworkManager {
    private val httpClientFactory = EnhancedPinnedHttpClient()
    private var httpClient: OkHttpClient? = null
    
    suspend fun getHttpClient(): OkHttpClient {
        if (httpClient == null) {
            httpClient = httpClientFactory.createClient()
        }
        return httpClient!!
    }
    
    // Call this when you need to refresh configuration
    suspend fun refreshConfiguration() {
        httpClient = httpClientFactory.createClient()
    }
}

// Usage in your API calls
class ApiService {
    private val networkManager = NetworkManager()
    
    suspend fun makeApiCall(): String {
        val client = networkManager.getHttpClient()
        val request = Request.Builder()
            .url("https://apigee.kreditplus.com/api/v1/data")
            .build()
            
        return client.newCall(request).execute().use { response ->
            response.body?.string() ?: ""
        }
    }
}
```

## Step 4: Migration from Part 1 to Part 2

### Before (Part 1 - Basic Implementation)
```kotlin
class PinnedHttpClient {
    companion object {
        val instance: OkHttpClient by lazy {
            val certificatePinner = CertificatePinner.Builder()
                .add("apigee.kreditplus.com", "sha256/pin")
                .build()
                
            OkHttpClient.Builder()
                .certificatePinner(certificatePinner)
                .build()
        }
    }
}
```

### After (Part 2 - Enhanced Implementation)
```kotlin
class EnhancedPinnedHttpClient {
    private val remoteConfigManager = RemoteConfigManager()
    
    suspend fun createClient(): OkHttpClient {
        // Add this check before using Part 1 logic
        return if (remoteConfigManager.shouldEnablePinning()) {
            // Use existing Part 1 pinning logic
            createPinnedClientFromPart1()
        } else {
            // New: Bypass pinning capability
            createNormalClient()
        }
    }
    
    private fun createPinnedClientFromPart1(): OkHttpClient {
        // Your existing Part 1 code here
        val certificatePinner = CertificatePinner.Builder()
            .add("apigee.kreditplus.com", "sha256/pin")
            .build()
            
        return OkHttpClient.Builder()
            .certificatePinner(certificatePinner)
            .build()
    }
}
```

## Dependencies

### Add to build.gradle (Module: app)
```kotlin
dependencies {
    // Firebase Remote Config
    implementation 'com.google.firebase:firebase-config-ktx:21.4.1'
    
    // Kotlin serialization for config parsing
    implementation 'org.jetbrains.kotlinx:kotlinx-serialization-json:1.5.1'
    
    // OkHttp (if not already included)
    implementation 'com.squareup.okhttp3:okhttp:4.11.0'
    
    // Coroutines
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'
}
```

### Add to build.gradle (Module: app) - plugins
```kotlin
plugins {
    id 'kotlinx-serialization'
    id 'com.google.gms.google-services'
}
```

## Testing

### Unit Test Example
```kotlin
// RemoteConfigManagerTest.kt
@RunWith(MockitoJUnitRunner::class)
class RemoteConfigManagerTest {
    
    @Test
    fun `should disable pinning when config enabled is false`() {
        // Mock Firebase Remote Config to return disabled config
        val manager = RemoteConfigManager()
        // ... test implementation
    }
    
    @Test
    fun `should disable pinning when app version is below minimum`() {
        // Test version comparison logic
        val manager = RemoteConfigManager()
        // ... test implementation
    }
}
```

### Integration Test
```kotlin
// Test kill switch functionality
@Test
fun `should use normal TLS when kill switch is activated`() = runTest {
    // Setup: Enable kill switch in Firebase console
    val httpClient = EnhancedPinnedHttpClient().createClient()
    
    // Verify: Client should not have certificate pinner
    assertNull(httpClient.certificatePinner)
}
```

## Best Practices

### 1. **Error Handling**
- Always provide safe defaults when Remote Config fails
- Don't crash the app if configuration parsing fails
- Log configuration errors for debugging

### 2. **Performance**
- Cache the HTTP client instance to avoid repeated configuration fetches
- Use appropriate fetch intervals (1 hour for production, immediate for debug)
- Consider background configuration refresh

### 3. **Security**
- Keep certificate pins embedded in the app build
- Use Firebase Remote Config only for control logic
- Validate configuration before applying changes

**📋 For configuration management, emergency procedures, and operational guidance, see Section 10.**
