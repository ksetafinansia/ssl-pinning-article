SSL pinning architecture is
([Clients (mobile/cms)] -> [ApiGee] -> [Backend Services]) 

Client -> apigee is external connections
ApiGee -> backend service is internal connections

Goals is to securing the connection throught ssl pinning.

Step1:  Obtain the Apigee server's certificate information
Explain how to do this step by step technically
Use the most robust and flexible pinning method so public key remains the same event if certificate reissued.
Output must be easy to implement and can be stored flexible. This is the value you will "pin" in your clients.

Step 2: Implement pinning in clients
Give sample implementation Mobile in Swift, Kotlin, and Flutter.
Give sample implementation CMS 

Best practices and considerations
- Use public key pinning: Pinning the public key hash is recommended over certificate pinning because it doesn't require a client-side update if the certificate is renewed, as long as the underlying key pair doesn't change.
- Pin backup keys: For extra resilience, pin multiple public key hashes—one for the current key and one for a backup key. This allows you to replace the primary key without a full client update, as long as you plan ahead.
- Include a transition plan: When you need to rotate your keys, ensure a new app version with the updated hashes is released and widely adopted before the old key is deprecated.
- Do not pin a wildcard certificate: Pinning should be done for a specific hostname.
- Integrate into CI/CD: Automate the process of extracting the public key hash from your certificate and injecting it into your client builds to minimize human error.
- Internal connection security (ApiGee -> Backend Services): Your internal connection from Apigee to the backend services is more effectively secured using mutual TLS (mTLS) and/or network-level access controls (e.g., allowlisting). This ensures that only trusted Apigee instances can connect to your backend, and the client-facing pinning is an independent security measure.


Created Implementation Files:
📱 iOS Swift Implementation (ssl-pinning-ios-swift.md)

URLSession with custom TrustManager
Certificate pinning using SHA-256 hashes
Complete API client with error handling
Unit and integration tests
Security best practices
🤖 Android Kotlin Implementation (ssl-pinning-android-kotlin.md)

OkHttp client with CertificatePinner
Retrofit integration for API calls
Manual X509TrustManager option
Network Security Configuration
Comprehensive testing setup
🐦 Flutter Dart Implementation (ssl-pinning-flutter-dart.md)

Two implementation options (plugin + manual)
Dio HTTP client with interceptors
Provider pattern for state management
Cross-platform certificate validation
Integration with Flutter UI
🌐 Nuxt.js CMS Implementation (ssl-pinning-nuxtjs-cms.md)

Server-side certificate validation
Client-side security composables
CMS dashboard with security monitoring
Security middleware and headers
Browser-compatible approach
Key Features Across All Platforms:
✅ Consistent Pin Format: All use the same Base64 SPKI hashes from Step 1 ✅ Error Handling: Dedicated handling for SSL pinning failures ✅ Kill Switch: Remote configuration to disable pinning ✅ Testing: Unit and integration tests for each platform ✅ Best Practices: Security considerations and troubleshooting ✅ Real-world Usage: Complete examples with UI integration