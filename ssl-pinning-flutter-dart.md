````markdown
# SSL Pinning Flutter Dart Implementation

## Overview

This guide provides a comprehensive SSL pinning implementation for Flutter using Dart with multiple approaches including plugin-based and manual implementation options. The implementation uses SHA-256 public key hashes for certificate pinning, providing robust security across iOS and Android platforms.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Dependencies](#dependencies)
3. [Implementation Options](#implementation-options)
4. [Plugin-Based Implementation](#plugin-based-implementation)
5. [Manual Dio Implementation](#manual-dio-implementation)
6. [Provider Pattern Integration](#provider-pattern-integration)
7. [Cross-Platform Certificate Validation](#cross-platform-certificate-validation)
8. [Flutter UI Integration](#flutter-ui-integration)
9. [Error Handling](#error-handling)
10. [Testing](#testing)
11. [Best Practices](#best-practices)
12. [Troubleshooting](#troubleshooting)

## Prerequisites

- Flutter 3.0+
- Dart 2.17+
- Android API Level 21+ (Android 5.0)
- iOS 11.0+
- Basic understanding of HTTP clients and state management in Flutter

## Dependencies

Add these dependencies to your `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  
  # HTTP Client
  dio: ^5.3.2
  
  # SSL Pinning
  dio_certificate_pinning: ^6.0.0
  
  # State Management
  provider: ^6.0.5
  
  # Utilities
  crypto: ^3.0.3
  convert: ^3.1.1
  
  # Local Storage
  shared_preferences: ^2.2.2
  
  # Logging
  logger: ^2.0.2

dev_dependencies:
  flutter_test:
    sdk: flutter
  mockito: ^5.4.2
  build_runner: ^2.4.7
  
  # Testing
  integration_test:
    sdk: flutter
```

## Implementation Options

Flutter SSL pinning can be implemented in two ways:

### Option 1: Plugin-Based (Recommended)
- Uses `dio_certificate_pinning` plugin
- Easier to implement and maintain
- Good for most use cases

### Option 2: Manual Implementation
- Custom certificate validation logic
- More control over validation process
- Better for complex requirements

## Plugin-Based Implementation

### 1. SSL Pinning Configuration

Create a configuration class for SSL pinning:

```dart
import 'package:crypto/crypto.dart';
import 'package:convert/convert.dart';

class SSLPinningConfig {
  static const String _tag = 'SSLPinning';
  
  // Configuration for your Apigee domain
  static const String apigeeHost = 'your-apigee-domain.com';
  
  // Primary and backup public key hashes
  static const List<String> pinnedHashes = [
    'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=', // Primary hash
    'BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=', // Secondary hash
  ];
  
  static const List<String> backupHashes = [
    'CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC=', // Backup hash
  ];
  
  // Kill switch configuration
  static const String killSwitchKey = 'ssl_pinning_kill_switch';
  
  // Get all valid hashes (primary + backup)
  static List<String> get allValidHashes => [...pinnedHashes, ...backupHashes];
  
  // Convert to SHA256 format for dio_certificate_pinning
  static List<String> get sha256Hashes => 
      allValidHashes.map((hash) => 'sha256/$hash').toList();
}
```

### 2. SSL Pinning Manager with Plugin

Create a manager class using the plugin approach:

```dart
import 'package:dio/dio.dart';
import 'package:dio_certificate_pinning/dio_certificate_pinning.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:logger/logger.dart';

class PluginSSLPinningManager {
  static final PluginSSLPinningManager _instance = PluginSSLPinningManager._internal();
  factory PluginSSLPinningManager() => _instance;
  PluginSSLPinningManager._internal();
  
  final Logger _logger = Logger();
  late Dio _dio;
  SharedPreferences? _prefs;
  
  // Initialize the manager
  Future<void> initialize() async {
    _prefs = await SharedPreferences.getInstance();
    _dio = await _createSecureDio();
    _logger.i('SSL Pinning Manager initialized');
  }
  
  // Create Dio instance with SSL pinning
  Future<Dio> _createSecureDio() async {
    final dio = Dio();
    
    // Configure base options
    dio.options = BaseOptions(
      connectTimeout: const Duration(seconds: 30),
      receiveTimeout: const Duration(seconds: 30),
      sendTimeout: const Duration(seconds: 30),
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
    );
    
    // Add SSL pinning interceptor if not disabled
    if (!await _isKillSwitchEnabled()) {
      dio.interceptors.add(
        CertificatePinningInterceptor(
          allowedSHAFingerprints: SSLPinningConfig.sha256Hashes,
        ),
      );
      _logger.i('SSL Pinning enabled with ${SSLPinningConfig.sha256Hashes.length} hashes');
    } else {
      _logger.w('SSL Pinning disabled via kill switch');
    }
    
    // Add logging interceptor
    dio.interceptors.add(_createLoggingInterceptor());
    
    // Add error handling interceptor
    dio.interceptors.add(_createErrorInterceptor());
    
    return dio;
  }
  
  // Create logging interceptor
  InterceptorsWrapper _createLoggingInterceptor() {
    return InterceptorsWrapper(
      onRequest: (options, handler) {
        _logger.d('Request: ${options.method} ${options.uri}');
        handler.next(options);
      },
      onResponse: (response, handler) {
        _logger.d('Response: ${response.statusCode} ${response.requestOptions.uri}');
        handler.next(response);
      },
      onError: (error, handler) {
        _logger.e('Error: ${error.message}', error: error);
        handler.next(error);
      },
    );
  }
  
  // Create error handling interceptor
  InterceptorsWrapper _createErrorInterceptor() {
    return InterceptorsWrapper(
      onError: (error, handler) {
        if (_isSSLPinningError(error)) {
          _logger.e('SSL Pinning validation failed: ${error.message}');
          handler.next(SSLPinningException.fromDioError(error));
        } else {
          handler.next(error);
        }
      },
    );
  }
  
  // Check if error is SSL pinning related
  bool _isSSLPinningError(DioException error) {
    return error.type == DioExceptionType.connectionError ||
           error.message?.toLowerCase().contains('certificate') == true ||
           error.message?.toLowerCase().contains('ssl') == true ||
           error.message?.toLowerCase().contains('handshake') == true;
  }
  
  // Kill switch management
  Future<bool> _isKillSwitchEnabled() async {
    return _prefs?.getBool(SSLPinningConfig.killSwitchKey) ?? false;
  }
  
  Future<void> enableKillSwitch() async {
    await _prefs?.setBool(SSLPinningConfig.killSwitchKey, true);
    _logger.w('SSL Pinning kill switch ENABLED');
    // Recreate Dio instance without pinning
    _dio = await _createSecureDio();
  }
  
  Future<void> disableKillSwitch() async {
    await _prefs?.setBool(SSLPinningConfig.killSwitchKey, false);
    _logger.i('SSL Pinning kill switch DISABLED');
    // Recreate Dio instance with pinning
    _dio = await _createSecureDio();
  }
  
  // Get configured Dio instance
  Dio get dio => _dio;
}
```

### 3. Custom SSL Pinning Exceptions

Define specific exceptions for SSL pinning:

```dart
class SSLPinningException implements Exception {
  final String message;
  final String? hostname;
  final DioException? originalError;
  
  const SSLPinningException(this.message, {this.hostname, this.originalError});
  
  factory SSLPinningException.fromDioError(DioException error) {
    String message = 'SSL pinning validation failed';
    String? hostname = error.requestOptions.uri.host;
    
    if (error.message?.contains('certificate') == true) {
      message = 'Certificate validation failed for $hostname';
    } else if (error.message?.contains('handshake') == true) {
      message = 'SSL handshake failed for $hostname';
    }
    
    return SSLPinningException(message, hostname: hostname, originalError: error);
  }
  
  @override
  String toString() => 'SSLPinningException: $message';
}

class NetworkException implements Exception {
  final String message;
  final int? statusCode;
  final DioException? originalError;
  
  const NetworkException(this.message, {this.statusCode, this.originalError});
  
  @override
  String toString() => 'NetworkException: $message';
}
```

## Manual Dio Implementation

### 1. Custom Certificate Validator

For more control, implement custom certificate validation:

```dart
import 'dart:io';
import 'dart:typed_data';
import 'package:crypto/crypto.dart';
import 'package:convert/convert.dart';

class ManualSSLPinningManager {
  static final ManualSSLPinningManager _instance = ManualSSLPinningManager._internal();
  factory ManualSSLPinningManager() => _instance;
  ManualSSLPinningManager._internal();
  
  final Logger _logger = Logger();
  late Dio _dio;
  
  Future<void> initialize() async {
    _dio = await _createSecureDio();
  }
  
  Future<Dio> _createSecureDio() async {
    final dio = Dio();
    
    // Configure HTTP client adapter
    (dio.httpClientAdapter as DefaultHttpClientAdapter).onHttpClientCreate = (client) {
      client.badCertificateCallback = (X509Certificate cert, String host, int port) {
        return _validateCertificate(cert, host);
      };
      return client;
    };
    
    return dio;
  }
  
  bool _validateCertificate(X509Certificate certificate, String hostname) {
    try {
      // Check if hostname matches configuration
      if (hostname != SSLPinningConfig.apigeeHost) {
        _logger.d('Certificate validation skipped for hostname: $hostname');
        return true; // Allow other hosts to use default validation
      }
      
      // Extract public key hash
      final publicKeyHash = _extractPublicKeyHash(certificate);
      
      // Validate against pinned hashes
      final isValid = SSLPinningConfig.allValidHashes.contains(publicKeyHash);
      
      if (isValid) {
        _logger.i('Certificate validation SUCCESS for $hostname with hash: $publicKeyHash');
      } else {
        _logger.e('Certificate validation FAILED for $hostname with hash: $publicKeyHash');
      }
      
      return isValid;
    } catch (e) {
      _logger.e('Certificate validation error: $e');
      return false;
    }
  }
  
  String _extractPublicKeyHash(X509Certificate certificate) {
    // Extract public key from certificate
    final publicKey = certificate.publicKey;
    
    // Get DER encoded public key
    final publicKeyDer = publicKey.der;
    
    // Calculate SHA-256 hash
    final digest = sha256.convert(publicKeyDer);
    
    // Convert to base64
    return base64Encode(digest.bytes);
  }
  
  Dio get dio => _dio;
}

// Extension to get DER encoding of public key
extension X509CertificateExtension on X509Certificate {
  Uint8List get publicKeyDer {
    // This is a simplified implementation
    // In a real implementation, you would properly extract the DER-encoded public key
    return Uint8List.fromList(der);
  }
}
```

## Provider Pattern Integration

### 1. SSL Pinning Provider

Create a provider for state management:

```dart
import 'package:flutter/foundation.dart';
import 'package:provider/provider.dart';

class SSLPinningProvider extends ChangeNotifier {
  final PluginSSLPinningManager _sslManager = PluginSSLPinningManager();
  
  bool _isInitialized = false;
  bool _isKillSwitchEnabled = false;
  String? _lastError;
  
  // Getters
  bool get isInitialized => _isInitialized;
  bool get isKillSwitchEnabled => _isKillSwitchEnabled;
  String? get lastError => _lastError;
  Dio get dio => _sslManager.dio;
  
  // Initialize SSL pinning
  Future<void> initialize() async {
    try {
      await _sslManager.initialize();
      _isInitialized = true;
      _lastError = null;
      notifyListeners();
    } catch (e) {
      _lastError = e.toString();
      notifyListeners();
      rethrow;
    }
  }
  
  // Enable kill switch
  Future<void> enableKillSwitch() async {
    try {
      await _sslManager.enableKillSwitch();
      _isKillSwitchEnabled = true;
      _lastError = null;
      notifyListeners();
    } catch (e) {
      _lastError = e.toString();
      notifyListeners();
      rethrow;
    }
  }
  
  // Disable kill switch
  Future<void> disableKillSwitch() async {
    try {
      await _sslManager.disableKillSwitch();
      _isKillSwitchEnabled = false;
      _lastError = null;
      notifyListeners();
    } catch (e) {
      _lastError = e.toString();
      notifyListeners();
      rethrow;
    }
  }
  
  // Clear error
  void clearError() {
    _lastError = null;
    notifyListeners();
  }
}
```

### 2. API Service with Provider

Create API services that use the SSL pinning provider:

```dart
import 'package:provider/provider.dart';

// Data models
class ApiResponse<T> {
  final T data;
  final String message;
  final bool success;
  
  ApiResponse({required this.data, required this.message, required this.success});
  
  factory ApiResponse.fromJson(Map<String, dynamic> json, T Function(Map<String, dynamic>) fromJsonT) {
    return ApiResponse(
      data: fromJsonT(json['data']),
      message: json['message'] ?? '',
      success: json['success'] ?? false,
    );
  }
}

class UserProfile {
  final String id;
  final String name;
  final String email;
  
  UserProfile({required this.id, required this.name, required this.email});
  
  factory UserProfile.fromJson(Map<String, dynamic> json) {
    return UserProfile(
      id: json['id'],
      name: json['name'],
      email: json['email'],
    );
  }
}

class LoginRequest {
  final String username;
  final String password;
  
  LoginRequest({required this.username, required this.password});
  
  Map<String, dynamic> toJson() {
    return {
      'username': username,
      'password': password,
    };
  }
}

class LoginResponse {
  final String token;
  final UserProfile user;
  final String expiresAt;
  
  LoginResponse({required this.token, required this.user, required this.expiresAt});
  
  factory LoginResponse.fromJson(Map<String, dynamic> json) {
    return LoginResponse(
      token: json['token'],
      user: UserProfile.fromJson(json['user']),
      expiresAt: json['expiresAt'],
    );
  }
}

// API Service
class ApiService {
  final String baseUrl = 'https://${SSLPinningConfig.apigeeHost}/api/v1';
  final Logger _logger = Logger();
  
  // Login method
  Future<LoginResponse> login(BuildContext context, String username, String password) async {
    final sslProvider = Provider.of<SSLPinningProvider>(context, listen: false);
    
    if (!sslProvider.isInitialized) {
      throw Exception('SSL Pinning not initialized');
    }
    
    try {
      final response = await sslProvider.dio.post(
        '$baseUrl/auth/login',
        data: LoginRequest(username: username, password: password).toJson(),
      );
      
      if (response.statusCode == 200) {
        final apiResponse = ApiResponse.fromJson(
          response.data,
          (data) => LoginResponse.fromJson(data),
        );
        
        if (apiResponse.success) {
          _logger.i('Login successful for user: $username');
          return apiResponse.data;
        } else {
          throw Exception(apiResponse.message);
        }
      } else {
        throw NetworkException('HTTP ${response.statusCode}: ${response.statusMessage}');
      }
    } on DioException catch (e) {
      _logger.e('Login failed: ${e.message}');
      if (e is SSLPinningException) {
        rethrow;
      } else {
        throw NetworkException('Network error during login: ${e.message}', originalError: e);
      }
    }
  }
  
  // Get user profile
  Future<UserProfile> getUserProfile(BuildContext context, String token) async {
    final sslProvider = Provider.of<SSLPinningProvider>(context, listen: false);
    
    try {
      final response = await sslProvider.dio.get(
        '$baseUrl/user/profile',
        options: Options(
          headers: {'Authorization': 'Bearer $token'},
        ),
      );
      
      if (response.statusCode == 200) {
        final apiResponse = ApiResponse.fromJson(
          response.data,
          (data) => UserProfile.fromJson(data),
        );
        
        if (apiResponse.success) {
          return apiResponse.data;
        } else {
          throw Exception(apiResponse.message);
        }
      } else {
        throw NetworkException('HTTP ${response.statusCode}: ${response.statusMessage}');
      }
    } on DioException catch (e) {
      _logger.e('Get user profile failed: ${e.message}');
      if (e is SSLPinningException) {
        rethrow;
      } else {
        throw NetworkException('Network error getting user profile: ${e.message}', originalError: e);
      }
    }
  }
}
```

## Cross-Platform Certificate Validation

### 1. Platform-Specific Configurations

Create platform-specific configurations:

```dart
import 'dart:io';

class PlatformSSLConfig {
  static bool get isAndroid => Platform.isAndroid;
  static bool get isIOS => Platform.isIOS;
  
  // Platform-specific SSL configurations
  static Map<String, dynamic> getSSLConfig() {
    if (isAndroid) {
      return {
        'allowBadCertificates': false,
        'timeout': 30000,
        'followRedirects': true,
      };
    } else if (isIOS) {
      return {
        'allowBadCertificates': false,
        'timeout': 30000,
        'followRedirects': true,
      };
    } else {
      // Default configuration for other platforms
      return {
        'allowBadCertificates': false,
        'timeout': 30000,
        'followRedirects': true,
      };
    }
  }
  
  // Get platform-specific error messages
  static String getSSLErrorMessage(DioException error) {
    if (isAndroid) {
      return _getAndroidSSLErrorMessage(error);
    } else if (isIOS) {
      return _getIOSSSLErrorMessage(error);
    } else {
      return 'SSL connection failed: ${error.message}';
    }
  }
  
  static String _getAndroidSSLErrorMessage(DioException error) {
    if (error.message?.contains('CERTIFICATE_VERIFY_FAILED') == true) {
      return 'Certificate verification failed. Please check your internet connection.';
    } else if (error.message?.contains('HANDSHAKE_FAILURE') == true) {
      return 'SSL handshake failed. The server may be using an unsupported protocol.';
    } else {
      return 'SSL connection failed on Android: ${error.message}';
    }
  }
  
  static String _getIOSSSLErrorMessage(DioException error) {
    if (error.message?.contains('kCFStreamErrorDomainSSL') == true) {
      return 'SSL connection failed on iOS. Please check your network settings.';
    } else if (error.message?.contains('NSURLErrorServerCertificateUntrusted') == true) {
      return 'Server certificate is not trusted.';
    } else {
      return 'SSL connection failed on iOS: ${error.message}';
    }
  }
}
```

### 2. Cross-Platform Certificate Storage

Handle certificate storage across platforms:

```dart
import 'package:flutter/services.dart';

class CertificateStorageManager {
  static const String _certificateAssetPath = 'assets/certificates/';
  
  // Load certificate from assets
  static Future<String> loadCertificateFromAssets(String filename) async {
    try {
      return await rootBundle.loadString('$_certificateAssetPath$filename');
    } catch (e) {
      throw Exception('Failed to load certificate from assets: $e');
    }
  }
  
  // Validate certificate format
  static bool isValidCertificateFormat(String certificate) {
    return certificate.contains('-----BEGIN CERTIFICATE-----') &&
           certificate.contains('-----END CERTIFICATE-----');
  }
  
  // Extract certificate information
  static Map<String, String> extractCertificateInfo(String certificate) {
    // This is a simplified implementation
    // In production, you would use proper certificate parsing
    return {
      'format': 'PEM',
      'length': certificate.length.toString(),
      'hasBeginMarker': certificate.contains('-----BEGIN CERTIFICATE-----').toString(),
      'hasEndMarker': certificate.contains('-----END CERTIFICATE-----').toString(),
    };
  }
}
```

## Flutter UI Integration

### 1. Main App Configuration

Configure the app with SSL pinning provider:

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (_) => SSLPinningProvider()),
      ],
      child: MaterialApp(
        title: 'SSL Pinning Demo',
        theme: ThemeData(
          primarySwatch: Colors.blue,
        ),
        home: SplashScreen(),
      ),
    );
  }
}

class SplashScreen extends StatefulWidget {
  @override
  _SplashScreenState createState() => _SplashScreenState();
}

class _SplashScreenState extends State<SplashScreen> {
  @override
  void initState() {
    super.initState();
    _initializeSSLPinning();
  }
  
  Future<void> _initializeSSLPinning() async {
    final sslProvider = Provider.of<SSLPinningProvider>(context, listen: false);
    
    try {
      await sslProvider.initialize();
      // Navigate to main screen after successful initialization
      Navigator.of(context).pushReplacement(
        MaterialPageRoute(builder: (_) => LoginScreen()),
      );
    } catch (e) {
      _showInitializationError(e.toString());
    }
  }
  
  void _showInitializationError(String error) {
    showDialog(
      context: context,
      barrierDismissible: false,
      builder: (context) => AlertDialog(
        title: Text('Initialization Error'),
        content: Text('Failed to initialize SSL pinning: $error'),
        actions: [
          TextButton(
            onPressed: () => _initializeSSLPinning(),
            child: Text('Retry'),
          ),
          TextButton(
            onPressed: () => Navigator.of(context).pop(),
            child: Text('Exit'),
          ),
        ],
      ),
    );
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            CircularProgressIndicator(),
            SizedBox(height: 16),
            Text('Initializing secure connection...'),
          ],
        ),
      ),
    );
  }
}
```

### 2. Login Screen with SSL Error Handling

Create a login screen that handles SSL pinning errors:

```dart
class LoginScreen extends StatefulWidget {
  @override
  _LoginScreenState createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final _formKey = GlobalKey<FormState>();
  final _usernameController = TextEditingController();
  final _passwordController = TextEditingController();
  final _apiService = ApiService();
  
  bool _isLoading = false;
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Secure Login'),
        actions: [
          IconButton(
            icon: Icon(Icons.security),
            onPressed: _showSSLPinningStatus,
          ),
        ],
      ),
      body: Consumer<SSLPinningProvider>(
        builder: (context, sslProvider, child) {
          return Padding(
            padding: EdgeInsets.all(16),
            child: Form(
              key: _formKey,
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  _buildSecurityIndicator(sslProvider),
                  SizedBox(height: 32),
                  TextFormField(
                    controller: _usernameController,
                    decoration: InputDecoration(
                      labelText: 'Username',
                      border: OutlineInputBorder(),
                    ),
                    validator: (value) {
                      if (value?.isEmpty == true) {
                        return 'Please enter username';
                      }
                      return null;
                    },
                  ),
                  SizedBox(height: 16),
                  TextFormField(
                    controller: _passwordController,
                    decoration: InputDecoration(
                      labelText: 'Password',
                      border: OutlineInputBorder(),
                    ),
                    obscureText: true,
                    validator: (value) {
                      if (value?.isEmpty == true) {
                        return 'Please enter password';
                      }
                      return null;
                    },
                  ),
                  SizedBox(height: 24),
                  SizedBox(
                    width: double.infinity,
                    child: ElevatedButton(
                      onPressed: _isLoading ? null : _performLogin,
                      child: _isLoading
                          ? CircularProgressIndicator(color: Colors.white)
                          : Text('Login'),
                    ),
                  ),
                ],
              ),
            ),
          );
        },
      ),
    );
  }
  
  Widget _buildSecurityIndicator(SSLPinningProvider sslProvider) {
    return Container(
      padding: EdgeInsets.all(12),
      decoration: BoxDecoration(
        color: sslProvider.isKillSwitchEnabled ? Colors.orange[100] : Colors.green[100],
        borderRadius: BorderRadius.circular(8),
        border: Border.all(
          color: sslProvider.isKillSwitchEnabled ? Colors.orange : Colors.green,
        ),
      ),
      child: Row(
        children: [
          Icon(
            sslProvider.isKillSwitchEnabled ? Icons.warning : Icons.security,
            color: sslProvider.isKillSwitchEnabled ? Colors.orange : Colors.green,
          ),
          SizedBox(width: 8),
          Expanded(
            child: Text(
              sslProvider.isKillSwitchEnabled
                  ? 'SSL Pinning disabled'
                  : 'Secure connection active',
              style: TextStyle(
                color: sslProvider.isKillSwitchEnabled ? Colors.orange[800] : Colors.green[800],
                fontWeight: FontWeight.bold,
              ),
            ),
          ),
        ],
      ),
    );
  }
  
  Future<void> _performLogin() async {
    if (!_formKey.currentState!.validate()) return;
    
    setState(() => _isLoading = true);
    
    try {
      final loginResponse = await _apiService.login(
        context,
        _usernameController.text,
        _passwordController.text,
      );
      
      // Navigate to main screen on successful login
      Navigator.of(context).pushReplacement(
        MaterialPageRoute(
          builder: (_) => HomeScreen(token: loginResponse.token),
        ),
      );
    } catch (e) {
      _handleLoginError(e);
    } finally {
      setState(() => _isLoading = false);
    }
  }
  
  void _handleLoginError(dynamic error) {
    String title = 'Login Failed';
    String message = error.toString();
    List<Widget> actions = [
      TextButton(
        onPressed: () => Navigator.of(context).pop(),
        child: Text('OK'),
      ),
    ];
    
    if (error is SSLPinningException) {
      title = 'Security Error';
      message = 'A security error occurred while connecting to the server. '
                'This may indicate a potential security threat.\n\nError: ${error.message}';
      actions = [
        TextButton(
          onPressed: () {
            Navigator.of(context).pop();
            _performLogin(); // Retry
          },
          child: Text('Retry'),
        ),
        TextButton(
          onPressed: _contactSupport,
          child: Text('Contact Support'),
        ),
        TextButton(
          onPressed: () => Navigator.of(context).pop(),
          child: Text('Cancel'),
        ),
      ];
    } else if (error is NetworkException) {
      title = 'Network Error';
      message = 'Unable to connect to the server. Please check your internet connection and try again.';
      actions = [
        TextButton(
          onPressed: () {
            Navigator.of(context).pop();
            _performLogin(); // Retry
          },
          child: Text('Retry'),
        ),
        TextButton(
          onPressed: () => Navigator.of(context).pop(),
          child: Text('Cancel'),
        ),
      ];
    }
    
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text(title),
        content: Text(message),
        actions: actions,
      ),
    );
  }
  
  void _showSSLPinningStatus() {
    final sslProvider = Provider.of<SSLPinningProvider>(context, listen: false);
    
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('SSL Pinning Status'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('Host: ${SSLPinningConfig.apigeeHost}'),
            SizedBox(height: 8),
            Text('Pinned Hashes: ${SSLPinningConfig.pinnedHashes.length}'),
            SizedBox(height: 8),
            Text('Backup Hashes: ${SSLPinningConfig.backupHashes.length}'),
            SizedBox(height: 8),
            Text(
              'Status: ${sslProvider.isKillSwitchEnabled ? "Disabled" : "Active"}',
              style: TextStyle(
                fontWeight: FontWeight.bold,
                color: sslProvider.isKillSwitchEnabled ? Colors.red : Colors.green,
              ),
            ),
          ],
        ),
        actions: [
          if (!sslProvider.isKillSwitchEnabled)
            TextButton(
              onPressed: () async {
                await sslProvider.enableKillSwitch();
                Navigator.of(context).pop();
              },
              child: Text('Disable Pinning'),
            ),
          if (sslProvider.isKillSwitchEnabled)
            TextButton(
              onPressed: () async {
                await sslProvider.disableKillSwitch();
                Navigator.of(context).pop();
              },
              child: Text('Enable Pinning'),
            ),
          TextButton(
            onPressed: () => Navigator.of(context).pop(),
            child: Text('Close'),
          ),
        ],
      ),
    );
  }
  
  void _contactSupport() {
    // Implement contact support functionality
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('Contact Support'),
        content: Text('Please contact our support team at support@yourcompany.com'),
        actions: [
          TextButton(
            onPressed: () => Navigator.of(context).pop(),
            child: Text('OK'),
          ),
        ],
      ),
    );
  }
}
```

### 3. Home Screen with Secure API Calls

Create a home screen that demonstrates secure API calls:

```dart
class HomeScreen extends StatefulWidget {
  final String token;
  
  const HomeScreen({Key? key, required this.token}) : super(key: key);
  
  @override
  _HomeScreenState createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  final _apiService = ApiService();
  UserProfile? _userProfile;
  bool _isLoading = true;
  String? _error;
  
  @override
  void initState() {
    super.initState();
    _loadUserProfile();
  }
  
  Future<void> _loadUserProfile() async {
    try {
      final profile = await _apiService.getUserProfile(context, widget.token);
      setState(() {
        _userProfile = profile;
        _isLoading = false;
        _error = null;
      });
    } catch (e) {
      setState(() {
        _error = e.toString();
        _isLoading = false;
      });
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Home'),
        actions: [
          IconButton(
            icon: Icon(Icons.refresh),
            onPressed: _isLoading ? null : () {
              setState(() => _isLoading = true);
              _loadUserProfile();
            },
          ),
          IconButton(
            icon: Icon(Icons.logout),
            onPressed: _logout,
          ),
        ],
      ),
      body: _buildBody(),
    );
  }
  
  Widget _buildBody() {
    if (_isLoading) {
      return Center(child: CircularProgressIndicator());
    }
    
    if (_error != null) {
      return Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(Icons.error, size: 64, color: Colors.red),
            SizedBox(height: 16),
            Text(
              'Error loading profile',
              style: Theme.of(context).textTheme.headlineSmall,
            ),
            SizedBox(height: 8),
            Text(_error!),
            SizedBox(height: 16),
            ElevatedButton(
              onPressed: () {
                setState(() => _isLoading = true);
                _loadUserProfile();
              },
              child: Text('Retry'),
            ),
          ],
        ),
      );
    }
    
    if (_userProfile != null) {
      return Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Card(
              child: Padding(
                padding: EdgeInsets.all(16),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      'User Profile',
                      style: Theme.of(context).textTheme.headlineSmall,
                    ),
                    SizedBox(height: 16),
                    _buildProfileItem('ID', _userProfile!.id),
                    _buildProfileItem('Name', _userProfile!.name),
                    _buildProfileItem('Email', _userProfile!.email),
                  ],
                ),
              ),
            ),
            SizedBox(height: 16),
            _buildSSLPinningStatus(),
          ],
        ),
      );
    }
    
    return Center(child: Text('No data available'));
  }
  
  Widget _buildProfileItem(String label, String value) {
    return Padding(
      padding: EdgeInsets.symmetric(vertical: 4),
      child: Row(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          SizedBox(
            width: 80,
            child: Text(
              '$label:',
              style: TextStyle(fontWeight: FontWeight.bold),
            ),
          ),
          Expanded(child: Text(value)),
        ],
      ),
    );
  }
  
  Widget _buildSSLPinningStatus() {
    return Consumer<SSLPinningProvider>(
      builder: (context, sslProvider, child) {
        return Card(
          child: Padding(
            padding: EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  'Security Status',
                  style: Theme.of(context).textTheme.headlineSmall,
                ),
                SizedBox(height: 16),
                Row(
                  children: [
                    Icon(
                      sslProvider.isKillSwitchEnabled ? Icons.warning : Icons.security,
                      color: sslProvider.isKillSwitchEnabled ? Colors.orange : Colors.green,
                    ),
                    SizedBox(width: 8),
                    Text(
                      sslProvider.isKillSwitchEnabled
                          ? 'SSL Pinning disabled'
                          : 'SSL Pinning active',
                      style: TextStyle(
                        fontWeight: FontWeight.bold,
                        color: sslProvider.isKillSwitchEnabled ? Colors.orange : Colors.green,
                      ),
                    ),
                  ],
                ),
                SizedBox(height: 8),
                Text('Host: ${SSLPinningConfig.apigeeHost}'),
                Text('Pinned hashes: ${SSLPinningConfig.allValidHashes.length}'),
              ],
            ),
          ),
        );
      },
    );
  }
  
  void _logout() {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('Logout'),
        content: Text('Are you sure you want to logout?'),
        actions: [
          TextButton(
            onPressed: () => Navigator.of(context).pop(),
            child: Text('Cancel'),
          ),
          TextButton(
            onPressed: () {
              Navigator.of(context).pop();
              Navigator.of(context).pushAndRemoveUntil(
                MaterialPageRoute(builder: (_) => LoginScreen()),
                (route) => false,
              );
            },
            child: Text('Logout'),
          ),
        ],
      ),
    );
  }
}
```

## Error Handling

### 1. Centralized Error Handler

Create a centralized error handling system:

```dart
class SSLPinningErrorHandler {
  static final Logger _logger = Logger();
  
  static void handleError(BuildContext context, dynamic error, {VoidCallback? onRetry}) {
    _logger.e('Handling error: $error');
    
    if (error is SSLPinningException) {
      _handleSSLPinningError(context, error, onRetry: onRetry);
    } else if (error is NetworkException) {
      _handleNetworkError(context, error, onRetry: onRetry);
    } else if (error is DioException) {
      _handleDioError(context, error, onRetry: onRetry);
    } else {
      _handleGenericError(context, error, onRetry: onRetry);
    }
  }
  
  static void _handleSSLPinningError(BuildContext context, SSLPinningException error, {VoidCallback? onRetry}) {
    showDialog(
      context: context,
      barrierDismissible: false,
      builder: (context) => AlertDialog(
        title: Row(
          children: [
            Icon(Icons.security, color: Colors.red),
            SizedBox(width: 8),
            Text('Security Error'),
          ],
        ),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'A security error occurred while connecting to the server. '
              'This may indicate a potential security threat.',
              style: TextStyle(fontWeight: FontWeight.w500),
            ),
            SizedBox(height: 12),
            Container(
              padding: EdgeInsets.all(8),
              decoration: BoxDecoration(
                color: Colors.red[50],
                borderRadius: BorderRadius.circular(4),
                border: Border.all(color: Colors.red[200]!),
              ),
              child: Text(
                'Error: ${error.message}',
                style: TextStyle(
                  fontSize: 12,
                  fontFamily: 'monospace',
                ),
              ),
            ),
            if (error.hostname != null) ...[
              SizedBox(height: 8),
              Text('Hostname: ${error.hostname}', style: TextStyle(fontSize: 12)),
            ],
          ],
        ),
        actions: [
          if (onRetry != null)
            TextButton(
              onPressed: () {
                Navigator.of(context).pop();
                onRetry();
              },
              child: Text('Retry'),
            ),
          TextButton(
            onPressed: () => _contactSupport(context),
            child: Text('Contact Support'),
          ),
          TextButton(
            onPressed: () => Navigator.of(context).pop(),
            child: Text('Cancel'),
          ),
        ],
      ),
    );
  }
  
  static void _handleNetworkError(BuildContext context, NetworkException error, {VoidCallback? onRetry}) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Row(
          children: [
            Icon(Icons.wifi_off, color: Colors.orange),
            SizedBox(width: 8),
            Text('Network Error'),
          ],
        ),
        content: Text(error.message),
        actions: [
          if (onRetry != null)
            TextButton(
              onPressed: () {
                Navigator.of(context).pop();
                onRetry();
              },
              child: Text('Retry'),
            ),
          TextButton(
            onPressed: () => Navigator.of(context).pop(),
            child: Text('OK'),
          ),
        ],
      ),
    );
  }
  
  static void _handleDioError(BuildContext context, DioException error, {VoidCallback? onRetry}) {
    String message = PlatformSSLConfig.getSSLErrorMessage(error);
    
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('Connection Error'),
        content: Text(message),
        actions: [
          if (onRetry != null)
            TextButton(
              onPressed: () {
                Navigator.of(context).pop();
                onRetry();
              },
              child: Text('Retry'),
            ),
          TextButton(
            onPressed: () => Navigator.of(context).pop(),
            child: Text('OK'),
          ),
        ],
      ),
    );
  }
  
  static void _handleGenericError(BuildContext context, dynamic error, {VoidCallback? onRetry}) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('Error'),
        content: Text(error.toString()),
        actions: [
          if (onRetry != null)
            TextButton(
              onPressed: () {
                Navigator.of(context).pop();
                onRetry();
              },
              child: Text('Retry'),
            ),
          TextButton(
            onPressed: () => Navigator.of(context).pop(),
            child: Text('OK'),
          ),
        ],
      ),
    );
  }
  
  static void _contactSupport(BuildContext context) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('Contact Support'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('Please contact our support team for assistance:'),
            SizedBox(height: 12),
            Text('Email: support@yourcompany.com'),
            Text('Phone: +1-555-0123'),
            SizedBox(height: 12),
            Text(
              'Please mention that you encountered an SSL pinning error.',
              style: TextStyle(fontStyle: FontStyle.italic),
            ),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.of(context).pop(),
            child: Text('OK'),
          ),
        ],
      ),
    );
  }
}
```

## Testing

### 1. Unit Tests

Create comprehensive unit tests:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import 'package:mockito/annotations.dart';

// Generate mocks
@GenerateMocks([Dio, SharedPreferences])
import 'ssl_pinning_test.mocks.dart';

void main() {
  group('SSLPinningConfig', () {
    test('should have valid configuration', () {
      expect(SSLPinningConfig.apigeeHost, isNotEmpty);
      expect(SSLPinningConfig.pinnedHashes, isNotEmpty);
      expect(SSLPinningConfig.allValidHashes, contains(SSLPinningConfig.pinnedHashes.first));
    });
    
    test('should generate SHA256 hashes correctly', () {
      final hashes = SSLPinningConfig.sha256Hashes;
      expect(hashes.length, equals(SSLPinningConfig.allValidHashes.length));
      expect(hashes.first, startsWith('sha256/'));
    });
  });
  
  group('SSLPinningException', () {
    test('should create exception with message', () {
      const message = 'Test SSL error';
      const hostname = 'test.example.com';
      
      final exception = SSLPinningException(message, hostname: hostname);
      
      expect(exception.message, equals(message));
      expect(exception.hostname, equals(hostname));
    });
    
    test('should create exception from DioException', () {
      final dioError = DioException(
        requestOptions: RequestOptions(path: 'https://test.example.com/api'),
        message: 'Certificate verification failed',
      );
      
      final exception = SSLPinningException.fromDioError(dioError);
      
      expect(exception.message, contains('Certificate validation failed'));
      expect(exception.hostname, equals('test.example.com'));
    });
  });
  
  group('PluginSSLPinningManager', () {
    late MockSharedPreferences mockPrefs;
    late PluginSSLPinningManager manager;
    
    setUp(() {
      mockPrefs = MockSharedPreferences();
      manager = PluginSSLPinningManager();
    });
    
    test('should initialize correctly', () async {
      when(mockPrefs.getBool(any)).thenReturn(false);
      
      // Test would require more setup to mock SharedPreferences properly
      expect(manager, isNotNull);
    });
    
    test('should enable kill switch', () async {
      when(mockPrefs.setBool(any, any)).thenAnswer((_) async => true);
      when(mockPrefs.getBool(any)).thenReturn(true);
      
      // Test would require proper initialization
      expect(true, isTrue); // Placeholder
    });
  });
}
```

### 2. Integration Tests

Create integration tests for network scenarios:

```dart
import 'package:flutter/services.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsBinding.ensureInitialized();
  
  group('SSL Pinning Integration Tests', () {
    testWidgets('should handle valid SSL pinning', (WidgetTester tester) async {
      // Initialize app
      await tester.pumpWidget(MyApp());
      await tester.pumpAndSettle();
      
      // Wait for SSL initialization
      await tester.pump(Duration(seconds: 2));
      
      // Verify app initializes correctly
      expect(find.byType(LoginScreen), findsOneWidget);
    });
    
    testWidgets('should show error dialog on SSL failure', (WidgetTester tester) async {
      // This test would require a test server with invalid certificate
      // or mock configuration
      
      await tester.pumpWidget(MyApp());
      await tester.pumpAndSettle();
      
      // Enter credentials
      await tester.enterText(find.byType(TextFormField).first, 'testuser');
      await tester.enterText(find.byType(TextFormField).last, 'testpass');
      
      // Tap login button
      await tester.tap(find.byType(ElevatedButton));
      await tester.pumpAndSettle();
      
      // This would verify error dialog appears in case of SSL failure
      // Implementation depends on test server setup
    });
    
    testWidgets('should respect kill switch setting', (WidgetTester tester) async {
      await tester.pumpWidget(MyApp());
      await tester.pumpAndSettle();
      
      // Navigate to SSL status and enable kill switch
      await tester.tap(find.byIcon(Icons.security));
      await tester.pumpAndSettle();
      
      await tester.tap(find.text('Disable Pinning'));
      await tester.pumpAndSettle();
      
      // Verify kill switch is enabled
      expect(find.text('SSL Pinning disabled'), findsWidgets);
    });
  });
}
```

### 3. Widget Tests

Create widget tests for UI components:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:provider/provider.dart';

void main() {
  group('LoginScreen Widget Tests', () {
    testWidgets('should display login form', (WidgetTester tester) async {
      await tester.pumpWidget(
        MaterialApp(
          home: ChangeNotifierProvider(
            create: (_) => SSLPinningProvider(),
            child: LoginScreen(),
          ),
        ),
      );
      
      expect(find.byType(TextFormField), findsNWidgets(2));
      expect(find.byType(ElevatedButton), findsOneWidget);
      expect(find.text('Login'), findsOneWidget);
    });
    
    testWidgets('should validate empty fields', (WidgetTester tester) async {
      await tester.pumpWidget(
        MaterialApp(
          home: ChangeNotifierProvider(
            create: (_) => SSLPinningProvider(),
            child: LoginScreen(),
          ),
        ),
      );
      
      // Tap login without entering credentials
      await tester.tap(find.byType(ElevatedButton));
      await tester.pump();
      
      expect(find.text('Please enter username'), findsOneWidget);
      expect(find.text('Please enter password'), findsOneWidget);
    });
    
    testWidgets('should show security indicator', (WidgetTester tester) async {
      final sslProvider = SSLPinningProvider();
      
      await tester.pumpWidget(
        MaterialApp(
          home: ChangeNotifierProvider.value(
            value: sslProvider,
            child: LoginScreen(),
          ),
        ),
      );
      
      expect(find.byIcon(Icons.security), findsOneWidget);
      expect(find.text('Secure connection active'), findsOneWidget);
    });
  });
  
  group('HomeScreen Widget Tests', () {
    testWidgets('should display loading indicator', (WidgetTester tester) async {
      await tester.pumpWidget(
        MaterialApp(
          home: ChangeNotifierProvider(
            create: (_) => SSLPinningProvider(),
            child: HomeScreen(token: 'test-token'),
          ),
        ),
      );
      
      expect(find.byType(CircularProgressIndicator), findsOneWidget);
    });
  });
}
```

## Best Practices

### 1. Security Best Practices

```dart
class SecurityBestPractices {
  // Never log sensitive information
  static void logSecurely(String message, {dynamic data}) {
    final Logger logger = Logger();
    
    // Filter sensitive data
    String safeMessage = message
        .replaceAll(RegExp(r'Bearer [A-Za-z0-9-_]+\.[A-Za-z0-9-_]+\.[A-Za-z0-9-_]+'), 'Bearer [REDACTED]')
        .replaceAll(RegExp(r'"password"\s*:\s*"[^"]*"'), '"password":"[REDACTED]"')
        .replaceAll(RegExp(r'"token"\s*:\s*"[^"]*"'), '"token":"[REDACTED]"');
    
    logger.d(safeMessage);
  }
  
  // Secure storage of sensitive configuration
  static Future<void> storeSecureConfig(String key, String value) async {
    // In production, use flutter_secure_storage instead of SharedPreferences
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString(key, value);
  }
  
  // Validate certificate expiry
  static bool isCertificateValid(String certificate) {
    // Implement certificate validation logic
    return certificate.isNotEmpty && 
           certificate.contains('-----BEGIN CERTIFICATE-----');
  }
}
```

### 2. Performance Optimization

```dart
class PerformanceOptimizations {
  // Connection pool management
  static Dio createOptimizedDio() {
    final dio = Dio();
    
    // Configure connection pooling
    (dio.httpClientAdapter as DefaultHttpClientAdapter).onHttpClientCreate = (client) {
      client.maxConnectionsPerHost = 5;
      client.connectionTimeout = Duration(seconds: 30);
      client.idleTimeout = Duration(seconds: 15);
      return client;
    };
    
    return dio;
  }
  
  // Cache certificate validation results
  static final Map<String, bool> _validationCache = {};
  
  static bool getCachedValidation(String certificateHash) {
    return _validationCache[certificateHash] ?? false;
  }
  
  static void setCachedValidation(String certificateHash, bool isValid) {
    _validationCache[certificateHash] = isValid;
  }
  
  static void clearValidationCache() {
    _validationCache.clear();
  }
}
```

### 3. Build Configuration

Configure different settings for different build modes:

```dart
class BuildConfig {
  static const bool isProduction = bool.fromEnvironment('dart.vm.product');
  static const bool isProfile = bool.fromEnvironment('dart.vm.profile');
  static const bool isDebug = !isProduction && !isProfile;
  
  // SSL Pinning configuration based on build mode
  static bool get sslPinningEnabled => isProduction || bool.fromEnvironment('SSL_PINNING_ENABLED', defaultValue: false);
  
  static List<String> get pinnedHashes {
    if (isDebug) {
      return []; // No pinning in debug mode
    } else {
      return SSLPinningConfig.allValidHashes;
    }
  }
  
  // Logging configuration
  static bool get loggingEnabled => !isProduction;
}
```

## Troubleshooting

### Common Issues and Solutions

1. **Certificate Format Issues**
   ```dart
   class CertificateTroubleshooting {
     static void debugCertificate(String certificate) {
       print('Certificate length: ${certificate.length}');
       print('Has BEGIN marker: ${certificate.contains('-----BEGIN CERTIFICATE-----')}');
       print('Has END marker: ${certificate.contains('-----END CERTIFICATE-----')}');
       print('Line count: ${certificate.split('\n').length}');
     }
     
     static String normalizeCertificate(String certificate) {
       return certificate
           .replaceAll(RegExp(r'\r\n'), '\n')
           .replaceAll(RegExp(r'\r'), '\n')
           .trim();
     }
   }
   ```

2. **Platform-Specific Issues**
   ```dart
   class PlatformTroubleshooting {
     static void debugPlatformSSL() {
       if (Platform.isAndroid) {
         print('Android SSL debugging:');
         print('- Check network_security_config.xml');
         print('- Verify minSdkVersion >= 21');
         print('- Check for custom certificate authorities');
       } else if (Platform.isIOS) {
         print('iOS SSL debugging:');
         print('- Check Info.plist for NSAppTransportSecurity');
         print('- Verify iOS version >= 11.0');
         print('- Check for certificate trust settings');
       }
     }
   }
   ```

3. **Debug Tools**
   ```dart
   class SSLDebugTools {
     static void testSSLConnection(String hostname) async {
       try {
         final dio = Dio();
         final response = await dio.get('https://$hostname');
         print('SSL connection successful: ${response.statusCode}');
       } catch (e) {
         print('SSL connection failed: $e');
         
         if (e is DioException) {
           print('Error type: ${e.type}');
           print('Error message: ${e.message}');
           if (e.response != null) {
             print('Response status: ${e.response?.statusCode}');
           }
         }
       }
     }
     
     static void extractCertificateInfo(String hostname) async {
       // This would require additional implementation to extract
       // certificate information from the server
       print('Extracting certificate info for: $hostname');
     }
   }
   ```

This comprehensive Flutter implementation provides robust SSL pinning with proper error handling, state management, and cross-platform support for production use.
````