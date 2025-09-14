# Server Public Key Extraction Guide

## Overview

This guide provides step-by-step instructions for obtaining Apigee server certificate information and generating public key hashes for SSL pinning. We use public key pinning as it's more robust and flexible - the public key remains the same even if the certificate is reissued.

## Prerequisites

- Access to the Apigee server
- OpenSSL installed on your system
- Command line access
- Network connectivity to the target server

## Step 1: Extract Certificate from Server

### Method 1: Using OpenSSL (Recommended)

Extract the certificate directly from the running server:

```bash
# Replace your-apigee-domain.com with your actual Apigee domain
echo | openssl s_client -servername your-apigee-domain.com -connect your-apigee-domain.com:443 -showcerts 2>/dev/null | openssl x509 -outform PEM > apigee-cert.pem
```

### Method 2: Using Browser (Alternative)

1. Open your browser and navigate to your Apigee endpoint (e.g., `https://your-apigee-domain.com`)
2. Click on the lock icon in the address bar
3. Click "Certificate" or "View Certificate"
4. Go to the "Details" tab
5. Click "Export" and save as PEM format

### Method 3: Using curl + OpenSSL

```bash
# Get certificate chain
curl -s -I https://your-apigee-domain.com | head -1

# Extract certificate
echo | openssl s_client -servername your-apigee-domain.com -connect your-apigee-domain.com:443 2>/dev/null | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > apigee-cert.pem
```

## Step 2: Extract Public Key from Certificate

Once you have the certificate file (`apigee-cert.pem`), extract the public key:

```bash
# Extract public key from certificate
openssl x509 -in apigee-cert.pem -pubkey -noout > apigee-public-key.pem
```

## Step 3: Generate Public Key Hash (SPKI)

Generate the SHA-256 hash of the Subject Public Key Info (SPKI):

```bash
# Generate SPKI SHA-256 hash
openssl pkey -in apigee-public-key.pem -pubin -outform DER | openssl dgst -sha256 -binary | openssl enc -base64
```

**Alternative one-liner method:**

```bash
# Direct extraction and hashing
echo | openssl s_client -servername your-apigee-domain.com -connect your-apigee-domain.com:443 2>/dev/null | openssl x509 -pubkey -noout | openssl pkey -pubin -outform DER | openssl dgst -sha256 -binary | openssl enc -base64
```

## Step 4: Generate Backup Key Hash

For resilience, you should also generate a backup key hash. This is typically done by creating a new key pair for future certificate rotation:

```bash
# Generate backup private key
openssl genrsa -out backup-private-key.pem 2048

# Generate backup public key
openssl rsa -in backup-private-key.pem -pubout -out backup-public-key.pem

# Generate backup key hash
openssl pkey -in backup-public-key.pem -pubin -outform DER | openssl dgst -sha256 -binary | openssl enc -base64
```

## Step 5: Verify Certificate Chain

Verify the complete certificate chain to ensure you're pinning the correct certificate:

```bash
# Get the full certificate chain
echo | openssl s_client -servername your-apigee-domain.com -connect your-apigee-domain.com:443 -showcerts 2>/dev/null
```

## Step 6: Create Pin Configuration

Create a configuration file with your pins:

```json
{
  "pins": {
    "your-apigee-domain.com": [
      "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",
      "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB="
    ]
  },
  "backup_pins": {
    "your-apigee-domain.com": [
      "sha256/CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC="
    ]
  }
}
```

Replace the placeholder hashes with your actual generated hashes.

## Step 7: Implement in Apigee (Server-Side Validation)

### Apigee Edge Policy Configuration

Create a JavaScript policy in Apigee to validate client certificates:

```javascript
// JavaScript Policy: validate-client-certificate.js
var clientCert = context.getVariable("request.header.X-Client-Certificate");
var pinnedHashes = [
    "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",
    "BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB="
];

function validateCertificatePin(certificate, pinnedHashes) {
    // Extract public key from client certificate
    // Calculate SHA-256 hash
    // Compare with pinned hashes
    // Return validation result
    
    // This is a simplified example - implement according to your needs
    return pinnedHashes.indexOf(certificate) !== -1;
}

if (!validateCertificatePin(clientCert, pinnedHashes)) {
    context.setVariable("pinning.validation.failed", true);
    throw new Error("Certificate pinning validation failed");
}
```

### Apigee Proxy Configuration

Add the policy to your API proxy:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ProxyEndpoint name="default">
    <Description/>
    <FaultRules/>
    <PreFlow name="PreFlow">
        <Request>
            <Step>
                <Name>validate-client-certificate</Name>
                <Condition>request.header.X-Client-Certificate != null</Condition>
            </Step>
        </Request>
        <Response/>
    </PreFlow>
    <PostFlow name="PostFlow">
        <Request/>
        <Response/>
    </PostFlow>
    <Flows/>
    <HTTPProxyConnection>
        <BasePath>/api/v1</BasePath>
        <VirtualHost>secure</VirtualHost>
    </HTTPProxyConnection>
    <RouteRule name="default">
        <TargetEndpoint>default</TargetEndpoint>
    </RouteRule>
</ProxyEndpoint>
```

## Automation Script

Create a script to automate the hash generation process:

```bash
#!/bin/bash

# extract-public-key-hash.sh
# Usage: ./extract-public-key-hash.sh your-apigee-domain.com

DOMAIN=$1
OUTPUT_FILE="pin-hashes-$(date +%Y%m%d).txt"

if [ -z "$DOMAIN" ]; then
    echo "Usage: $0 <domain>"
    exit 1
fi

echo "Extracting public key hash for: $DOMAIN"
echo "Output file: $OUTPUT_FILE"

# Extract and hash public key
HASH=$(echo | openssl s_client -servername $DOMAIN -connect $DOMAIN:443 2>/dev/null | openssl x509 -pubkey -noout | openssl pkey -pubin -outform DER | openssl dgst -sha256 -binary | openssl enc -base64)

# Save to file
echo "Domain: $DOMAIN" > $OUTPUT_FILE
echo "Hash: sha256/$HASH" >> $OUTPUT_FILE
echo "Generated: $(date)" >> $OUTPUT_FILE

echo "Public key hash: sha256/$HASH"
echo "Saved to: $OUTPUT_FILE"

# Verify certificate info
echo ""
echo "Certificate information:"
echo | openssl s_client -servername $DOMAIN -connect $DOMAIN:443 2>/dev/null | openssl x509 -text -noout | grep -E "(Subject:|Issuer:|Not Before:|Not After:)"
```

Make the script executable:

```bash
chmod +x extract-public-key-hash.sh
```

Run the script:

```bash
./extract-public-key-hash.sh your-apigee-domain.com
```

## CI/CD Integration

### GitHub Actions Example

Create a workflow to automatically extract and update pin hashes:

```yaml
name: Extract SSL Pin Hashes

on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly check
  workflow_dispatch:

jobs:
  extract-hashes:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      
    - name: Extract pin hashes
      run: |
        DOMAIN="${{ secrets.APIGEE_DOMAIN }}"
        HASH=$(echo | openssl s_client -servername $DOMAIN -connect $DOMAIN:443 2>/dev/null | openssl x509 -pubkey -noout | openssl pkey -pubin -outform DER | openssl dgst -sha256 -binary | openssl enc -base64)
        echo "NEW_HASH=sha256/$HASH" >> $GITHUB_ENV
        
    - name: Check if hash changed
      id: hash-check
      run: |
        if [ "$NEW_HASH" != "${{ secrets.CURRENT_HASH }}" ]; then
          echo "Hash changed from ${{ secrets.CURRENT_HASH }} to $NEW_HASH"
          echo "changed=true" >> $GITHUB_OUTPUT
        fi
        
    - name: Create PR with new hash
      if: steps.hash-check.outputs.changed == 'true'
      uses: peter-evans/create-pull-request@v4
      with:
        token: ${{ secrets.GITHUB_TOKEN }}
        commit-message: "Update SSL pin hash"
        title: "Update SSL pin hash"
        body: |
          SSL pin hash has changed:
          - Old: ${{ secrets.CURRENT_HASH }}
          - New: ${{ env.NEW_HASH }}
          
          Please review and update client applications.
```

## Validation and Testing

### Test Your Generated Hash

Verify your generated hash works correctly:

```bash
# Test hash generation
DOMAIN="your-apigee-domain.com"
EXPECTED_HASH="your-expected-hash-here"

ACTUAL_HASH=$(echo | openssl s_client -servername $DOMAIN -connect $DOMAIN:443 2>/dev/null | openssl x509 -pubkey -noout | openssl pkey -pubin -outform DER | openssl dgst -sha256 -binary | openssl enc -base64)

if [ "sha256/$ACTUAL_HASH" = "$EXPECTED_HASH" ]; then
    echo "✅ Hash validation successful"
else
    echo "❌ Hash validation failed"
    echo "Expected: $EXPECTED_HASH"
    echo "Actual: sha256/$ACTUAL_HASH"
fi
```

### Certificate Expiry Check

Monitor certificate expiry:

```bash
# Check certificate expiry
echo | openssl s_client -servername your-apigee-domain.com -connect your-apigee-domain.com:443 2>/dev/null | openssl x509 -noout -dates
```

## Best Practices

1. **Store Multiple Hashes**: Always maintain current and backup key hashes
2. **Automate Generation**: Use CI/CD to automate hash extraction and validation
3. **Monitor Certificate Changes**: Set up monitoring for certificate renewals
4. **Document Pin Rotation**: Maintain clear documentation of pin rotation procedures
5. **Test Regularly**: Regularly test pin validation in development environments
6. **Secure Storage**: Store pin hashes securely in your configuration management system

## Troubleshooting

### Common Issues

1. **Certificate Chain Issues**: Ensure you're extracting from the correct certificate in the chain
2. **Wrong Hash Format**: Ensure you're using SPKI hash, not certificate hash
3. **Network Issues**: Check firewall and proxy settings
4. **Certificate Validation**: Verify the certificate is valid and trusted

### Debug Commands

```bash
# Check certificate chain
openssl s_client -servername your-apigee-domain.com -connect your-apigee-domain.com:443 -showcerts

# Verify certificate
openssl x509 -in apigee-cert.pem -text -noout

# Compare hashes
echo "Certificate hash:"
openssl x509 -in apigee-cert.pem -fingerprint -sha256 -noout

echo "Public key hash:"
openssl x509 -in apigee-cert.pem -pubkey -noout | openssl pkey -pubin -outform DER | openssl dgst -sha256 -binary | openssl enc -base64
```

## Security Considerations

1. **Protect Private Keys**: Never expose private keys in client applications
2. **Secure Hash Storage**: Store pin hashes securely in your applications
3. **Regular Rotation**: Plan for regular key rotation
4. **Emergency Response**: Have procedures for emergency pin updates
5. **Monitoring**: Monitor for unusual pinning failures that might indicate attacks

## Output Format

The final output should be in this format for use in client applications:

```
Domain: your-apigee-domain.com
Primary Hash: sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
Backup Hash: sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=
Generated: 2025-09-14 10:30:00 UTC
Expires: 2026-09-14 10:30:00 UTC
```

This format ensures consistency across all client implementations and provides clear documentation of when hashes were generated and when they expire.
