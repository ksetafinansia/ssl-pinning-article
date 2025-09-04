# Section 6: Platform Implementation Summary

## Overview
This section provides detailed implementation examples for SSL pinning across different platforms in the Kredit Plus technology stack. Each implementation focuses on pinning only `apigee.kreditplus.com` as established in Section 5.

## Detailed Implementation Examples

### iOS Implementation (Swift)

#### Step 1: Create Certificate Pinning Manager
```swift
import Foundation
import Security

class CertificatePinner: NSObject {
    private let pinnedHost = "apigee.kreditplus.com"
    private let pinnedHashes: Set<String> = [
        "your_primary_pin_hash_here",
        "your_backup_pin_hash_here"
    ]
    
    func createPinnedURLSession() -> URLSession {
        let config = URLSessionConfiguration.default
        let session = URLSession(
            configuration: config,
            delegate: self,
            delegateQueue: nil
        )
        return session
    }
    
    private func sha256(data: Data) -> String {
        var hash = [UInt8](repeating: 0, count: Int(CC_SHA256_DIGEST_LENGTH))
        data.withUnsafeBytes {
            _ = CC_SHA256($0.baseAddress, CC_LONG(data.count), &hash)
        }
        return Data(hash).base64EncodedString()
    }
}

// MARK: - URLSessionDelegate
extension CertificatePinner: URLSessionDelegate {
    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        
        // Only apply pinning to our target host
        guard challenge.protectionSpace.host == pinnedHost else {
            completionHandler(.performDefaultHandling, nil)
            return
        }
        
        // Get server trust
        guard let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        // Evaluate server trust
        let policy = SecPolicyCreateSSL(true, pinnedHost as CFString)
        SecTrustSetPolicies(serverTrust, policy)
        
        var result: SecTrustResultType = .invalid
        let status = SecTrustEvaluate(serverTrust, &result)
        
        guard status == errSecSuccess,
              result == .unspecified || result == .proceed else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        // Extract and validate public key
        guard let serverCert = SecTrustGetCertificateAtIndex(serverTrust, 0),
              let serverPublicKey = SecCertificateCopyKey(serverCert),
              let serverPublicKeyData = SecKeyCopyExternalRepresentation(serverPublicKey, nil) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        let serverKeyHash = sha256(data: serverPublicKeyData as Data)
        
        // Check if server key hash matches any of our pinned hashes
        if pinnedHashes.contains(serverKeyHash) {
            let credential = URLCredential(trust: serverTrust)
            completionHandler(.useCredential, credential)
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}
```

#### Step 2: Implement API Client
```swift
class APIClient {
    private let pinner = CertificatePinner()
    private lazy var session = pinner.createPinnedURLSession()
    
    func makeRequest(to endpoint: String) async throws -> Data {
        guard let url = URL(string: "https://apigee.kreditplus.com/\(endpoint)") else {
            throw APIError.invalidURL
        }
        
        let (data, response) = try await session.data(from: url)
        
        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw APIError.requestFailed
        }
        
        return data
    }
}

enum APIError: Error {
    case invalidURL
    case requestFailed
}
```

### Android Implementation (Kotlin)

#### Step 1: Create Certificate Pinning Configuration
```kotlin
import okhttp3.CertificatePinner
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import java.util.concurrent.TimeUnit

class PinnedHttpClient {
    companion object {
        private const val PINNED_HOST = "apigee.kreditplus.com"
        private const val PRIMARY_PIN = "sha256/your_primary_pin_hash_here"
        private const val BACKUP_PIN = "sha256/your_backup_pin_hash_here"
        
        val instance: OkHttpClient by lazy {
            createPinnedClient()
        }
        
        private fun createPinnedClient(): OkHttpClient {
            val certificatePinner = CertificatePinner.Builder()
                .add(PINNED_HOST, PRIMARY_PIN)
                .add(PINNED_HOST, BACKUP_PIN)
                .build()
            
            val loggingInterceptor = HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG) {
                    HttpLoggingInterceptor.Level.BODY
                } else {
                    HttpLoggingInterceptor.Level.NONE
                }
            }
            
            return OkHttpClient.Builder()
                .certificatePinner(certificatePinner)
                .addInterceptor(loggingInterceptor)
                .connectTimeout(30, TimeUnit.SECONDS)
                .readTimeout(30, TimeUnit.SECONDS)
                .writeTimeout(30, TimeUnit.SECONDS)
                .build()
        }
    }
}
```

#### Step 2: Implement API Service
```kotlin
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import retrofit2.http.GET
import retrofit2.http.Path

interface ApiService {
    @GET("api/v1/{endpoint}")
    suspend fun getData(@Path("endpoint") endpoint: String): ApiResponse
    
    companion object {
        private const val BASE_URL = "https://apigee.kreditplus.com/"
        
        val instance: ApiService by lazy {
            Retrofit.Builder()
                .baseUrl(BASE_URL)
                .client(PinnedHttpClient.instance)
                .addConverterFactory(GsonConverterFactory.create())
                .build()
                .create(ApiService::class.java)
        }
    }
}

data class ApiResponse(
    val status: String,
    val data: Any?
)
```

#### Step 3: Repository Implementation
```kotlin
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext

class ApiRepository {
    private val apiService = ApiService.instance
    
    suspend fun fetchData(endpoint: String): Result<ApiResponse> {
        return withContext(Dispatchers.IO) {
            try {
                val response = apiService.getData(endpoint)
                Result.success(response)
            } catch (e: Exception) {
                // Log pinning failures for monitoring
                if (e.message?.contains("Certificate pinning failure") == true) {
                    // Report pinning failure to analytics
                    reportPinningFailure(e)
                }
                Result.failure(e)
            }
        }
    }
    
    private fun reportPinningFailure(exception: Exception) {
        // Implementation for reporting pinning failures
        // This could be Firebase Crashlytics, custom analytics, etc.
    }
}
```

### Flutter Implementation

#### Step 1: Add Dependencies
```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  dio: ^5.3.2
  dio_certificate_pinning: ^3.0.3
```

#### Step 2: Create Pinned HTTP Client
```dart
import 'package:dio/dio.dart';
import 'package:dio_certificate_pinning/dio_certificate_pinning.dart';

class PinnedHttpClient {
  static const String _pinnedHost = 'apigee.kreditplus.com';
  static const List<String> _pinnedHashes = [
    'your_primary_pin_hash_here',
    'your_backup_pin_hash_here',
  ];
  
  static Dio? _instance;
  
  static Dio get instance {
    _instance ??= _createPinnedClient();
    return _instance!;
  }
  
  static Dio _createPinnedClient() {
    final dio = Dio();
    
    // Add certificate pinning interceptor
    dio.interceptors.add(
      CertificatePinningInterceptor(
        allowedSHAFingerprints: _pinnedHashes,
      ),
    );
    
    // Add logging in debug mode
    if (kDebugMode) {
      dio.interceptors.add(
        LogInterceptor(
          requestBody: true,
          responseBody: true,
          requestHeader: true,
          responseHeader: true,
        ),
      );
    }
    
    // Base configuration
    dio.options = BaseOptions(
      baseUrl: 'https://$_pinnedHost/',
      connectTimeout: const Duration(seconds: 30),
      receiveTimeout: const Duration(seconds: 30),
      sendTimeout: const Duration(seconds: 30),
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
    );
    
    return dio;
  }
}
```

#### Step 3: Implement API Service
```dart
class ApiService {
  static final Dio _dio = PinnedHttpClient.instance;
  
  static Future<Map<String, dynamic>> getData(String endpoint) async {
    try {
      final response = await _dio.get('/api/v1/$endpoint');
      
      if (response.statusCode == 200) {
        return response.data;
      } else {
        throw ApiException('Request failed with status: ${response.statusCode}');
      }
    } on DioException catch (e) {
      // Handle certificate pinning failures
      if (e.type == DioExceptionType.badCertificate) {
        // Report pinning failure
        _reportPinningFailure(e);
        throw PinningException('Certificate pinning validation failed');
      }
      
      throw ApiException('Network error: ${e.message}');
    } catch (e) {
      throw ApiException('Unexpected error: $e');
    }
  }
  
  static void _reportPinningFailure(DioException exception) {
    // Implementation for reporting pinning failures
    // Could be Firebase Analytics, Crashlytics, etc.
    print('Certificate pinning failure reported: ${exception.message}');
  }
}

class ApiException implements Exception {
  final String message;
  ApiException(this.message);
  
  @override
  String toString() => 'ApiException: $message';
}

class PinningException implements Exception {
  final String message;
  PinningException(this.message);
  
  @override
  String toString() => 'PinningException: $message';
}
```

#### Step 4: Repository Implementation
```dart
class ApiRepository {
  Future<Result<Map<String, dynamic>>> fetchData(String endpoint) async {
    try {
      final data = await ApiService.getData(endpoint);
      return Result.success(data);
    } on PinningException catch (e) {
      // Handle pinning failures specifically
      return Result.failure(e.message);
    } on ApiException catch (e) {
      return Result.failure(e.message);
    } catch (e) {
      return Result.failure('Unknown error occurred');
    }
  }
}

class Result<T> {
  final T? data;
  final String? error;
  final bool isSuccess;
  
  Result.success(this.data) : error = null, isSuccess = true;
  Result.failure(this.error) : data = null, isSuccess = false;
}
```

### Nuxt.js Implementation

#### Step 1: Create Certificate Pinning Module
```javascript
// plugins/certificate-pinning.js
import https from 'https'
import crypto from 'crypto'

const PINNED_HOST = 'apigee.kreditplus.com'
const PINNED_HASHES = [
  'your_primary_pin_hash_here',
  'your_backup_pin_hash_here'
]

function createPinnedAgent() {
  return new https.Agent({
    checkServerIdentity: (hostname, cert) => {
      // Only apply pinning to our target host
      if (hostname !== PINNED_HOST) {
        return undefined // Use default validation
      }
      
      // Extract public key and create hash
      const publicKey = cert.pubkey
      const hash = crypto.createHash('sha256').update(publicKey).digest('base64')
      
      // Check if hash matches any pinned hash
      if (!PINNED_HASHES.includes(hash)) {
        throw new Error(`Certificate pinning failure for ${hostname}. Hash: ${hash}`)
      }
      
      return undefined // Validation passed
    },
    rejectUnauthorized: true
  })
}

export default createPinnedAgent
```

#### Step 2: Configure Axios with Pinning
```javascript
// plugins/api-client.js
import axios from 'axios'
import createPinnedAgent from './certificate-pinning'

const pinnedAgent = createPinnedAgent()

const apiClient = axios.create({
  baseURL: 'https://apigee.kreditplus.com/api/v1',
  timeout: 30000,
  httpsAgent: pinnedAgent,
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json'
  }
})

// Request interceptor
apiClient.interceptors.request.use(
  (config) => {
    // Add any authentication tokens here
    if (process.server) {
      console.log(`Making pinned request to: ${config.url}`)
    }
    return config
  },
  (error) => {
    return Promise.reject(error)
  }
)

// Response interceptor
apiClient.interceptors.response.use(
  (response) => {
    return response
  },
  (error) => {
    // Handle certificate pinning failures
    if (error.message && error.message.includes('Certificate pinning failure')) {
      console.error('Certificate pinning validation failed:', error.message)
      
      // Report pinning failure to monitoring system
      reportPinningFailure(error)
    }
    
    return Promise.reject(error)
  }
)

function reportPinningFailure(error) {
  // Implementation for server-side pinning failure reporting
  // Could be logging service, monitoring system, etc.
  console.error('Pinning failure reported:', error.message)
}

export default apiClient
```

#### Step 3: Nuxt Configuration
```javascript
// nuxt.config.js
export default {
  // ... other config
  plugins: [
    '~/plugins/api-client.js'
  ],
  
  // Server-side configuration
  serverMiddleware: [
    '~/server-middleware/api-proxy.js'
  ],
  
  // Runtime config for different environments
  runtimeConfig: {
    // Private keys (only available on server-side)
    apiBaseUrl: process.env.API_BASE_URL || 'https://apigee.kreditplus.com',
    
    // Public keys (exposed to client-side)
    public: {
      apiBaseUrl: process.env.API_BASE_URL || 'https://apigee.kreditplus.com'
    }
  }
}
```

#### Step 4: API Service Implementation
```javascript
// services/api.service.js
import apiClient from '~/plugins/api-client'

export class ApiService {
  static async getData(endpoint) {
    try {
      const response = await apiClient.get(`/${endpoint}`)
      return {
        success: true,
        data: response.data
      }
    } catch (error) {
      console.error('API request failed:', error.message)
      
      return {
        success: false,
        error: error.message,
        isPinningFailure: error.message.includes('Certificate pinning failure')
      }
    }
  }
  
  static async postData(endpoint, data) {
    try {
      const response = await apiClient.post(`/${endpoint}`, data)
      return {
        success: true,
        data: response.data
      }
    } catch (error) {
      console.error('API request failed:', error.message)
      
      return {
        success: false,
        error: error.message,
        isPinningFailure: error.message.includes('Certificate pinning failure')
      }
    }
  }
}
```

#### Step 5: Server Middleware for SSR
```javascript
// server-middleware/api-proxy.js
import { ApiService } from '~/services/api.service'

export default async function (req, res, next) {
  // Handle API routes on server-side with pinning
  if (req.url.startsWith('/api/')) {
    const endpoint = req.url.replace('/api/', '')
    
    try {
      const result = await ApiService.getData(endpoint)
      
      if (result.success) {
        res.setHeader('Content-Type', 'application/json')
        res.end(JSON.stringify(result.data))
      } else {
        res.statusCode = 500
        res.end(JSON.stringify({ error: result.error }))
      }
    } catch (error) {
      res.statusCode = 500
      res.end(JSON.stringify({ error: 'Internal server error' }))
    }
  } else {
    next()
  }
}
```

## Implementation Best Practices

### Common Configuration Management
```javascript
// shared/config.js - Common configuration across platforms
export const SSL_PINNING_CONFIG = {
  HOST: 'apigee.kreditplus.com',
  PRIMARY_PIN: 'your_primary_pin_hash_here',
  BACKUP_PIN: 'your_backup_pin_hash_here',
  STAGING_HOST: 'apigee.kbfinansia.com',
  STAGING_PIN: 'your_staging_pin_hash_here'
}
```

### Error Handling and Monitoring
```javascript
// All platforms should implement similar error handling
function handlePinningError(error, platform) {
  const errorData = {
    platform: platform,
    timestamp: new Date().toISOString(),
    error: error.message,
    host: 'apigee.kreditplus.com'
  }
  
  // Send to monitoring system
  // Analytics.track('ssl_pinning_failure', errorData)
}
```

### Testing Strategy
1. **Unit Tests**: Test pinning logic with mock certificates
2. **Integration Tests**: Test against staging environment
3. **E2E Tests**: Verify complete flow with pinning enabled
4. **Negative Tests**: Verify rejection of invalid certificates

### Performance Considerations
- **Caching**: Cache validated certificates where possible
- **Timeout**: Set appropriate timeouts for pinning validation
- **Fallback**: Implement graceful degradation if needed
- **Monitoring**: Track performance impact of pinning validation
