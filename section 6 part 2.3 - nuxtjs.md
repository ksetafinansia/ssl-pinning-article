# Nuxt.js Kill Switch Implementation (JavaScript)

This guide shows how to implement Firebase Remote Config kill switch for SSL pinning in Nuxt.js server-side applications using Node.js and Firebase Admin SDK.

## Prerequisites
- Existing SSL pinning implementation from **Section 6 Part 1**
- Firebase project with Remote Config enabled
- Firebase Admin SDK service account credentials
- Nuxt.js application with server-side functionality

## Step 1: Firebase Admin SDK Setup

### Environment Configuration
```bash
# .env
FIREBASE_PROJECT_ID=your-project-id
GOOGLE_APPLICATION_CREDENTIALS=path/to/service-account-key.json
```

### Service Account Setup
1. Go to Firebase Console → Project Settings → Service Accounts
2. Generate new private key
3. Download JSON file and place in secure location
4. Set `GOOGLE_APPLICATION_CREDENTIALS` environment variable

## Step 2: Server-Side Remote Config Manager

### Remote Config Manager
```javascript
// server/utils/remote-config.js
import admin from 'firebase-admin'

class ServerRemoteConfigManager {
  constructor() {
    if (!admin.apps.length) {
      admin.initializeApp({
        credential: admin.credential.applicationDefault(),
        projectId: process.env.FIREBASE_PROJECT_ID
      })
    }
    this.remoteConfig = admin.remoteConfig()
    this.configCache = null
    this.lastFetch = 0
    this.cacheTimeout = 5 * 60 * 1000 // 5 minutes
  }

  async getSSLPinningConfig() {
    // Check cache first
    if (this.configCache && (Date.now() - this.lastFetch) < this.cacheTimeout) {
      return this.configCache
    }

    try {
      const template = await this.remoteConfig.getTemplate()
      const configParam = template.parameters['ssl_pinning_config']
      
      if (configParam && configParam.defaultValue) {
        let config = JSON.parse(configParam.defaultValue.value)
        
        // Handle both array and comma-separated string formats for target_hosts
        if (typeof config.target_hosts === 'string') {
          config.target_hosts = config.target_hosts
            .split(',')
            .map(host => host.trim())
            .filter(host => host.length > 0)
        }
        
        // Cache the result
        this.configCache = config
        this.lastFetch = Date.now()
        
        return config
      }
      
      return this.getDefaultConfig()
    } catch (error) {
      console.error('Failed to fetch remote config:', error)
      
      // Return cached config if available, otherwise default
      return this.configCache || this.getDefaultConfig()
    }
  }

  getDefaultConfig() {
    return {
      enabled: false,
      target_hosts: [],
      min_app_version: '1.0.0'
    }
  }

  async shouldEnablePinning(appVersion = '1.0.0') {
    const config = await this.getSSLPinningConfig()

    // Global enable check
    if (!config.enabled) {
      console.log('SSL pinning disabled by config')
      return false
    }

    // App version check
    if (!this.compareVersions(appVersion, config.min_app_version)) {
      console.log(`SSL pinning disabled: app version ${appVersion} < required ${config.min_app_version}`)
      return false
    }

    // Check if we have any target hosts
    if (!config.target_hosts || config.target_hosts.length === 0) {
      console.log('SSL pinning disabled: no target hosts configured')
      return false
    }

    return true
  }

  shouldPinHost(hostname, config) {
    return config && config.target_hosts && config.target_hosts.includes(hostname)
  }

  compareVersions(version1, version2) {
    const v1Parts = version1.split('.').map(num => parseInt(num) || 0)
    const v2Parts = version2.split('.').map(num => parseInt(num) || 0)
    const maxLength = Math.max(v1Parts.length, v2Parts.length)
    
    for (let i = 0; i < maxLength; i++) {
      const v1Part = v1Parts[i] || 0
      const v2Part = v2Parts[i] || 0
      if (v1Part > v2Part) return true
      if (v1Part < v2Part) return false
    }
    return true // Equal versions
  }

  // Force refresh cache
  clearCache() {
    this.configCache = null
    this.lastFetch = 0
  }
}

export default new ServerRemoteConfigManager()
```

## Step 3: Enhanced HTTP Client with Remote Config

### HTTP Client Factory
```javascript
// server/utils/http-client.js
import https from 'https'
import fs from 'fs'
import remoteConfigManager from './remote-config.js'

class EnhancedHttpClient {
  constructor() {
    this.buildTimePins = {
      'apigee.kreditplus.com': [
        'your_build_time_primary_pin',
        'your_build_time_backup_pin'
      ],
      'apigee.kbfinansia.com': [
        'your_staging_pin'
      ]
    }
  }

  async createHttpsAgent(hostname, appVersion = '1.0.0') {
    const config = await remoteConfigManager.getSSLPinningConfig()
    const shouldPin = await remoteConfigManager.shouldEnablePinning(appVersion)
    const shouldPinThisHost = remoteConfigManager.shouldPinHost(hostname, config)

    if (shouldPin && shouldPinThisHost) {
      return this.createPinnedAgent(hostname)
    } else {
      return this.createNormalAgent()
    }
  }

  createPinnedAgent(hostname) {
    const pins = this.buildTimePins[hostname]
    
    if (!pins || pins.length === 0) {
      console.warn(`No pins configured for ${hostname}, using normal TLS`)
      return this.createNormalAgent()
    }

    return new https.Agent({
      checkServerIdentity: (host, cert) => {
        // Extract public key from certificate
        const publicKey = cert.pubkey
        const publicKeyHash = this.sha256(publicKey).toString('base64')
        
        // Check if public key hash matches any of our pins
        if (pins.includes(publicKeyHash)) {
          return undefined // Valid pin
        } else {
          throw new Error(`Certificate pin validation failed for ${host}`)
        }
      },
      keepAlive: true,
      timeout: 30000
    })
  }

  createNormalAgent() {
    return new https.Agent({
      keepAlive: true,
      timeout: 30000
    })
  }

  sha256(data) {
    const crypto = require('crypto')
    return crypto.createHash('sha256').update(data).digest()
  }
}

export default new EnhancedHttpClient()
```

## Step 4: Nuxt.js Server API Integration

### Server API Route with Remote Config
```javascript
// server/api/secure-data.get.js
import httpClientFactory from '~/server/utils/http-client.js'
import remoteConfigManager from '~/server/utils/remote-config.js'

export default defineEventHandler(async (event) => {
  try {
    const query = getQuery(event)
    const appVersion = query.app_version || '1.0.0'
    const hostname = 'apigee.kreditplus.com'
    
    // Create HTTPS agent with remote config control
    const agent = await httpClientFactory.createHttpsAgent(hostname, appVersion)
    
    // Make request with appropriate SSL configuration
    const response = await fetch(`https://${hostname}/api/v1/secure-data`, {
      agent,
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${getAuthToken(event)}`
      }
    })
    
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`)
    }
    
    const data = await response.json()
    
    return {
      success: true,
      data: data,
      pinning_enabled: await remoteConfigManager.shouldEnablePinning(appVersion)
    }
    
  } catch (error) {
    console.error('Secure API call failed:', error)
    
    throw createError({
      statusCode: 500,
      statusMessage: 'Failed to fetch secure data',
      data: {
        error: error.message,
        pinning_enabled: false
      }
    })
  }
})

function getAuthToken(event) {
  const authHeader = getHeader(event, 'authorization')
  return authHeader?.replace('Bearer ', '') || ''
}
```

### Configuration Status API
```javascript
// server/api/ssl-config.get.js
import remoteConfigManager from '~/server/utils/remote-config.js'

export default defineEventHandler(async (event) => {
  try {
    const query = getQuery(event)
    const appVersion = query.app_version || '1.0.0'
    
    const config = await remoteConfigManager.getSSLPinningConfig()
    const shouldEnable = await remoteConfigManager.shouldEnablePinning(appVersion)
    
    return {
      ssl_pinning_config: config,
      should_enable_pinning: shouldEnable,
      app_version: appVersion,
      timestamp: new Date().toISOString()
    }
    
  } catch (error) {
    console.error('Failed to get SSL config:', error)
    
    throw createError({
      statusCode: 500,
      statusMessage: 'Failed to retrieve SSL configuration'
    })
  }
})
```

### Configuration Refresh API
```javascript
// server/api/refresh-config.post.js
import remoteConfigManager from '~/server/utils/remote-config.js'

export default defineEventHandler(async (event) => {
  try {
    // Clear cache to force fresh fetch
    remoteConfigManager.clearCache()
    
    // Fetch fresh configuration
    const config = await remoteConfigManager.getSSLPinningConfig()
    
    return {
      success: true,
      message: 'Configuration refreshed successfully',
      config: config,
      timestamp: new Date().toISOString()
    }
    
  } catch (error) {
    console.error('Failed to refresh config:', error)
    
    throw createError({
      statusCode: 500,
      statusMessage: 'Failed to refresh configuration'
    })
  }
})
```

## Step 5: Client-Side Integration

### Composable for SSL Config
```javascript
// composables/useSSLConfig.js
export const useSSLConfig = () => {
  const config = ref(null)
  const loading = ref(false)
  const error = ref(null)

  const fetchConfig = async (appVersion = '1.0.0') => {
    loading.value = true
    error.value = null
    
    try {
      const { data } = await $fetch('/api/ssl-config', {
        query: { app_version: appVersion }
      })
      
      config.value = data
      return data
    } catch (err) {
      error.value = err
      console.error('Failed to fetch SSL config:', err)
      throw err
    } finally {
      loading.value = false
    }
  }

  const refreshConfig = async () => {
    loading.value = true
    error.value = null
    
    try {
      const { data } = await $fetch('/api/refresh-config', {
        method: 'POST'
      })
      
      // Fetch updated config
      await fetchConfig()
      return data
    } catch (err) {
      error.value = err
      console.error('Failed to refresh config:', err)
      throw err
    } finally {
      loading.value = false
    }
  }

  return {
    config: readonly(config),
    loading: readonly(loading),
    error: readonly(error),
    fetchConfig,
    refreshConfig
  }
}
```

### Vue Component Usage
```vue
<!-- pages/admin/ssl-config.vue -->
<template>
  <div class="ssl-config-admin">
    <h1>SSL Pinning Configuration</h1>
    
    <div v-if="loading" class="loading">
      Loading configuration...
    </div>
    
    <div v-else-if="error" class="error">
      Error: {{ error.message }}
    </div>
    
    <div v-else-if="config" class="config-display">
      <div class="config-section">
        <h2>Current Configuration</h2>
        <pre>{{ JSON.stringify(config.ssl_pinning_config, null, 2) }}</pre>
      </div>
      
      <div class="status-section">
        <h2>Status</h2>
        <p>
          <strong>Pinning Enabled:</strong> 
          <span :class="config.should_enable_pinning ? 'enabled' : 'disabled'">
            {{ config.should_enable_pinning ? 'Yes' : 'No' }}
          </span>
        </p>
        <p><strong>App Version:</strong> {{ config.app_version }}</p>
        <p><strong>Last Updated:</strong> {{ new Date(config.timestamp).toLocaleString() }}</p>
      </div>
      
      <div class="actions">
        <button @click="refreshConfiguration" :disabled="loading" class="btn-refresh">
          Refresh Configuration
        </button>
        
        <button @click="testConnection" :disabled="loading" class="btn-test">
          Test Connection
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
const { config, loading, error, fetchConfig, refreshConfig } = useSSLConfig()

onMounted(() => {
  fetchConfig()
})

const refreshConfiguration = async () => {
  try {
    await refreshConfig()
    // Show success notification
  } catch (err) {
    // Show error notification
  }
}

const testConnection = async () => {
  try {
    const { data } = await $fetch('/api/secure-data')
    // Show success notification
  } catch (err) {
    // Show error notification
  }
}
</script>

<style scoped>
.ssl-config-admin {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.config-section, .status-section {
  margin-bottom: 30px;
  padding: 20px;
  border: 1px solid #ddd;
  border-radius: 8px;
}

.enabled {
  color: green;
  font-weight: bold;
}

.disabled {
  color: red;
  font-weight: bold;
}

.actions {
  display: flex;
  gap: 10px;
}

.btn-refresh, .btn-test {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.btn-refresh {
  background-color: #007bff;
  color: white;
}

.btn-test {
  background-color: #28a745;
  color: white;
}

button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

pre {
  background-color: #f8f9fa;
  padding: 15px;
  border-radius: 4px;
  overflow-x: auto;
}
</style>
```

## Step 6: Migration from Part 1 to Part 2

### Before (Part 1 - Basic Implementation)
```javascript
// Basic pinning - always enabled
import https from 'https'

const pinnedAgent = new https.Agent({
  checkServerIdentity: (host, cert) => {
    // Always perform pinning validation
    return validateCertificatePin(host, cert)
  }
})

export default pinnedAgent
```

### After (Part 2 - Enhanced Implementation)
```javascript
// Enhanced with remote config control
import remoteConfigManager from './remote-config.js'

const createAgent = async (hostname, appVersion) => {
  // NEW: Check remote config first
  const shouldPin = await remoteConfigManager.shouldEnablePinning(appVersion)
  const config = await remoteConfigManager.getSSLPinningConfig()
  
  if (shouldPin && remoteConfigManager.shouldPinHost(hostname, config)) {
    // Use existing Part 1 pinning logic
    return createPinnedAgent(hostname)
  } else {
    // NEW: Bypass pinning capability
    return createNormalAgent()
  }
}
```

## Dependencies

### Add to package.json
```json
{
  "dependencies": {
    "firebase-admin": "^11.11.0",
    "nuxt": "^3.8.0"
  },
  "devDependencies": {
    "@nuxt/test-utils": "^3.8.0"
  }
}
```

### Nuxt Configuration
```javascript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    firebaseProjectId: process.env.FIREBASE_PROJECT_ID,
    googleApplicationCredentials: process.env.GOOGLE_APPLICATION_CREDENTIALS,
    
    public: {
      appVersion: process.env.APP_VERSION || '1.0.0'
    }
  },
  
  serverHandlers: [
    {
      route: '/api/**',
      handler: '~/server/api/**'
    }
  ]
})
```

## Testing

### Unit Test Example
```javascript
// tests/remote-config.test.js
import { describe, it, expect, beforeEach, vi } from 'vitest'
import remoteConfigManager from '../server/utils/remote-config.js'

// Mock Firebase Admin
vi.mock('firebase-admin', () => ({
  apps: [],
  initializeApp: vi.fn(),
  credential: {
    applicationDefault: vi.fn()
  },
  remoteConfig: vi.fn(() => ({
    getTemplate: vi.fn()
  }))
}))

describe('RemoteConfigManager', () => {
  beforeEach(() => {
    remoteConfigManager.clearCache()
  })

  it('should disable pinning when config enabled is false', async () => {
    // Mock remote config response
    const mockConfig = {
      enabled: false,
      target_hosts: ['apigee.kreditplus.com'],
      min_app_version: '1.0.0'
    }
    
    // Test
    const shouldEnable = await remoteConfigManager.shouldEnablePinning('1.5.0')
    expect(shouldEnable).toBe(false)
  })

  it('should compare versions correctly', () => {
    expect(remoteConfigManager.compareVersions('1.5.0', '1.4.9')).toBe(true)
    expect(remoteConfigManager.compareVersions('1.5.0', '1.5.0')).toBe(true)
    expect(remoteConfigManager.compareVersions('1.4.9', '1.5.0')).toBe(false)
  })
})
```

### Integration Test
```javascript
// tests/api.test.js
import { describe, it, expect } from 'vitest'
import { setup, $fetch } from '@nuxt/test-utils'

describe('SSL Config API', async () => {
  await setup({
    // test config
  })

  it('should return SSL configuration', async () => {
    const response = await $fetch('/api/ssl-config?app_version=1.5.0')
    
    expect(response).toHaveProperty('ssl_pinning_config')
    expect(response).toHaveProperty('should_enable_pinning')
    expect(response.app_version).toBe('1.5.0')
  })

  it('should refresh configuration', async () => {
    const response = await $fetch('/api/refresh-config', {
      method: 'POST'
    })
    
    expect(response.success).toBe(true)
    expect(response).toHaveProperty('config')
  })
})
```

## Best Practices

### 1. **Error Handling**
- Always provide safe defaults when Remote Config fails
- Don't crash the server if configuration parsing fails
- Log configuration errors for debugging

### 2. **Performance**
- Cache configuration responses (5-minute cache implemented)
- Use connection pooling with `keepAlive: true`
- Consider background configuration refresh

### 3. **Security**
- Keep certificate pins embedded in the server code
- Secure service account credentials properly
- Use Firebase Remote Config only for control logic

### 4. **Monitoring**
- Log SSL pinning successes and failures
- Monitor configuration fetch failures
- Set up alerts for kill switch activations

**📋 For configuration management, emergency procedures, and operational guidance, see Section 10.**
