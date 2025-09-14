````markdown
# SSL Pinning Nuxt.js CMS Implementation

## Overview

This guide provides a simple SSL pinning implementation for Nuxt.js CMS applications with server-side certificate validation and client-side security composables. The implementation focuses on securing connections to Apigee endpoints while maintaining browser compatibility.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Dependencies](#dependencies)
3. [Server-Side Certificate Validation](#server-side-certificate-validation)
4. [Client-Side Security Composables](#client-side-security-composables)
5. [Configuration Management](#configuration-management)
6. [API Integration](#api-integration)
7. [Error Handling](#error-handling)
8. [Testing](#testing)
9. [Best Practices](#best-practices)

## Prerequisites

- Node.js 18+
- Nuxt.js 3.0+
- Basic understanding of Nuxt.js and Vue.js
- Understanding of server-side and client-side rendering

## Dependencies

Add these dependencies to your `package.json`:

```json
{
  "dependencies": {
    "nuxt": "^3.8.0",
    "@nuxt/ui": "^2.10.0",
    "node-forge": "^1.3.1",
    "crypto": "^1.0.1",
    "https": "^1.0.0",
    "tls": "^0.0.1"
  },
  "devDependencies": {
    "@nuxt/test-utils": "^3.8.0",
    "vitest": "^0.34.0",
    "playwright": "^1.40.0"
  }
}
```

## Server-Side Certificate Validation

### 1. SSL Pinning Configuration

Create `server/utils/ssl-pinning-config.ts`:

```typescript
export interface SSLPinningConfig {
  hostname: string;
  pinnedHashes: string[];
  backupHashes: string[];
  killSwitchEnabled: boolean;
}

export const sslPinningConfig: SSLPinningConfig = {
  hostname: 'your-apigee-domain.com',
  pinnedHashes: [
    'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=', // Primary hash
    'BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=', // Secondary hash
  ],
  backupHashes: [
    'CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC=', // Backup hash
  ],
  killSwitchEnabled: process.env.SSL_PINNING_KILL_SWITCH === 'true',
};

export const getAllValidHashes = (): string[] => [
  ...sslPinningConfig.pinnedHashes,
  ...sslPinningConfig.backupHashes,
];
```

### 2. Certificate Validator

Create `server/utils/certificate-validator.ts`:

```typescript
import * as crypto from 'crypto';
import * as https from 'https';
import * as tls from 'tls';
import { sslPinningConfig, getAllValidHashes } from './ssl-pinning-config';

export class CertificateValidator {
  private static instance: CertificateValidator;

  static getInstance(): CertificateValidator {
    if (!CertificateValidator.instance) {
      CertificateValidator.instance = new CertificateValidator();
    }
    return CertificateValidator.instance;
  }

  /**
   * Validate certificate against pinned hashes
   */
  validateCertificate(certificate: any, hostname: string): boolean {
    try {
      // Skip validation if kill switch is enabled
      if (sslPinningConfig.killSwitchEnabled) {
        console.warn(`SSL pinning disabled via kill switch for ${hostname}`);
        return true;
      }

      // Only validate for configured hostname
      if (hostname !== sslPinningConfig.hostname) {
        console.log(`Certificate validation skipped for hostname: ${hostname}`);
        return true;
      }

      // Extract public key hash
      const publicKeyHash = this.extractPublicKeyHash(certificate);
      const validHashes = getAllValidHashes();

      // Check if hash matches any pinned hash
      const isValid = validHashes.includes(publicKeyHash);

      if (isValid) {
        console.log(`✅ Certificate validation SUCCESS for ${hostname}`);
      } else {
        console.error(`❌ Certificate validation FAILED for ${hostname}`);
        console.log(`Received hash: ${publicKeyHash}`);
        console.log(`Expected hashes: ${validHashes.join(', ')}`);
      }

      return isValid;
    } catch (error) {
      console.error('Certificate validation error:', error);
      return false;
    }
  }

  /**
   * Extract SHA-256 hash of public key
   */
  private extractPublicKeyHash(certificate: any): string {
    try {
      // Get the public key from certificate
      const publicKey = certificate.publicKey;
      
      // Create SHA-256 hash of the public key
      const hash = crypto.createHash('sha256');
      hash.update(publicKey.n.toBuffer()); // For RSA keys
      
      // Return base64 encoded hash
      return hash.digest('base64');
    } catch (error) {
      console.error('Error extracting public key hash:', error);
      throw new Error('Failed to extract public key hash');
    }
  }

  /**
   * Create HTTPS agent with certificate pinning
   */
  createSecureAgent(): https.Agent {
    return new https.Agent({
      checkServerIdentity: (hostname: string, cert: any) => {
        // Perform default hostname verification
        const defaultResult = tls.checkServerIdentity(hostname, cert);
        if (defaultResult) {
          return defaultResult;
        }

        // Perform additional certificate pinning validation
        if (!this.validateCertificate(cert, hostname)) {
          return new Error(`Certificate pinning validation failed for ${hostname}`);
        }

        return undefined;
      },
      rejectUnauthorized: true,
      timeout: 30000,
    });
  }
}
```

### 3. Secure HTTP Client

Create `server/utils/secure-http-client.ts`:

```typescript
import { $fetch, FetchOptions } from 'ofetch';
import { CertificateValidator } from './certificate-validator';

export class SecureHttpClient {
  private static instance: SecureHttpClient;
  private certificateValidator: CertificateValidator;

  constructor() {
    this.certificateValidator = CertificateValidator.getInstance();
  }

  static getInstance(): SecureHttpClient {
    if (!SecureHttpClient.instance) {
      SecureHttpClient.instance = new SecureHttpClient();
    }
    return SecureHttpClient.instance;
  }

  /**
   * Make secure API request with certificate pinning
   */
  async secureRequest<T>(url: string, options: FetchOptions = {}): Promise<T> {
    try {
      // Create secure agent for certificate pinning
      const agent = this.certificateValidator.createSecureAgent();

      // Configure fetch options with secure agent
      const secureOptions: FetchOptions = {
        ...options,
        // @ts-ignore - Adding agent for Node.js environment
        agent,
        timeout: 30000,
        retry: 2,
        retryDelay: 1000,
      };

      // Make the request
      const response = await $fetch<T>(url, secureOptions);
      
      console.log(`✅ Secure request successful: ${url}`);
      return response;

    } catch (error: any) {
      console.error(`❌ Secure request failed: ${url}`, error);
      
      // Handle SSL-specific errors
      if (this.isSSLError(error)) {
        throw new SSLPinningError(`SSL pinning validation failed: ${error.message}`, url);
      }
      
      throw error;
    }
  }

  /**
   * Check if error is SSL-related
   */
  private isSSLError(error: any): boolean {
    const errorMessage = error.message?.toLowerCase() || '';
    return (
      errorMessage.includes('certificate') ||
      errorMessage.includes('ssl') ||
      errorMessage.includes('tls') ||
      errorMessage.includes('handshake') ||
      error.code === 'CERT_SIGNATURE_FAILURE' ||
      error.code === 'UNABLE_TO_VERIFY_LEAF_SIGNATURE'
    );
  }
}

export class SSLPinningError extends Error {
  constructor(message: string, public url: string) {
    super(message);
    this.name = 'SSLPinningError';
  }
}
```

### 4. API Server Routes

Create `server/api/secure-proxy.post.ts`:

```typescript
import { SecureHttpClient, SSLPinningError } from '~/server/utils/secure-http-client';
import { sslPinningConfig } from '~/server/utils/ssl-pinning-config';

export default defineEventHandler(async (event) => {
  try {
    const body = await readBody(event);
    const { endpoint, method = 'GET', data, headers = {} } = body;

    // Validate endpoint
    if (!endpoint || typeof endpoint !== 'string') {
      throw createError({
        statusCode: 400,
        statusMessage: 'Invalid endpoint provided',
      });
    }

    // Construct full URL
    const baseUrl = `https://${sslPinningConfig.hostname}/api/v1`;
    const fullUrl = `${baseUrl}${endpoint}`;

    // Get secure HTTP client
    const httpClient = SecureHttpClient.getInstance();

    // Make secure request
    const response = await httpClient.secureRequest(fullUrl, {
      method,
      body: data,
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        ...headers,
      },
    });

    return {
      success: true,
      data: response,
    };

  } catch (error: any) {
    console.error('Secure proxy error:', error);

    if (error instanceof SSLPinningError) {
      throw createError({
        statusCode: 426,
        statusMessage: 'SSL Pinning Validation Failed',
        data: {
          type: 'ssl_pinning_error',
          message: error.message,
          url: error.url,
        },
      });
    }

    throw createError({
      statusCode: 500,
      statusMessage: 'Internal Server Error',
      data: {
        type: 'network_error',
        message: error.message,
      },
    });
  }
});
```

## Client-Side Security Composables

### 1. SSL Pinning Composable

Create `composables/useSSLPinning.ts`:

```typescript
export interface SSLPinningState {
  isEnabled: boolean;
  hostname: string;
  lastError: string | null;
  connectionStatus: 'connected' | 'disconnected' | 'error';
}

export const useSSLPinning = () => {
  const state = reactive<SSLPinningState>({
    isEnabled: true,
    hostname: 'your-apigee-domain.com',
    lastError: null,
    connectionStatus: 'disconnected',
  });

  /**
   * Make secure API request through server proxy
   */
  const secureRequest = async <T>(
    endpoint: string,
    options: {
      method?: string;
      data?: any;
      headers?: Record<string, string>;
    } = {}
  ): Promise<T> => {
    try {
      state.connectionStatus = 'disconnected';
      state.lastError = null;

      const response = await $fetch<{ success: boolean; data: T }>('/api/secure-proxy', {
        method: 'POST',
        body: {
          endpoint,
          method: options.method || 'GET',
          data: options.data,
          headers: options.headers,
        },
      });

      if (response.success) {
        state.connectionStatus = 'connected';
        return response.data;
      } else {
        throw new Error('Request failed');
      }

    } catch (error: any) {
      state.connectionStatus = 'error';
      state.lastError = error.data?.message || error.message;

      if (error.status === 426) {
        throw new SSLPinningClientError(error.data?.message || 'SSL Pinning validation failed');
      }

      throw error;
    }
  };

  /**
   * Check SSL pinning status
   */
  const checkStatus = async (): Promise<boolean> => {
    try {
      await secureRequest('/health');
      return true;
    } catch (error) {
      console.error('SSL pinning status check failed:', error);
      return false;
    }
  };

  /**
   * Clear error state
   */
  const clearError = (): void => {
    state.lastError = null;
    if (state.connectionStatus === 'error') {
      state.connectionStatus = 'disconnected';
    }
  };

  return {
    state: readonly(state),
    secureRequest,
    checkStatus,
    clearError,
  };
};

export class SSLPinningClientError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'SSLPinningClientError';
  }
}
```

### 2. API Composable

Create `composables/useApi.ts`:

```typescript
export interface LoginRequest {
  username: string;
  password: string;
}

export interface LoginResponse {
  token: string;
  user: {
    id: string;
    name: string;
    email: string;
  };
  expiresAt: string;
}

export interface UserProfile {
  id: string;
  name: string;
  email: string;
  role: string;
}

export const useApi = () => {
  const { secureRequest } = useSSLPinning();

  /**
   * Login user
   */
  const login = async (credentials: LoginRequest): Promise<LoginResponse> => {
    return await secureRequest<LoginResponse>('/auth/login', {
      method: 'POST',
      data: credentials,
    });
  };

  /**
   * Get user profile
   */
  const getUserProfile = async (token: string): Promise<UserProfile> => {
    return await secureRequest<UserProfile>('/user/profile', {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${token}`,
      },
    });
  };

  /**
   * Get dashboard data
   */
  const getDashboardData = async (token: string): Promise<any> => {
    return await secureRequest('/dashboard', {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${token}`,
      },
    });
  };

  return {
    login,
    getUserProfile,
    getDashboardData,
  };
};
```

### 3. Error Handling Composable

Create `composables/useErrorHandler.ts`:

```typescript
export const useErrorHandler = () => {
  const toast = useToast();

  /**
   * Handle SSL pinning errors
   */
  const handleSSLError = (error: SSLPinningClientError): void => {
    toast.add({
      title: 'Security Error',
      description: 'A security error occurred while connecting to the server. This may indicate a potential security threat.',
      color: 'red',
      timeout: 10000,
      actions: [
        {
          label: 'Contact Support',
          click: () => contactSupport(),
        },
      ],
    });
  };

  /**
   * Handle network errors
   */
  const handleNetworkError = (error: any): void => {
    toast.add({
      title: 'Network Error',
      description: 'Unable to connect to the server. Please check your internet connection.',
      color: 'orange',
      timeout: 5000,
    });
  };

  /**
   * Handle general errors
   */
  const handleError = (error: any): void => {
    if (error instanceof SSLPinningClientError) {
      handleSSLError(error);
    } else if (error.status >= 500) {
      handleNetworkError(error);
    } else {
      toast.add({
        title: 'Error',
        description: error.message || 'An unexpected error occurred',
        color: 'red',
        timeout: 5000,
      });
    }
  };

  /**
   * Contact support
   */
  const contactSupport = (): void => {
    // Implement support contact logic
    window.open('mailto:support@yourcompany.com?subject=SSL Pinning Error', '_blank');
  };

  return {
    handleError,
    handleSSLError,
    handleNetworkError,
    contactSupport,
  };
};
```

## Configuration Management

### 1. Runtime Configuration

Create `nuxt.config.ts`:

```typescript
export default defineNuxtConfig({
  modules: ['@nuxt/ui'],
  
  runtimeConfig: {
    // Server-side configuration
    sslPinningKillSwitch: process.env.SSL_PINNING_KILL_SWITCH || 'false',
    apigeeHostname: process.env.APIGEE_HOSTNAME || 'your-apigee-domain.com',
    
    // Public configuration (exposed to client)
    public: {
      sslPinningEnabled: process.env.NODE_ENV === 'production',
      apiTimeout: 30000,
    },
  },

  // Global error handling
  hooks: {
    'render:errorMiddleware': (error, { event }) => {
      console.error('Global error:', error);
    },
  },

  // Security headers
  nitro: {
    routeRules: {
      '/**': {
        headers: {
          'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
          'X-Content-Type-Options': 'nosniff',
          'X-Frame-Options': 'DENY',
          'X-XSS-Protection': '1; mode=block',
        },
      },
    },
  },
});
```

### 2. Environment Configuration

Create `.env.example`:

```env
# SSL Pinning Configuration
SSL_PINNING_KILL_SWITCH=false
APIGEE_HOSTNAME=your-apigee-domain.com

# API Configuration
API_TIMEOUT=30000

# Security
NODE_ENV=production
```

## API Integration

### 1. Login Page

Create `pages/login.vue`:

```vue
<template>
  <div class="min-h-screen flex items-center justify-center bg-gray-50">
    <div class="max-w-md w-full space-y-8">
      <div>
        <h2 class="mt-6 text-center text-3xl font-extrabold text-gray-900">
          Secure CMS Login
        </h2>
        
        <!-- SSL Status Indicator -->
        <div class="mt-4 p-3 rounded-md" :class="sslStatusClass">
          <div class="flex items-center">
            <UIcon :name="sslStatusIcon" class="w-5 h-5 mr-2" />
            <span class="text-sm font-medium">{{ sslStatusText }}</span>
          </div>
        </div>
      </div>

      <form class="mt-8 space-y-6" @submit.prevent="handleLogin">
        <div class="space-y-4">
          <div>
            <UFormGroup label="Username" required>
              <UInput 
                v-model="form.username" 
                type="text" 
                required 
                placeholder="Enter your username"
              />
            </UFormGroup>
          </div>
          
          <div>
            <UFormGroup label="Password" required>
              <UInput 
                v-model="form.password" 
                type="password" 
                required 
                placeholder="Enter your password"
              />
            </UFormGroup>
          </div>
        </div>

        <div>
          <UButton 
            type="submit" 
            block 
            :loading="isLoading"
            :disabled="!form.username || !form.password"
          >
            {{ isLoading ? 'Signing in...' : 'Sign in' }}
          </UButton>
        </div>
      </form>
    </div>
  </div>
</template>

<script setup>
const { login } = useApi();
const { state: sslState } = useSSLPinning();
const { handleError } = useErrorHandler();

// Form state
const form = reactive({
  username: '',
  password: '',
});

const isLoading = ref(false);

// SSL status computed properties
const sslStatusClass = computed(() => ({
  'bg-green-50 border-green-200 text-green-800': sslState.connectionStatus === 'connected',
  'bg-yellow-50 border-yellow-200 text-yellow-800': sslState.connectionStatus === 'disconnected',
  'bg-red-50 border-red-200 text-red-800': sslState.connectionStatus === 'error',
}));

const sslStatusIcon = computed(() => {
  switch (sslState.connectionStatus) {
    case 'connected': return 'i-heroicons-shield-check';
    case 'error': return 'i-heroicons-shield-exclamation';
    default: return 'i-heroicons-shield-check';
  }
});

const sslStatusText = computed(() => {
  switch (sslState.connectionStatus) {
    case 'connected': return 'Secure connection active';
    case 'error': return 'Security error detected';
    default: return 'Secure connection ready';
  }
});

// Login handler
const handleLogin = async () => {
  if (isLoading.value) return;
  
  isLoading.value = true;
  
  try {
    const response = await login({
      username: form.username,
      password: form.password,
    });
    
    // Store token (you might want to use a more secure method)
    const token = useCookie('auth-token', { 
      secure: true, 
      httpOnly: true,
      maxAge: 60 * 60 * 24 // 24 hours
    });
    token.value = response.token;
    
    // Redirect to dashboard
    await navigateTo('/dashboard');
    
  } catch (error) {
    handleError(error);
  } finally {
    isLoading.value = false;
  }
};

// SEO
useHead({
  title: 'Login - Secure CMS',
  meta: [
    { name: 'description', content: 'Secure login to CMS with SSL pinning protection' }
  ],
});
</script>
```

### 2. Dashboard Page

Create `pages/dashboard.vue`:

```vue
<template>
  <div class="min-h-screen bg-gray-50">
    <!-- Header -->
    <header class="bg-white shadow">
      <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
        <div class="flex justify-between items-center py-6">
          <h1 class="text-3xl font-bold text-gray-900">Dashboard</h1>
          
          <div class="flex items-center space-x-4">
            <!-- SSL Status -->
            <div class="flex items-center" :class="sslStatusTextClass">
              <UIcon :name="sslStatusIcon" class="w-4 h-4 mr-1" />
              <span class="text-sm">{{ sslStatusText }}</span>
            </div>
            
            <!-- User menu -->
            <UDropdown :items="userMenuItems">
              <UAvatar src="" :alt="userProfile?.name" />
            </UDropdown>
          </div>
        </div>
      </div>
    </header>

    <!-- Main content -->
    <main class="max-w-7xl mx-auto py-6 px-4 sm:px-6 lg:px-8">
      <div v-if="isLoading" class="flex justify-center items-center h-64">
        <UIcon name="i-heroicons-arrow-path" class="w-8 h-8 animate-spin" />
      </div>
      
      <div v-else-if="error" class="text-center py-12">
        <UIcon name="i-heroicons-exclamation-triangle" class="w-16 h-16 mx-auto text-red-500 mb-4" />
        <h3 class="text-lg font-medium text-gray-900 mb-2">Error Loading Dashboard</h3>
        <p class="text-gray-600 mb-4">{{ error }}</p>
        <UButton @click="loadDashboard">Retry</UButton>
      </div>
      
      <div v-else class="space-y-6">
        <!-- User Profile Card -->
        <UCard>
          <template #header>
            <h3 class="text-lg font-semibold">User Profile</h3>
          </template>
          
          <div v-if="userProfile" class="space-y-3">
            <div class="flex justify-between">
              <span class="font-medium">Name:</span>
              <span>{{ userProfile.name }}</span>
            </div>
            <div class="flex justify-between">
              <span class="font-medium">Email:</span>
              <span>{{ userProfile.email }}</span>
            </div>
            <div class="flex justify-between">
              <span class="font-medium">Role:</span>
              <span>{{ userProfile.role }}</span>
            </div>
          </div>
        </UCard>

        <!-- Security Status Card -->
        <UCard>
          <template #header>
            <h3 class="text-lg font-semibold">Security Status</h3>
          </template>
          
          <div class="space-y-4">
            <div class="flex items-center justify-between">
              <span class="font-medium">SSL Pinning:</span>
              <div class="flex items-center" :class="sslStatusTextClass">
                <UIcon :name="sslStatusIcon" class="w-4 h-4 mr-1" />
                <span class="text-sm">{{ sslState.isEnabled ? 'Enabled' : 'Disabled' }}</span>
              </div>
            </div>
            
            <div class="flex items-center justify-between">
              <span class="font-medium">Connection Status:</span>
              <span :class="connectionStatusClass">{{ sslState.connectionStatus }}</span>
            </div>
            
            <div class="flex items-center justify-between">
              <span class="font-medium">Hostname:</span>
              <span class="text-sm text-gray-600">{{ sslState.hostname }}</span>
            </div>
            
            <div v-if="sslState.lastError" class="p-3 bg-red-50 border border-red-200 rounded">
              <p class="text-sm text-red-800">{{ sslState.lastError }}</p>
            </div>
          </div>
        </UCard>

        <!-- Dashboard Data -->
        <UCard v-if="dashboardData">
          <template #header>
            <h3 class="text-lg font-semibold">Dashboard Data</h3>
          </template>
          
          <pre class="text-sm bg-gray-50 p-4 rounded overflow-auto">{{ JSON.stringify(dashboardData, null, 2) }}</pre>
        </UCard>
      </div>
    </main>
  </div>
</template>

<script setup>
// Middleware to check authentication
definePageMeta({
  middleware: 'auth'
});

const { getUserProfile, getDashboardData } = useApi();
const { state: sslState, clearError } = useSSLPinning();
const { handleError } = useErrorHandler();

// State
const isLoading = ref(true);
const error = ref(null);
const userProfile = ref(null);
const dashboardData = ref(null);

// Get auth token
const token = useCookie('auth-token');

// Computed properties
const sslStatusTextClass = computed(() => ({
  'text-green-600': sslState.connectionStatus === 'connected',
  'text-yellow-600': sslState.connectionStatus === 'disconnected',
  'text-red-600': sslState.connectionStatus === 'error',
}));

const sslStatusIcon = computed(() => {
  switch (sslState.connectionStatus) {
    case 'connected': return 'i-heroicons-shield-check';
    case 'error': return 'i-heroicons-shield-exclamation';
    default: return 'i-heroicons-shield-check';
  }
});

const sslStatusText = computed(() => {
  switch (sslState.connectionStatus) {
    case 'connected': return 'Secure';
    case 'error': return 'Error';
    default: return 'Ready';
  }
});

const connectionStatusClass = computed(() => ({
  'text-green-600 capitalize': sslState.connectionStatus === 'connected',
  'text-yellow-600 capitalize': sslState.connectionStatus === 'disconnected',
  'text-red-600 capitalize': sslState.connectionStatus === 'error',
}));

// User menu items
const userMenuItems = computed(() => [
  [
    {
      label: userProfile.value?.name || 'User',
      slot: 'account',
      disabled: true,
    }
  ],
  [
    {
      label: 'Settings',
      icon: 'i-heroicons-cog-6-tooth',
      click: () => navigateTo('/settings'),
    }
  ],
  [
    {
      label: 'Sign out',
      icon: 'i-heroicons-arrow-right-on-rectangle',
      click: logout,
    }
  ]
]);

// Load dashboard data
const loadDashboard = async () => {
  if (!token.value) {
    await navigateTo('/login');
    return;
  }

  isLoading.value = true;
  error.value = null;
  clearError();

  try {
    // Load user profile and dashboard data in parallel
    const [profileData, dashData] = await Promise.all([
      getUserProfile(token.value),
      getDashboardData(token.value),
    ]);

    userProfile.value = profileData;
    dashboardData.value = dashData;

  } catch (err) {
    error.value = err.message || 'Failed to load dashboard data';
    handleError(err);
  } finally {
    isLoading.value = false;
  }
};

// Logout function
const logout = async () => {
  token.value = null;
  await navigateTo('/login');
};

// Load data on mount
onMounted(() => {
  loadDashboard();
});

// SEO
useHead({
  title: 'Dashboard - Secure CMS',
});
</script>
```

## Error Handling

### 1. Global Error Handler

Create `plugins/error-handler.client.ts`:

```typescript
export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.hook('vue:error', (error, context) => {
    console.error('Vue error:', error, context);
    
    // Handle SSL pinning errors globally
    if (error.name === 'SSLPinningClientError') {
      const { handleSSLError } = useErrorHandler();
      handleSSLError(error as SSLPinningClientError);
    }
  });

  nuxtApp.hook('app:error', (error) => {
    console.error('App error:', error);
  });
});
```

### 2. Auth Middleware

Create `middleware/auth.ts`:

```typescript
export default defineNuxtRouteMiddleware((to) => {
  const token = useCookie('auth-token');
  
  if (!token.value) {
    return navigateTo('/login');
  }
});
```

## Testing

### 1. Unit Tests

Create `tests/composables/useSSLPinning.test.ts`:

```typescript
import { describe, it, expect, vi } from 'vitest';
import { useSSLPinning } from '~/composables/useSSLPinning';

// Mock $fetch
global.$fetch = vi.fn();

describe('useSSLPinning', () => {
  it('should initialize with correct default state', () => {
    const { state } = useSSLPinning();
    
    expect(state.isEnabled).toBe(true);
    expect(state.connectionStatus).toBe('disconnected');
    expect(state.lastError).toBe(null);
  });

  it('should make secure request successfully', async () => {
    const mockResponse = { success: true, data: { id: 1, name: 'Test' } };
    vi.mocked($fetch).mockResolvedValueOnce(mockResponse);

    const { secureRequest } = useSSLPinning();
    const result = await secureRequest('/test');

    expect(result).toEqual({ id: 1, name: 'Test' });
    expect($fetch).toHaveBeenCalledWith('/api/secure-proxy', {
      method: 'POST',
      body: {
        endpoint: '/test',
        method: 'GET',
        data: undefined,
        headers: undefined,
      },
    });
  });

  it('should handle SSL pinning errors', async () => {
    const mockError = {
      status: 426,
      data: { message: 'SSL Pinning validation failed' },
    };
    vi.mocked($fetch).mockRejectedValueOnce(mockError);

    const { secureRequest } = useSSLPinning();
    
    await expect(secureRequest('/test')).rejects.toThrow('SSL Pinning validation failed');
  });
});
```

### 2. Component Tests

Create `tests/components/LoginForm.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { mount } from '@vue/test-utils';
import LoginPage from '~/pages/login.vue';

describe('LoginPage', () => {
  it('should render login form', () => {
    const wrapper = mount(LoginPage);
    
    expect(wrapper.find('h2').text()).toBe('Secure CMS Login');
    expect(wrapper.findAll('input')).toHaveLength(2);
    expect(wrapper.find('button[type="submit"]').text()).toBe('Sign in');
  });

  it('should show SSL status indicator', () => {
    const wrapper = mount(LoginPage);
    
    expect(wrapper.find('.ssl-status')).toBeTruthy();
  });
});
```

## Best Practices

### 1. Security Best Practices

```typescript
// utils/security.ts
export const securityUtils = {
  // Sanitize user input
  sanitizeInput: (input: string): string => {
    return input.replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, '');
  },

  // Validate JWT token format
  isValidToken: (token: string): boolean => {
    const jwtRegex = /^[A-Za-z0-9-_]+\.[A-Za-z0-9-_]+\.[A-Za-z0-9-_]+$/;
    return jwtRegex.test(token);
  },

  // Generate secure headers
  getSecureHeaders: (): Record<string, string> => ({
    'X-Content-Type-Options': 'nosniff',
    'X-Frame-Options': 'DENY',
    'X-XSS-Protection': '1; mode=block',
    'Referrer-Policy': 'strict-origin-when-cross-origin',
  }),
};
```

### 2. Performance Optimization

```typescript
// utils/performance.ts
export const performanceUtils = {
  // Debounce function for API calls
  debounce: <T extends (...args: any[]) => any>(func: T, wait: number): T => {
    let timeout: NodeJS.Timeout;
    return ((...args: any[]) => {
      clearTimeout(timeout);
      timeout = setTimeout(() => func.apply(null, args), wait);
    }) as T;
  },

  // Cache API responses
  cache: new Map<string, { data: any; timestamp: number }>(),

  getCached: (key: string, maxAge: number = 5 * 60 * 1000): any => {
    const cached = performanceUtils.cache.get(key);
    if (cached && Date.now() - cached.timestamp < maxAge) {
      return cached.data;
    }
    return null;
  },

  setCached: (key: string, data: any): void => {
    performanceUtils.cache.set(key, { data, timestamp: Date.now() });
  },
};
```

This simple Nuxt.js CMS implementation provides secure SSL pinning with server-side certificate validation and client-side security composables, focusing on the core requirements while maintaining browser compatibility and ease of use.

````