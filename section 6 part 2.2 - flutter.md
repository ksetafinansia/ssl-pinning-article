# Flutter Kill Switch Implementation (Dart)

This guide shows how to implement Firebase Remote Config kill switch for SSL pinning in Flutter applications using Dart and Dio.

## Prerequisites
- Existing SSL pinning implementation from **Section 6 Part 1**
- Firebase project with Remote Config enabled
- Firebase Flutter SDK integrated

## Step 1: Remote Config Manager

### Remote Config Manager
```dart
// remote_config_manager.dart
import 'dart:convert';
import 'package:firebase_remote_config/firebase_remote_config.dart';
import 'package:package_info_plus/package_info_plus.dart';
import 'package:flutter/foundation.dart';
import 'dart:math' as math;

class SSLPinningConfig {
  final bool enabled;
  final List<String> targetHosts;
  final String minAppVersion;

  SSLPinningConfig({
    required this.enabled,
    required this.targetHosts,
    required this.minAppVersion,
  });

  factory SSLPinningConfig.fromJson(Map<String, dynamic> json) {
    // Handle both array and comma-separated string formats
    List<String> hosts = [];
    final hostsData = json['target_hosts'];
    
    if (hostsData is List) {
      hosts = hostsData.cast<String>();
    } else if (hostsData is String) {
      hosts = hostsData.isEmpty ? [] : hostsData.split(',').map((e) => e.trim()).toList();
    }

    return SSLPinningConfig(
      enabled: json['enabled'] ?? false,
      targetHosts: hosts,
      minAppVersion: json['min_app_version'] ?? '1.0.0',
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'enabled': enabled,
      'target_hosts': targetHosts,
      'min_app_version': minAppVersion,
    };
  }
}

class RemoteConfigManager {
  static const String _sslConfigKey = 'ssl_pinning_config';
  late FirebaseRemoteConfig _remoteConfig;
  static RemoteConfigManager? _instance;

  RemoteConfigManager._() {
    _remoteConfig = FirebaseRemoteConfig.instance;
    _initialize();
  }

  static RemoteConfigManager get instance {
    _instance ??= RemoteConfigManager._();
    return _instance!;
  }

  void _initialize() async {
    await _remoteConfig.setConfigSettings(
      RemoteConfigSettings(
        fetchTimeout: const Duration(minutes: 1),
        minimumFetchInterval: kDebugMode ? Duration.zero : const Duration(hours: 1),
      ),
    );

    await _remoteConfig.setDefaults({
      _sslConfigKey: jsonEncode(_getDefaultConfig().toJson()),
    });
  }

  Future<bool> fetchAndActivate() async {
    try {
      return await _remoteConfig.fetchAndActivate();
    } catch (e) {
      print('Remote config fetch failed: $e');
      return false;
    }
  }

  SSLPinningConfig getSSLPinningConfig() {
    try {
      final configString = _remoteConfig.getString(_sslConfigKey);
      final configJson = jsonDecode(configString) as Map<String, dynamic>;
      return SSLPinningConfig.fromJson(configJson);
    } catch (e) {
      print('Failed to decode SSL pinning config: $e');
      return _getDefaultConfig();
    }
  }

  Future<bool> shouldEnablePinning() async {
    final config = getSSLPinningConfig();

    // Global enable check
    if (!config.enabled) {
      return false;
    }

    // App version check
    if (!await _isAppVersionSupported(config.minAppVersion)) {
      return false;
    }

    // Check if we have any target hosts
    if (config.targetHosts.isEmpty) {
      return false;
    }

    return true;
  }

  bool shouldPinHost(String hostname) {
    final config = getSSLPinningConfig();
    
    // Quick checks without async
    if (!config.enabled || config.targetHosts.isEmpty) {
      return false;
    }
    
    return config.targetHosts.contains(hostname);
  }

  SSLPinningConfig _getDefaultConfig() {
    return SSLPinningConfig(
      enabled: false,
      targetHosts: [],
      minAppVersion: '1.0.0',
    );
  }

  Future<bool> _isAppVersionSupported(String minVersion) async {
    try {
      final packageInfo = await PackageInfo.fromPlatform();
      return _compareVersions(packageInfo.version, minVersion) >= 0;
    } catch (e) {
      return false;
    }
  }

  int _compareVersions(String version1, String version2) {
    final v1Parts = version1.split('.').map((e) => int.tryParse(e) ?? 0).toList();
    final v2Parts = version2.split('.').map((e) => int.tryParse(e) ?? 0).toList();
    
    final maxLength = math.max(v1Parts.length, v2Parts.length);
    
    for (int i = 0; i < maxLength; i++) {
      final v1Part = i < v1Parts.length ? v1Parts[i] : 0;
      final v2Part = i < v2Parts.length ? v2Parts[i] : 0;
      final result = v1Part.compareTo(v2Part);
      if (result != 0) return result;
    }
    return 0;
  }
}
```

## Step 2: Updated HTTP Client with Remote Config

### Enhanced HTTP Client
```dart
// updated_http_client.dart
import 'package:dio/dio.dart';
import 'package:dio_certificate_pinning/dio_certificate_pinning.dart';
import 'remote_config_manager.dart';

class UpdatedPinnedHttpClient {
  static final RemoteConfigManager _remoteConfigManager = RemoteConfigManager.instance;
  static Dio? _instance;

  // Build-time pins (from Section 6 Part 1)
  static const Map<String, List<String>> _buildTimePins = {
    'apigee.kreditplus.com': [
      'your_build_time_primary_pin',
      'your_build_time_backup_pin'
    ],
    'apigee.kbfinansia.com': [
      'your_staging_pin'
    ],
  };

  static Future<Dio> getInstance() async {
    if (_instance == null) {
      await _remoteConfigManager.fetchAndActivate();
      _instance = await _createClient();
    }
    return _instance!;
  }

  static Future<Dio> _createClient() async {
    final shouldPin = await _remoteConfigManager.shouldEnablePinning();
    
    if (shouldPin) {
      return _createPinnedClient();
    } else {
      return _createNormalClient();
    }
  }

  static Dio _createPinnedClient() {
    final dio = Dio();

    // Add certificate pinning with build-time pins
    final allPins = _buildTimePins.values.expand((pins) => pins).toList();
    dio.interceptors.add(
      CertificatePinningInterceptor(
        allowedSHAFingerprints: allPins,
      ),
    );

    // Add host check interceptor
    dio.interceptors.add(HostCheckInterceptor(_remoteConfigManager));

    _configureBaseOptions(dio);
    return dio;
  }

  static Dio _createNormalClient() {
    final dio = Dio();
    _configureBaseOptions(dio);
    return dio;
  }

  static void _configureBaseOptions(Dio dio) {
    dio.options = BaseOptions(
      connectTimeout: const Duration(seconds: 30),
      receiveTimeout: const Duration(seconds: 30),
      sendTimeout: const Duration(seconds: 30),
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
    );
  }

  // Force refresh of configuration and recreate client
  static Future<void> refreshConfiguration() async {
    await _remoteConfigManager.fetchAndActivate();
    _instance = await _createClient();
  }
  
  // Clear instance to force recreation
  static void clearInstance() {
    _instance = null;
  }
}

class HostCheckInterceptor extends Interceptor {
  final RemoteConfigManager remoteConfigManager;

  HostCheckInterceptor(this.remoteConfigManager);

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final hostname = options.uri.host;
    
    // Check if this specific host should be pinned
    if (!remoteConfigManager.shouldPinHost(hostname)) {
      // If host is not in target list, we could either:
      // 1. Allow with normal TLS (current approach)
      // 2. Reject the request
      // For this implementation, we'll allow it
      print('Host $hostname not in pinning target list, using normal TLS');
    }
    
    handler.next(options);
  }
  
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    // Log SSL pinning failures for monitoring
    if (err.type == DioExceptionType.connectionError && 
        err.message?.contains('certificate') == true) {
      print('SSL Certificate validation failed for ${err.requestOptions.uri.host}');
    }
    handler.next(err);
  }
}
```

## Step 3: API Service Integration

### API Service
```dart
// api_service.dart
import 'package:dio/dio.dart';
import 'updated_http_client.dart';

class ApiService {
  static ApiService? _instance;
  late Dio _dio;

  ApiService._() {
    _initializeDio();
  }

  static ApiService get instance {
    _instance ??= ApiService._();
    return _instance!;
  }

  Future<void> _initializeDio() async {
    _dio = await UpdatedPinnedHttpClient.getInstance();
  }

  // Force refresh configuration (call this when needed)
  Future<void> refreshConfiguration() async {
    await UpdatedPinnedHttpClient.refreshConfiguration();
    _dio = await UpdatedPinnedHttpClient.getInstance();
  }

  Future<Map<String, dynamic>> get(String endpoint) async {
    try {
      final response = await _dio.get(endpoint);
      return response.data;
    } catch (e) {
      print('API GET error: $e');
      rethrow;
    }
  }

  Future<Map<String, dynamic>> post(String endpoint, Map<String, dynamic> data) async {
    try {
      final response = await _dio.post(endpoint, data: data);
      return response.data;
    } catch (e) {
      print('API POST error: $e');
      rethrow;
    }
  }
}

// Usage example
class UserRepository {
  final ApiService _apiService = ApiService.instance;

  Future<Map<String, dynamic>> getUserData() async {
    return await _apiService.get('https://apigee.kreditplus.com/api/v1/user');
  }

  Future<Map<String, dynamic>> updateUser(Map<String, dynamic> userData) async {
    return await _apiService.post('https://apigee.kreditplus.com/api/v1/user', userData);
  }
}
```

## Step 4: Migration from Part 1 to Part 2

### Before (Part 1 - Basic Implementation)
```dart
// Basic pinning - always enabled
class BasicPinnedHttpClient {
  static Dio get instance {
    final dio = Dio();
    
    // Always add certificate pinning
    dio.interceptors.add(
      CertificatePinningInterceptor(
        allowedSHAFingerprints: ['your_pin_here'],
      ),
    );
    
    return dio;
  }
}
```

### After (Part 2 - Enhanced Implementation)
```dart
// Enhanced with remote config control
class EnhancedPinnedHttpClient {
  static Future<Dio> createClient() async {
    final remoteConfig = RemoteConfigManager.instance;
    await remoteConfig.fetchAndActivate();
    
    // NEW: Check remote config before applying pinning
    if (await remoteConfig.shouldEnablePinning()) {
      // Use existing Part 1 pinning logic
      return _createPinnedClient();
    } else {
      // NEW: Bypass pinning capability
      return _createNormalClient();
    }
  }
  
  static Dio _createPinnedClient() {
    final dio = Dio();
    dio.interceptors.add(
      CertificatePinningInterceptor(
        allowedSHAFingerprints: ['your_pin_here'],
      ),
    );
    return dio;
  }
  
  static Dio _createNormalClient() {
    return Dio(); // No pinning
  }
}
```

## Step 5: App Initialization

### Main App Setup
```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'firebase_options.dart';
import 'remote_config_manager.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Initialize Firebase
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  
  // Initialize Remote Config
  await RemoteConfigManager.instance.fetchAndActivate();
  
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'SSL Pinning Demo',
      home: HomePage(),
    );
  }
}
```

### Widget Integration
```dart
// home_page.dart
import 'package:flutter/material.dart';
import 'api_service.dart';

class HomePage extends StatefulWidget {
  @override
  _HomePageState createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  final ApiService _apiService = ApiService.instance;
  
  @override
  void initState() {
    super.initState();
    _initializeConfiguration();
  }
  
  Future<void> _initializeConfiguration() async {
    try {
      await _apiService.refreshConfiguration();
      print('SSL pinning configuration updated');
    } catch (e) {
      print('Failed to update configuration: $e');
    }
  }
  
  Future<void> _makeApiCall() async {
    try {
      final result = await _apiService.get('https://apigee.kreditplus.com/api/v1/test');
      print('API call successful: $result');
    } catch (e) {
      print('API call failed: $e');
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('API call failed: $e')),
      );
    }
  }
  
  Future<void> _refreshConfiguration() async {
    try {
      await _apiService.refreshConfiguration();
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Configuration refreshed')),
      );
    } catch (e) {
      print('Failed to refresh configuration: $e');
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('SSL Pinning Demo')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: _makeApiCall,
              child: Text('Make API Call'),
            ),
            SizedBox(height: 20),
            ElevatedButton(
              onPressed: _refreshConfiguration,
              child: Text('Refresh Configuration'),
            ),
          ],
        ),
      ),
    );
  }
}
```

## Dependencies

### Add to pubspec.yaml
```yaml
dependencies:
  flutter:
    sdk: flutter
  
  # Firebase
  firebase_core: ^2.17.0
  firebase_remote_config: ^4.3.8
  
  # HTTP client
  dio: ^5.3.2
  dio_certificate_pinning: ^6.0.0
  
  # App info
  package_info_plus: ^4.2.0
  
dev_dependencies:
  flutter_test:
    sdk: flutter
  mockito: ^5.4.2
  build_runner: ^2.4.7
```

## Testing

### Unit Test Example
```dart
// test/remote_config_manager_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import 'package:your_app/remote_config_manager.dart';

class MockFirebaseRemoteConfig extends Mock implements FirebaseRemoteConfig {}

void main() {
  group('RemoteConfigManager', () {
    test('should disable pinning when config enabled is false', () async {
      // Mock setup
      final manager = RemoteConfigManager.instance;
      // ... test implementation
    });
    
    test('should disable pinning when app version is below minimum', () async {
      // Test version comparison logic
      final manager = RemoteConfigManager.instance;
      // ... test implementation
    });
    
    test('version comparison works correctly', () {
      final manager = RemoteConfigManager.instance;
      
      expect(manager._compareVersions('1.5.0', '1.4.9'), greaterThan(0));
      expect(manager._compareVersions('1.5.0', '1.5.0'), equals(0));
      expect(manager._compareVersions('1.4.9', '1.5.0'), lessThan(0));
    });
  });
}
```

### Integration Test
```dart
// integration_test/kill_switch_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:your_app/main.dart' as app;
import 'package:your_app/updated_http_client.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  
  group('Kill Switch Integration Tests', () {
    testWidgets('should use normal TLS when kill switch is activated', (tester) async {
      app.main();
      await tester.pumpAndSettle();
      
      // Setup: Enable kill switch in Firebase console (done manually)
      
      // Test: Create HTTP client
      final client = await UpdatedPinnedHttpClient.getInstance();
      
      // Verify: Should be able to make requests without pinning
      // ... test implementation
    });
  });
}
```

### Widget Test
```dart
// test/widget_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:your_app/home_page.dart';

void main() {
  testWidgets('HomePage should handle API calls correctly', (WidgetTester tester) async {
    await tester.pumpWidget(MaterialApp(home: HomePage()));
    
    // Find and tap the API call button
    expect(find.text('Make API Call'), findsOneWidget);
    await tester.tap(find.text('Make API Call'));
    await tester.pump();
    
    // Verify the behavior
    // ... test implementation
  });
}
```

## Best Practices

### 1. **Error Handling**
- Always provide safe defaults when Remote Config fails
- Don't crash the app if configuration parsing fails
- Log configuration errors for debugging

### 2. **Performance**
- Cache the Dio instance to avoid repeated configuration fetches
- Use appropriate fetch intervals (1 hour for production, immediate for debug)
- Consider background configuration refresh

### 3. **Security**
- Keep certificate pins embedded in the app build
- Use Firebase Remote Config only for control logic
- Validate configuration before applying changes

### 4. **State Management**
If using state management solutions like Provider, Riverpod, or Bloc:

```dart
// Using Provider example
class ConfigurationProvider extends ChangeNotifier {
  bool _isPinningEnabled = false;
  
  bool get isPinningEnabled => _isPinningEnabled;
  
  Future<void> updateConfiguration() async {
    final config = RemoteConfigManager.instance;
    await config.fetchAndActivate();
    _isPinningEnabled = await config.shouldEnablePinning();
    notifyListeners();
  }
}
```

**📋 For configuration management, emergency procedures, and operational guidance, see Section 10.**
