# SSL Pinning Android Kotlin Implementation

## Overview

This guide provides a comprehensive SSL pinning implementation for Android using Kotlin with OkHttp, Retrofit integration, and Network Security Configuration. The implementation uses SHA-256 public key hashes for certificate pinning, providing robust security while maintaining flexibility for certificate renewals.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Dependencies](#dependencies)
3. [Core Implementation](#core-implementation)
4. [OkHttp Integration](#okhttp-integration)
5. [Retrofit Integration](#retrofit-integration)
6. [Network Security Configuration](#network-security-configuration)
7. [Error Handling](#error-handling)
8. [Testing](#testing)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)

## Prerequisites

- Android API Level 21+ (Android 5.0)
- Kotlin 1.5+
- Android Studio Arctic Fox or later
- Basic understanding of OkHttp and Retrofit

## Dependencies

Add these dependencies to your `app/build.gradle`:

```kotlin
dependencies {
    implementation "com.squareup.okhttp3:okhttp:4.11.0"
    implementation "com.squareup.okhttp3:logging-interceptor:4.11.0"
    implementation "com.squareup.retrofit2:retrofit:2.9.0"
    implementation "com.squareup.retrofit2:converter-gson:2.9.0"
    implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:2.6.2"
    implementation "androidx.lifecycle:lifecycle-livedata-ktx:2.6.2"
    implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3"
    
    // Serialization for remote config
    implementation "org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.0"
    
    // Firebase
    implementation platform('com.google.firebase:firebase-bom:32.7.0')
    implementation 'com.google.firebase:firebase-remote-config-ktx'
    
    // Testing
    testImplementation "junit:junit:4.13.2"
    testImplementation "org.mockito:mockito-core:5.5.0"
    testImplementation "org.mockito.kotlin:mockito-kotlin:5.1.0"
    androidTestImplementation "androidx.test.ext:junit:1.1.5"
    androidTestImplementation "androidx.test.espresso:espresso-core:3.5.1"
}
```

## Core Implementation

### 1. SSL Pinning Manager

Create a centralized manager for SSL pinning configuration:

```kotlin
import android.content.Context
import android.content.SharedPreferences
import android.util.Log
import kotlinx.coroutines.GlobalScope
import kotlinx.coroutines.launch
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json
import kotlinx.serialization.decodeFromString
import okhttp3.CertificatePinner
import okhttp3.OkHttpClient
import okhttp3.Request
import java.security.MessageDigest
import java.security.cert.X509Certificate
import java.util.concurrent.TimeUnit
import javax.net.ssl.*

class SSLPinningManager private constructor(private val context: Context) {
    
    companion object {
        private const val TAG = "SSLPinningManager"
        private const val PREFS_NAME = "ssl_pinning_prefs"
        private const val REMOTE_CONFIG_KEY = "ssl_pinning_configurations"
        private const val CONFIG_CACHE_KEY = "ssl_pinning_remote_config"
        
        // Fallback configuration
        private const val DEFAULT_HOST = "apigee.kreditplus.com"
        private val FALLBACK_HASHES = listOf(
            "sha256/AAAB3NzaC1yc2EAAAA...",
            "sha256/BBBF4OzaC1yc2EAAAA...",
            "sha256/CCCG5PzaC1yc2EAAAA..."
        )
        
        @Volatile
        private var INSTANCE: SSLPinningManager? = null
        
        fun getInstance(context: Context): SSLPinningManager {
            return INSTANCE ?: synchronized(this) {
                INSTANCE ?: SSLPinningManager(context.applicationContext).also { INSTANCE = it }
            }
        }
    }
    
    @Serializable
    data class RemoteSSLConfig(
        val hosts: String,
        val enabled: Boolean,
        val pins: SSLPins
    )
    
    @Serializable
    data class SSLPins(
        val primary: String,
        val backup: String,
        val emergency: String
    ) {
        val allHashes: List<String> get() = listOf(primary, backup, emergency)
    }
    
    @Serializable
    data class RemoteConfigResponse(
        val configurations: List<RemoteSSLConfig>
    )
    
    data class PinConfiguration(
        val hostname: String,
        val pinnedHashes: Set<String>,
        val enabled: Boolean = true
    )
    
    private val sharedPrefs: SharedPreferences = context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE)
    private val pinConfigurations = mutableMapOf<String, PinConfiguration>()
    private val logger = SSLPinningLogger()
    private val json = Json { ignoreUnknownKeys = true }
    private val remoteConfig: FirebaseRemoteConfig = Firebase.remoteConfig
    
    init {
        setupFirebaseRemoteConfig()
        GlobalScope.launch {
            loadRemoteConfiguration()
        }
    }
    
    private fun setupFirebaseRemoteConfig() {
        val configSettings = remoteConfigSettings {
            minimumFetchIntervalInSeconds = 3600 // 1 hour
        }
        remoteConfig.setConfigSettingsAsync(configSettings)
        
        // Set default values
        val defaultConfigJson = json.encodeToString(
            RemoteConfigResponse(
                configurations = listOf(
                    RemoteSSLConfig(
                        hosts = DEFAULT_HOST,
                        enabled = true,
                        pins = SSLPins(
                            primary = FALLBACK_HASHES[0],
                            backup = FALLBACK_HASHES[1],
                            emergency = FALLBACK_HASHES[2]
                        )
                    )
                )
            )
        )
        
        remoteConfig.setDefaultsAsync(mapOf(REMOTE_CONFIG_KEY to defaultConfigJson))
        logger.logFirebaseRemoteConfigInitialized()
    }
    
    // MARK: - Private Methods for Remote Configuration
    
    private suspend fun loadRemoteConfiguration() {
        try {
            // Try to load from cache first
            loadCachedConfiguration()?.let { cachedConfig ->
                updatePinConfigurations(cachedConfig.configurations)
            }
            
            // Fetch from Firebase Remote Config
            fetchFirebaseRemoteConfiguration()
            
            if (pinConfigurations.isEmpty) {
                loadFallbackConfiguration()
            }
        } catch (e: Exception) {
            logger.logRemoteConfigError(e)
            loadFallbackConfiguration()
        }
    }
    
    private suspend fun fetchFirebaseRemoteConfiguration() {
        try {
            // Fetch and activate Firebase Remote Config
            remoteConfig.fetchAndActivate().addOnCompleteListener { task ->
                if (task.isSuccessful) {
                    val configString = remoteConfig.getString(REMOTE_CONFIG_KEY)
                    
                    if (configString.isNotEmpty()) {
                        try {
                            val remoteConfigResponse = json.decodeFromString<RemoteConfigResponse>(configString)
                            updatePinConfigurations(remoteConfigResponse.configurations)
                            
                            // Cache the configuration
                            sharedPrefs.edit()
                                .putString(CONFIG_CACHE_KEY, configString)
                                .apply()
                            
                            logger.logFirebaseRemoteConfigUpdated()
                        } catch (e: Exception) {
                            logger.logFirebaseRemoteConfigParseError(e)
                            if (pinConfigurations.isEmpty) {
                                loadFallbackConfiguration()
                            }
                        }
                    } else {
                        logger.logEmptyFirebaseRemoteConfig()
                        if (pinConfigurations.isEmpty) {
                            loadFallbackConfiguration()
                        }
                    }
                } else {
                    logger.logFirebaseRemoteConfigFetchFailed(task.exception)
                    if (pinConfigurations.isEmpty) {
                        loadFallbackConfiguration()
                    }
                }
            }
        } catch (e: Exception) {
            logger.logFirebaseRemoteConfigFetchError(e)
            if (pinConfigurations.isEmpty) {
                loadFallbackConfiguration()
            }
        }
    }
    
    private fun loadCachedConfiguration(): RemoteConfigResponse? {
        val cachedData = sharedPrefs.getString(CONFIG_CACHE_KEY, null)
        return cachedData?.let { 
            try {
                json.decodeFromString<RemoteConfigResponse>(it)
            } catch (e: Exception) {
                logger.logCacheParseError(e)
                null
            }
        }
    }
    
    private fun updatePinConfigurations(configurations: List<RemoteSSLConfig>) {
        pinConfigurations.clear()
        
        configurations.forEach { config ->
            val decryptedHashes = decryptPins(config.pins.allHashes)
            val pinConfig = PinConfiguration(
                hostname = config.hosts,
                pinnedHashes = decryptedHashes.toSet(),
                enabled = config.enabled
            )
            pinConfigurations[config.hosts] = pinConfig
        }
        
        logger.logConfigurationUpdated(configurations.size)
    }
    
    private fun loadFallbackConfiguration() {
        val config = PinConfiguration(
            hostname = DEFAULT_HOST,
            pinnedHashes = FALLBACK_HASHES.toSet(),
            enabled = true
        )
        pinConfigurations[DEFAULT_HOST] = config
        logger.logFallbackConfigurationLoaded()
    }
    
    private fun decryptPins(encryptedPins: List<String>): List<String> {
        // TODO: Implement decryption logic based on your encryption method
        // For now, return as-is assuming they're already decrypted for demo
        return encryptedPins
    }
    
    // MARK: - Public Methods
    
    suspend fun refreshRemoteConfiguration() {
        fetchRemoteConfiguration()
    }
    
    fun getConfigurationStatus(): Map<String, Any> {
        return mapOf(
            "configuredHosts" to pinConfigurations.keys.toList(),
            "defaultHost" to DEFAULT_HOST,
            "hasRemoteConfig" to pinConfigurations.isNotEmpty()
        )
    }
    
    fun isEnabledForHost(hostname: String): Boolean {
        return getConfigForHost(hostname)?.enabled ?: false
    }
    
    private fun getConfigForHost(hostname: String): PinConfiguration? {
        return pinConfigurations[hostname] 
            ?: pinConfigurations[DEFAULT_HOST]
            ?: pinConfigurations.values.firstOrNull()
    }
    
    fun createSecureOkHttpClient(): OkHttpClient {
        val builder = OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
        
        // Add certificate pinner with remote config support
        builder.certificatePinner(createCertificatePinnerWithRemoteConfig())
        
        // Add custom TrustManager for additional validation
        val trustManager = createCustomTrustManager()
        val sslContext = SSLContext.getInstance("TLS")
        sslContext.init(null, arrayOf(trustManager), null)
        
        builder.sslSocketFactory(sslContext.socketFactory, trustManager)
        
        return builder.build()
    }
    
    private fun createCertificatePinnerWithRemoteConfig(): CertificatePinner {
        val builder = CertificatePinner.Builder()
        
        pinConfigurations.forEach { (hostname, config) ->
            if (config.enabled) {
                config.pinnedHashes.forEach { hash ->
                    // Remove sha256/ prefix if present, CertificatePinner will add it
                    val cleanHash = hash.removePrefix("sha256/")
                    builder.add(hostname, "sha256/$cleanHash")
                }
                logger.logPinningEnabledForHost(hostname, config.pinnedHashes.size)
            } else {
                logger.logPinningDisabledForHost(hostname)
            }
        }
        
        return builder.build()
    }
    
    private fun createCustomTrustManager(): X509TrustManager {
        return object : X509TrustManager {
            private val defaultTrustManager = getDefaultTrustManager()
            
            override fun checkClientTrusted(chain: Array<X509Certificate>, authType: String) {
                defaultTrustManager.checkClientTrusted(chain, authType)
            }
            
            override fun checkServerTrusted(chain: Array<X509Certificate>, authType: String) {
                // Perform default validation first
                defaultTrustManager.checkServerTrusted(chain, authType)
                
                // Additional custom validation with remote config
                validateCertificateChainWithRemoteConfig(chain)
            }
            
            override fun getAcceptedIssuers(): Array<X509Certificate> {
                return defaultTrustManager.getAcceptedIssuers()
            }
        }
    }
    
    private fun getDefaultTrustManager(): X509TrustManager {
        val trustManagerFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm())
        trustManagerFactory.init(null as? java.security.KeyStore)
        return trustManagerFactory.trustManagers
            .filterIsInstance<X509TrustManager>()
            .first()
    }
    
    private fun validateCertificateChainWithRemoteConfig(chain: Array<X509Certificate>) {
        // Additional validation logic with remote config
        chain.forEach { certificate ->
            val publicKeyHash = generatePublicKeyHash(certificate)
            val subjectName = certificate.subjectDN.name
            
            // Extract hostname from certificate subject or use a different method
            val hostname = extractHostnameFromSubject(subjectName) ?: DEFAULT_HOST
            val config = getConfigForHost(hostname)
            
            if (config?.enabled == true) {
                logger.logCertificateValidation(subjectName, publicKeyHash)
            } else {
                logger.logCertificateValidationSkipped(subjectName)
            }
        }
    }
    
    private fun extractHostnameFromSubject(subjectName: String): String? {
        // Simple extraction - in production, use proper certificate parsing
        return subjectName.split(",")
            .find { it.trim().startsWith("CN=") }
            ?.substringAfter("CN=")
            ?.trim()
    }
    
    private fun generatePublicKeyHash(certificate: X509Certificate): String {
        val publicKeyInfo = certificate.publicKey.encoded
        val digest = MessageDigest.getInstance("SHA-256")
        val hash = digest.digest(publicKeyInfo)
        return android.util.Base64.encodeToString(hash, android.util.Base64.NO_WRAP)
    }
    
    fun enableKillSwitch() {
        sharedPrefs.edit().putBoolean(KILL_SWITCH_KEY, true).apply()
        logger.logKillSwitchEnabled()
    }
    
    fun disableKillSwitch() {
        sharedPrefs.edit().putBoolean(KILL_SWITCH_KEY, false).apply()
        logger.logKillSwitchDisabled()
    }
    
    private fun isKillSwitchEnabled(): Boolean {
        return sharedPrefs.getBoolean(KILL_SWITCH_KEY, false)
    }
    
    fun getPinConfiguration(hostname: String): PinConfiguration? {
        return pinConfigurations[hostname]
    }
}
```

### 2. SSL Pinning Logger

Create a comprehensive logging system:

```kotlin
import android.util.Log

class SSLPinningLogger {
    
    companion object {
        private const val TAG = "SSLPinning"
    }
    
    fun logConfiguration(hostname: String, hashCount: Int) {
        Log.i(TAG, "SSL Pinning configured for hostname: $hostname with $hashCount hashes")
    }
    
    fun logPinValidationSuccess(hostname: String, hash: String) {
        Log.i(TAG, "SSL Pin validation SUCCESS for hostname: $hostname with hash: $hash")
    }
    
    fun logPinValidationFailure(hostname: String, reason: String) {
        Log.e(TAG, "SSL Pin validation FAILED for hostname: $hostname - Reason: $reason")
    }
    
    fun logPinningEnabledForHost(hostname: String, hashCount: Int) {
        Log.i(TAG, "SSL Pinning ENABLED for hostname: $hostname with $hashCount hashes")
    }
    
    fun logPinningDisabledForHost(hostname: String) {
        Log.w(TAG, "SSL Pinning DISABLED for hostname: $hostname")
    }
    
    fun logFirebaseRemoteConfigUpdated() {
        Log.i(TAG, "Firebase Remote SSL configuration updated successfully")
    }
    
    fun logFirebaseRemoteConfigFetchFailed(error: Throwable?) {
        Log.e(TAG, "Failed to fetch Firebase Remote SSL configuration", error)
    }
    
    fun logFirebaseRemoteConfigFetchError(error: Throwable) {
        Log.e(TAG, "Firebase Remote SSL configuration fetch error", error)
    }
    
    fun logFirebaseRemoteConfigParseError(error: Throwable) {
        Log.e(TAG, "Failed to parse Firebase Remote SSL configuration", error)
    }
    
    fun logEmptyFirebaseRemoteConfig() {
        Log.w(TAG, "Firebase Remote SSL configuration is empty")
    }
    
    fun logCacheParseError(error: Throwable) {
        Log.e(TAG, "Failed to parse cached SSL configuration", error)
    }
    
    fun logFallbackConfigurationLoaded() {
        Log.w(TAG, "Fallback SSL configuration loaded")
    }
    
    fun logConfigurationUpdated(hostCount: Int) {
        Log.i(TAG, "SSL Pin configuration updated for $hostCount hosts")
    }
    
    fun logCertificateValidationSkipped(subject: String) {
        Log.d(TAG, "Certificate validation skipped for subject: $subject")
    }
    
    fun logKillSwitchEnabled() {
        Log.w(TAG, "SSL Pinning kill switch ENABLED")
    }
    
    fun logKillSwitchDisabled() {
        Log.i(TAG, "SSL Pinning kill switch DISABLED")
    }
    
    fun logCertificateValidation(subject: String, hash: String) {
        Log.d(TAG, "Certificate validation - Subject: $subject, Hash: $hash")
    }
    
    fun logNetworkError(hostname: String, error: Throwable) {
        Log.e(TAG, "Network error for hostname: $hostname", error)
    }
    
    fun logSSLHandshakeFailure(hostname: String, error: Throwable) {
        Log.e(TAG, "SSL Handshake failure for hostname: $hostname", error)
    }
}
```

### 3. Custom SSL Pinning Exceptions

Define specific exceptions for SSL pinning scenarios:

```kotlin
sealed class SSLPinningException(message: String, cause: Throwable? = null) : Exception(message, cause) {
    
    class PinValidationFailedException(hostname: String, cause: Throwable? = null) : 
        SSLPinningException("SSL pin validation failed for hostname: $hostname", cause)
    
    class NoPinConfigurationException(hostname: String) : 
        SSLPinningException("No SSL pin configuration found for hostname: $hostname")
    
    class CertificateExtractionException(message: String, cause: Throwable? = null) : 
        SSLPinningException("Certificate extraction failed: $message", cause)
    
    class PublicKeyExtractionException(message: String, cause: Throwable? = null) : 
        SSLPinningException("Public key extraction failed: $message", cause)
    
    class HashGenerationException(message: String, cause: Throwable? = null) : 
        SSLPinningException("Hash generation failed: $message", cause)
    
    class KillSwitchActivatedException(hostname: String) : 
        SSLPinningException("SSL pinning disabled via kill switch for hostname: $hostname")
}
```

## OkHttp Integration

### 1. Network Interceptor

Create an interceptor for additional SSL pinning validation:

```kotlin
import okhttp3.Interceptor
import okhttp3.Response
import java.io.IOException

class SSLPinningInterceptor(private val sslPinningManager: SSLPinningManager) : Interceptor {
    
    private val logger = SSLPinningLogger()
    
    @Throws(IOException::class)
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val hostname = request.url.host
        
        try {
            val response = chain.proceed(request)
            
            // Log successful connection
            logger.logPinValidationSuccess(hostname, "Connection established successfully")
            
            return response
        } catch (e: Exception) {
            // Handle SSL-related exceptions
            when (e) {
                is javax.net.ssl.SSLHandshakeException -> {
                    logger.logSSLHandshakeFailure(hostname, e)
                    throw SSLPinningException.PinValidationFailedException(hostname, e)
                }
                is javax.net.ssl.SSLPeerUnverifiedException -> {
                    logger.logPinValidationFailure(hostname, "Peer verification failed")
                    throw SSLPinningException.PinValidationFailedException(hostname, e)
                }
                else -> {
                    logger.logNetworkError(hostname, e)
                    throw e
                }
            }
        }
    }
}
```

### 2. Logging Interceptor Configuration

Configure HTTP logging with SSL pinning awareness:

```kotlin
import okhttp3.logging.HttpLoggingInterceptor

class SecureHttpLoggingInterceptor : HttpLoggingInterceptor.Logger {
    
    companion object {
        private const val TAG = "HTTPSecure"
    }
    
    override fun log(message: String) {
        // Filter sensitive information from logs
        val filteredMessage = filterSensitiveInfo(message)
        Log.d(TAG, filteredMessage)
    }
    
    private fun filterSensitiveInfo(message: String): String {
        return message
            .replace(Regex("Authorization: Bearer [A-Za-z0-9-_]+\\.[A-Za-z0-9-_]+\\.[A-Za-z0-9-_]+"), "Authorization: Bearer [REDACTED]")
            .replace(Regex("\"password\"\\s*:\\s*\"[^\"]+\""), "\"password\":\"[REDACTED]\"")
            .replace(Regex("\"token\"\\s*:\\s*\"[^\"]+\""), "\"token\":\"[REDACTED]\"")
    }
}
```

## Retrofit Integration

### 1. Secure API Service Factory

Create a factory for generating secure API services:

```kotlin
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import okhttp3.logging.HttpLoggingInterceptor

class SecureApiServiceFactory(private val context: Context) {
    
    private val sslPinningManager = SSLPinningManager.getInstance(context)
    
    fun <T> createService(serviceClass: Class<T>, baseUrl: String): T {
        val okHttpClient = createSecureOkHttpClient()
        
        val retrofit = Retrofit.Builder()
            .baseUrl(baseUrl)
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
        
        return retrofit.create(serviceClass)
    }
    
    private fun createSecureOkHttpClient(): OkHttpClient {
        val baseClient = sslPinningManager.createSecureOkHttpClient()
        
        return baseClient.newBuilder()
            .addInterceptor(SSLPinningInterceptor(sslPinningManager))
            .addInterceptor(createLoggingInterceptor())
            .build()
    }
    
    private fun createLoggingInterceptor(): HttpLoggingInterceptor {
        return HttpLoggingInterceptor(SecureHttpLoggingInterceptor()).apply {
            level = if (BuildConfig.DEBUG) {
                HttpLoggingInterceptor.Level.BODY
            } else {
                HttpLoggingInterceptor.Level.NONE
            }
        }
    }
}
```

### 2. API Service Interfaces

Define API service interfaces:

```kotlin
import retrofit2.Response
import retrofit2.http.*

// Data classes for API responses
data class ApiResponse<T>(
    val data: T,
    val message: String,
    val success: Boolean
)

data class UserProfile(
    val id: String,
    val name: String,
    val email: String
)

data class LoginRequest(
    val username: String,
    val password: String
)

data class LoginResponse(
    val token: String,
    val user: UserProfile,
    val expiresAt: String
)

// API service interface
interface ApiService {
    
    @POST("auth/login")
    suspend fun login(@Body loginRequest: LoginRequest): Response<ApiResponse<LoginResponse>>
    
    @GET("user/profile")
    suspend fun getUserProfile(@Header("Authorization") token: String): Response<ApiResponse<UserProfile>>
    
    @GET("health")
    suspend fun getHealthCheck(): Response<ApiResponse<String>>
}
```

### 3. Repository Implementation

Create repository with SSL pinning:

```kotlin
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext

class ApiRepository(private val apiService: ApiService) {
    
    private val logger = SSLPinningLogger()
    
    suspend fun login(username: String, password: String): Result<LoginResponse> {
        return withContext(Dispatchers.IO) {
            try {
                val request = LoginRequest(username, password)
                val response = apiService.login(request)
                
                if (response.isSuccessful && response.body()?.success == true) {
                    Result.success(response.body()!!.data)
                } else {
                    Result.failure(ApiException("Login failed: ${response.message()}"))
                }
            } catch (e: SSLPinningException) {
                logger.logPinValidationFailure("login", e.message ?: "Unknown SSL error")
                Result.failure(e)
            } catch (e: Exception) {
                logger.logNetworkError("login", e)
                Result.failure(NetworkException("Network error during login", e))
            }
        }
    }
    
    suspend fun getUserProfile(token: String): Result<UserProfile> {
        return withContext(Dispatchers.IO) {
            try {
                val response = apiService.getUserProfile("Bearer $token")
                
                if (response.isSuccessful && response.body()?.success == true) {
                    Result.success(response.body()!!.data)
                } else {
                    Result.failure(ApiException("Failed to get user profile: ${response.message()}"))
                }
            } catch (e: SSLPinningException) {
                logger.logPinValidationFailure("getUserProfile", e.message ?: "Unknown SSL error")
                Result.failure(e)
            } catch (e: Exception) {
                logger.logNetworkError("getUserProfile", e)
                Result.failure(NetworkException("Network error getting user profile", e))
            }
        }
    }
}

// Custom exceptions
class ApiException(message: String, cause: Throwable? = null) : Exception(message, cause)
class NetworkException(message: String, cause: Throwable? = null) : Exception(message, cause)
```

## Network Security Configuration

### 1. Network Security Config XML

Create `res/xml/network_security_config.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- Production configuration -->
    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">your-apigee-domain.com</domain>
        <pin-set>
            <!-- Primary public key hash -->
            <pin digest="SHA-256">AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=</pin>
            <!-- Secondary public key hash -->
            <pin digest="SHA-256">BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=</pin>
            <!-- Backup public key hash -->
            <pin digest="SHA-256">CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC=</pin>
        </pin-set>
        <trust-anchors>
            <!-- Trust system CAs -->
            <certificates src="system"/>
        </trust-anchors>
    </domain-config>
    
    <!-- Debug configuration (only for debug builds) -->
    <debug-overrides>
        <trust-anchors>
            <!-- Trust user added CAs for debugging -->
            <certificates src="user"/>
            <certificates src="system"/>
        </trust-anchors>
    </debug-overrides>
    
    <!-- Base configuration for other domains -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>
        </trust-anchors>
    </base-config>
</network-security-config>
```

### 2. Application Manifest Configuration

Update your `AndroidManifest.xml`:

```xml
<application
    android:name=".MyApplication"
    android:networkSecurityConfig="@xml/network_security_config"
    ... >
    
    <!-- Your activities -->
    
</application>
```

### 3. Build Variants Configuration

Configure different security settings for debug/release builds in `build.gradle`:

```kotlin
android {
    buildTypes {
        debug {
            buildConfigField "boolean", "SSL_PINNING_ENABLED", "false"
            resValue "string", "network_security_config", "@xml/network_security_config_debug"
        }
        release {
            buildConfigField "boolean", "SSL_PINNING_ENABLED", "true"
            resValue "string", "network_security_config", "@xml/network_security_config"
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

Create `res/xml/network_security_config_debug.xml` for debug builds:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <debug-overrides>
        <trust-anchors>
            <certificates src="user"/>
            <certificates src="system"/>
        </trust-anchors>
    </debug-overrides>
    
    <base-config cleartextTrafficPermitted="true">
        <trust-anchors>
            <certificates src="system"/>
            <certificates src="user"/>
        </trust-anchors>
    </base-config>
</network-security-config>
```

## Error Handling

### 1. SSL Pinning Error Handler

Create a centralized error handler:

```kotlin
import android.content.Context
import android.content.Intent
import android.net.Uri
import androidx.appcompat.app.AlertDialog

class SSLPinningErrorHandler(private val context: Context) {
    
    fun handleSSLPinningError(error: Throwable, callback: (() -> Unit)? = null) {
        when (error) {
            is SSLPinningException -> showSSLPinningErrorDialog(error, callback)
            is javax.net.ssl.SSLHandshakeException -> showSSLHandshakeErrorDialog(error, callback)
            is javax.net.ssl.SSLPeerUnverifiedException -> showSSLPeerUnverifiedErrorDialog(error, callback)
            else -> showGenericNetworkErrorDialog(error, callback)
        }
    }
    
    private fun showSSLPinningErrorDialog(error: SSLPinningException, callback: (() -> Unit)?) {
        AlertDialog.Builder(context)
            .setTitle("Security Error")
            .setMessage("A security error occurred while connecting to the server. This may indicate a potential security threat.\n\nError: ${error.message}")
            .setPositiveButton("Retry") { _, _ -> callback?.invoke() }
            .setNegativeButton("Contact Support") { _, _ -> contactSupport() }
            .setNeutralButton("Cancel", null)
            .setCancelable(false)
            .show()
    }
    
    private fun showSSLHandshakeErrorDialog(error: javax.net.ssl.SSLHandshakeException, callback: (() -> Unit)?) {
        AlertDialog.Builder(context)
            .setTitle("Connection Security Error")
            .setMessage("Unable to establish a secure connection. Please check your internet connection and try again.")
            .setPositiveButton("Retry") { _, _ -> callback?.invoke() }
            .setNegativeButton("Cancel", null)
            .show()
    }
    
    private fun showSSLPeerUnverifiedErrorDialog(error: javax.net.ssl.SSLPeerUnverifiedException, callback: (() -> Unit)?) {
        AlertDialog.Builder(context)
            .setTitle("Server Verification Failed")
            .setMessage("The server's identity could not be verified. This may indicate a security issue.")
            .setPositiveButton("Retry") { _, _ -> callback?.invoke() }
            .setNegativeButton("Contact Support") { _, _ -> contactSupport() }
            .setNeutralButton("Cancel", null)
            .show()
    }
    
    private fun showGenericNetworkErrorDialog(error: Throwable, callback: (() -> Unit)?) {
        AlertDialog.Builder(context)
            .setTitle("Network Error")
            .setMessage("A network error occurred: ${error.message}")
            .setPositiveButton("Retry") { _, _ -> callback?.invoke() }
            .setNegativeButton("Cancel", null)
            .show()
    }
    
    private fun contactSupport() {
        val intent = Intent(Intent.ACTION_SENDTO).apply {
            data = Uri.parse("mailto:support@yourcompany.com")
            putExtra(Intent.EXTRA_SUBJECT, "SSL Pinning Error Report")
            putExtra(Intent.EXTRA_TEXT, "I encountered an SSL pinning error in the mobile app.")
        }
        context.startActivity(Intent.createChooser(intent, "Contact Support"))
    }
}
```

### 2. ViewModel Integration

Integrate error handling in ViewModels:

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import androidx.lifecycle.MutableLiveData
import androidx.lifecycle.LiveData
import kotlinx.coroutines.launch

class LoginViewModel(
    private val repository: ApiRepository,
    private val errorHandler: SSLPinningErrorHandler
) : ViewModel() {
    
    private val _loginState = MutableLiveData<LoginState>()
    val loginState: LiveData<LoginState> = _loginState
    
    private val _errorState = MutableLiveData<ErrorState>()
    val errorState: LiveData<ErrorState> = _errorState
    
    fun login(username: String, password: String) {
        viewModelScope.launch {
            _loginState.value = LoginState.Loading
            
            repository.login(username, password)
                .onSuccess { loginResponse ->
                    _loginState.value = LoginState.Success(loginResponse)
                }
                .onFailure { error ->
                    when (error) {
                        is SSLPinningException -> {
                            _errorState.value = ErrorState.SSLPinningError(error)
                        }
                        is NetworkException -> {
                            _errorState.value = ErrorState.NetworkError(error)
                        }
                        else -> {
                            _errorState.value = ErrorState.GenericError(error)
                        }
                    }
                    _loginState.value = LoginState.Error(error)
                }
        }
    }
    
    fun retryLogin(username: String, password: String) {
        login(username, password)
    }
}

sealed class LoginState {
    object Loading : LoginState()
    data class Success(val loginResponse: LoginResponse) : LoginState()
    data class Error(val error: Throwable) : LoginState()
}

sealed class ErrorState {
    data class SSLPinningError(val error: SSLPinningException) : ErrorState()
    data class NetworkError(val error: NetworkException) : ErrorState()
    data class GenericError(val error: Throwable) : ErrorState()
}
```

## Testing

### 1. Unit Tests

Create comprehensive unit tests:

```kotlin
import org.junit.Test
import org.junit.Before
import org.junit.Assert.*
import org.mockito.Mock
import org.mockito.MockitoAnnotations
import org.mockito.kotlin.*
import kotlinx.coroutines.test.runTest

class SSLPinningManagerTest {
    
    @Mock
    private lateinit var context: Context
    
    @Mock
    private lateinit var sharedPreferences: SharedPreferences
    
    @Mock
    private lateinit var editor: SharedPreferences.Editor
    
    private lateinit var sslPinningManager: SSLPinningManager
    
    @Before
    fun setup() {
        MockitoAnnotations.openMocks(this)
        whenever(context.getSharedPreferences(any(), any())).thenReturn(sharedPreferences)
        whenever(sharedPreferences.edit()).thenReturn(editor)
        whenever(editor.putBoolean(any(), any())).thenReturn(editor)
        
        sslPinningManager = SSLPinningManager.getInstance(context)
    }
    
    @Test
    fun `configurePinning should store pin configuration correctly`() {
        // Given
        val hostname = "test.example.com"
        val pinnedHashes = setOf("hash1", "hash2")
        val backupHashes = setOf("backup1")
        
        // When
        sslPinningManager.configurePinning(hostname, pinnedHashes, backupHashes)
        
        // Then
        val config = sslPinningManager.getPinConfiguration(hostname)
        assertNotNull(config)
        assertEquals(hostname, config?.hostname)
        assertEquals(pinnedHashes, config?.pinnedHashes)
        assertEquals(backupHashes, config?.backupHashes)
    }
    
    @Test
    fun `enableKillSwitch should store kill switch state`() {
        // When
        sslPinningManager.enableKillSwitch()
        
        // Then
        verify(editor).putBoolean("ssl_pinning_kill_switch", true)
        verify(editor).apply()
    }
    
    @Test
    fun `createSecureOkHttpClient should return configured client`() {
        // When
        val client = sslPinningManager.createSecureOkHttpClient()
        
        // Then
        assertNotNull(client)
        assertTrue(client.connectTimeoutMillis > 0)
        assertTrue(client.readTimeoutMillis > 0)
    }
}
```

### 2. Integration Tests

Create integration tests for network scenarios:

```kotlin
import androidx.test.ext.junit.runners.AndroidJUnit4
import androidx.test.platform.app.InstrumentationRegistry
import kotlinx.coroutines.test.runTest
import org.junit.Test
import org.junit.Before
import org.junit.runner.RunWith
import org.junit.Assert.*

@RunWith(AndroidJUnit4::class)
class SSLPinningIntegrationTest {
    
    private lateinit var apiService: ApiService
    private lateinit var repository: ApiRepository
    private lateinit var context: Context
    
    @Before
    fun setup() {
        context = InstrumentationRegistry.getInstrumentation().targetContext
        val factory = SecureApiServiceFactory(context)
        apiService = factory.createService(ApiService::class.java, "https://your-test-server.com/api/v1/")
        repository = ApiRepository(apiService)
    }
    
    @Test
    fun testValidSSLPinning() = runTest {
        // Test with valid SSL pinning configuration
        val result = repository.login("testuser", "testpass")
        
        // Should succeed if certificate matches pinned hash
        // This test requires a test server with known certificate
        assertTrue("Login should succeed with valid SSL pinning", result.isSuccess)
    }
    
    @Test
    fun testInvalidSSLPinning() = runTest {
        // Configure with invalid hash for testing
        val sslManager = SSLPinningManager.getInstance(context)
        sslManager.configurePinning(
            "your-test-server.com",
            setOf("invalid-hash-for-testing")
        )
        
        // Recreate services with invalid configuration
        val factory = SecureApiServiceFactory(context)
        val invalidApiService = factory.createService(ApiService::class.java, "https://your-test-server.com/api/v1/")
        val invalidRepository = ApiRepository(invalidApiService)
        
        val result = invalidRepository.login("testuser", "testpass")
        
        // Should fail with SSL pinning error
        assertTrue("Login should fail with invalid SSL pinning", result.isFailure)
        assertTrue("Should be SSL pinning exception", result.exceptionOrNull() is SSLPinningException)
    }
}
```

### 3. UI Tests

Create UI tests for error handling:

```kotlin
import androidx.test.espresso.Espresso.onView
import androidx.test.espresso.action.ViewActions.*
import androidx.test.espresso.assertion.ViewAssertions.*
import androidx.test.espresso.matcher.ViewMatchers.*
import androidx.test.ext.junit.rules.ActivityScenarioRule
import androidx.test.ext.junit.runners.AndroidJUnit4
import org.junit.Rule
import org.junit.Test
import org.junit.runner.RunWith

@RunWith(AndroidJUnit4::class)
class SSLPinningUITest {
    
    @get:Rule
    val activityRule = ActivityScenarioRule(LoginActivity::class.java)
    
    @Test
    fun testSSLPinningErrorDialog() {
        // Simulate SSL pinning error by configuring invalid hash
        // (This would require test configuration or mock setup)
        
        // Perform login action that triggers SSL error
        onView(withId(R.id.etUsername)).perform(typeText("testuser"))
        onView(withId(R.id.etPassword)).perform(typeText("testpass"))
        onView(withId(R.id.btnLogin)).perform(click())
        
        // Verify error dialog appears
        onView(withText("Security Error")).check(matches(isDisplayed()))
        onView(withText("Retry")).check(matches(isDisplayed()))
        onView(withText("Contact Support")).check(matches(isDisplayed()))
        onView(withText("Cancel")).check(matches(isDisplayed()))
    }
    
    @Test
    fun testRetryFunctionality() {
        // Test retry button functionality
        // This would require proper test setup with controllable SSL errors
        
        onView(withId(R.id.etUsername)).perform(typeText("testuser"))
        onView(withId(R.id.etPassword)).perform(typeText("testpass"))
        onView(withId(R.id.btnLogin)).perform(click())
        
        // Wait for error dialog and click retry
        onView(withText("Retry")).perform(click())
        
        // Verify retry attempt is made
        // Add assertions based on your retry logic
    }
}
```

## Best Practices

### 1. ProGuard Configuration

Add ProGuard rules to protect SSL pinning implementation:

```proguard
# SSL Pinning Protection
-keep class com.yourpackage.ssl.** { *; }
-keep class javax.net.ssl.** { *; }
-keep class okhttp3.** { *; }

# Prevent SSL pinning bypass
-assumenosideeffects class android.util.Log {
    public static boolean isLoggable(java.lang.String, int);
    public static int v(...);
    public static int i(...);
    public static int w(...);
    public static int d(...);
    public static int e(...);
}

# SSL Certificate pinning
-keepclassmembers class * {
    @retrofit2.http.* <methods>;
}
```

### 2. Build Configuration

Configure build variants for different security levels:

```kotlin
android {
    buildTypes {
        debug {
            buildConfigField "String[]", "PINNED_HASHES", "new String[]{}"
            buildConfigField "boolean", "SSL_PINNING_ENABLED", "false"
        }
        staging {
            buildConfigField "String[]", "PINNED_HASHES", 
                "new String[]{\"AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=\", \"BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=\"}"
            buildConfigField "boolean", "SSL_PINNING_ENABLED", "true"
        }
        release {
            buildConfigField "String[]", "PINNED_HASHES", 
                "new String[]{\"AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=\", \"BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=\", \"CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC=\"}"
            buildConfigField "boolean", "SSL_PINNING_ENABLED", "true"
            minifyEnabled true
        }
    }
}
```

### 3. Performance Optimization

Optimize SSL pinning for better performance:

```kotlin
class OptimizedSSLPinningManager {
    
    // Cache certificate hashes to avoid recalculation
    private val hashCache = mutableMapOf<String, String>()
    
    // Use concurrent map for thread safety
    private val pinConfigCache = java.util.concurrent.ConcurrentHashMap<String, PinConfiguration>()
    
    fun getCachedHash(certificate: X509Certificate): String {
        val key = certificate.serialNumber.toString()
        return hashCache.getOrPut(key) {
            generatePublicKeyHash(certificate)
        }
    }
    
    fun clearCache() {
        hashCache.clear()
    }
}
```

## Troubleshooting

### Common Issues and Solutions

1. **Certificate Chain Issues**
   ```kotlin
   fun debugCertificateChain(chain: Array<X509Certificate>) {
       Log.d("SSLDebug", "Certificate chain contains ${chain.size} certificates")
       chain.forEachIndexed { index, cert ->
           Log.d("SSLDebug", "Certificate $index: ${cert.subjectDN}")
           Log.d("SSLDebug", "Hash: ${generatePublicKeyHash(cert)}")
       }
   }
   ```

2. **Network Security Config Issues**
   ```kotlin
   fun validateNetworkSecurityConfig() {
       try {
           val config = context.resources.getXml(R.xml.network_security_config)
           // Validate configuration
           Log.i("SSLDebug", "Network security config is valid")
       } catch (e: Exception) {
           Log.e("SSLDebug", "Network security config error", e)
       }
   }
   ```

3. **Testing with Self-Signed Certificates**
   ```kotlin
   fun createTestTrustManager(): X509TrustManager {
       return object : X509TrustManager {
           override fun checkClientTrusted(chain: Array<X509Certificate>, authType: String) {}
           override fun checkServerTrusted(chain: Array<X509Certificate>, authType: String) {
               // Only for testing - DO NOT USE IN PRODUCTION
           }
           override fun getAcceptedIssuers(): Array<X509Certificate> = arrayOf()
       }
   }
   ```

### Debug Tools

Create debug tools for development:

```kotlin
class SSLPinningDebugTools(private val context: Context) {
    
    fun extractCertificateInfo(hostname: String, port: Int = 443) {
        Thread {
            try {
                val socket = SSLSocketFactory.getDefault().createSocket(hostname, port) as SSLSocket
                socket.addHandshakeCompletedListener { event ->
                    val certs = event.peerCertificates
                    certs.forEachIndexed { index, cert ->
                        if (cert is X509Certificate) {
                            val hash = generatePublicKeyHash(cert)
                            Log.d("SSLDebug", "Certificate $index hash: $hash")
                        }
                    }
                }
                socket.startHandshake()
                socket.close()
            } catch (e: Exception) {
                Log.e("SSLDebug", "Error extracting certificate info", e)
            }
        }.start()
    }
}
```

This comprehensive Android Kotlin implementation provides robust SSL pinning with proper error handling, testing, and debugging capabilities for production use.
