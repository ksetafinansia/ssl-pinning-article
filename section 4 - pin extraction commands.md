# Section 4: Pin Extraction Commands - Step-by-Step Guide

## Overview
This comprehensive guide walks you through the process of extracting SPKI (Subject Public Key Info) hashes from certificates for SSL pinning implementation. Follow these steps to securely generate and validate certificate pins.

## What This Section Generates

**🔑 Primary Output: Base64-Encoded SPKI Hashes**

The commands in this section generate Base64-encoded SHA-256 hashes of public keys (SPKI), which look like this:
```
YLh1dHA681GQhQHU04xGhB/F8R2HZXZQh1m52HhQc=
```

**📋 Complete Pin Set Example:**
After following this guide, you'll have pins like:
```bash
# Primary pin (from live server)
PRIMARY_PIN="YLh1dHA681GQhQHU04xGhB/F8R2HZXZQh1m52HhQc="

# Backup pin (from backup keypair)  
BACKUP_PIN="ZMi2eEA123ABcDefGHI04xGhB/F8R2HZXZQh1m98W="
```

## How These Pins Are Used in Section 6

**🛠️ Implementation Context:**
The pins generated here become **hardcoded string constants** in your application code (Section 6).

**Example from Section 6 - iOS Implementation:**
```swift
class CertificatePinner: NSObject {
    private let pinnedHashes: Set<String> = [
        "YLh1dHA681GQhQHU04xGhB/F8R2HZXZQh1m52HhQc=", // ← FROM SECTION 4
        "ZMi2eEA123ABcDefGHI04xGhB/F8R2HZXZQh1m98W="  // ← FROM SECTION 4
    ]
}
```

**Example from Section 6 - Android Implementation:**
```kotlin
class CertificatePinner {
    companion object {
        private const val PRIMARY_PIN = "sha256/YLh1dHA681GQhQHU04xGhB/F8R2HZXZQh1m52HhQc=" // ← FROM SECTION 4
        private const val BACKUP_PIN = "sha256/ZMi2eEA123ABcDefGHI04xGhB/F8R2HZXZQh1m98W="   // ← FROM SECTION 4
    }
}
```

**🔄 Workflow Connection:**
```
Section 4 (This Section)     →     Section 6 (Implementation)
Extract SPKI hashes                Hardcode pins in app
from certificates                  Compare at runtime
                                  
OUTPUT: Base64 strings       →     INPUT: String constants
```

**⚠️ Important Notes:**
- **These are just strings**, but they represent cryptographic hashes
- **Format matters**: Some platforms need `sha256/` prefix, others don't
- **Security critical**: Wrong pins = broken app connectivity
- **Version control**: These pins go into your source code and app builds

## Prerequisites
Before you begin, ensure you have:
- OpenSSL installed on your system
- Access to the target host or certificate files
- Proper permissions to execute certificate extraction commands
- A secure environment for handling cryptographic materials

## Step-by-Step Pin Extraction Process

### Method 1: Extracting Pins from Live Host

**Step 1: Prepare Your Environment**
1. Open a terminal in a secure environment
2. Verify OpenSSL is installed by running: `openssl version`
3. Set the target hostname as a variable for easier reuse:
   ```bash
   HOST=api.kreditplus.com
   ```

**Step 2: Connect and Extract Certificate**
Execute the following command to connect to the live host and extract the certificate:
```bash
echo | openssl s_client -connect ${HOST}:443 -servername ${HOST} 2>/dev/null
```
This command:
- Establishes an SSL connection to the specified host
- Retrieves the server's certificate
- Suppresses error messages for cleaner output

**Step 3: Extract Public Key**
From the certificate output, extract the public key:
```bash
echo | openssl s_client -connect ${HOST}:443 -servername ${HOST} 2>/dev/null | openssl x509 -pubkey -noout
```
This command:
- Takes the certificate from Step 2
- Extracts only the public key portion
- Excludes other certificate information

**Step 4: Convert to DER Format**
Convert the public key to DER (Distinguished Encoding Rules) format:
```bash
echo | openssl s_client -connect ${HOST}:443 -servername ${HOST} 2>/dev/null | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der
```
This step:
- Converts the public key to binary DER format
- Prepares the key for hash generation

**Step 5: Generate SHA-256 Hash**
Create a SHA-256 hash of the DER-formatted public key:
```bash
echo | openssl s_client -connect ${HOST}:443 -servername ${HOST} 2>/dev/null | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary
```
This step:
- Generates a SHA-256 digest of the public key
- Outputs the hash in binary format

**Step 6: Encode to Base64**
Convert the binary hash to Base64 format for use in applications:
```bash
echo | openssl s_client -connect ${HOST}:443 -servername ${HOST} 2>/dev/null | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | openssl base64
```
This final step:
- Encodes the binary hash to Base64
- Produces the final SPKI pin ready for implementation

**Complete Command:**
```bash
HOST=api.kreditplus.com
echo | openssl s_client -connect ${HOST}:443 -servername ${HOST} 2>/dev/null | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | openssl base64
```

### Method 2: Extracting Pins from Backup Key Files

**Step 1: Locate Your Key File**
1. Ensure you have access to the private key file (e.g., `backup_key.pem`)
2. Verify the file is in the correct format and accessible
3. Set appropriate file permissions for security

**Step 2: Extract Public Key from Private Key**
```bash
openssl rsa -in backup_key.pem -pubout -outform der
```
This command:
- Reads the private key file
- Extracts the corresponding public key
- Outputs in DER format

**Step 3: Generate Hash and Encode**
```bash
openssl rsa -in backup_key.pem -pubout -outform der | openssl dgst -sha256 -binary | openssl base64
```
This command:
- Takes the public key from Step 2
- Generates SHA-256 hash
- Encodes to Base64 format

## Pin Validation and Verification

### Step 1: Cross-Validation Process
1. **Extract from Multiple Sources**: Generate pins from both live host and backup keys
2. **Compare Results**: Ensure pins match between different extraction methods
3. **Document Findings**: Record all extracted pins with timestamps and sources

### Step 2: Staging Environment Testing
1. **Deploy Test Pins**: Implement extracted pins in staging environment
2. **Verify Connectivity**: Test application functionality with pinning enabled
3. **Monitor Logs**: Check for any SSL/TLS connection errors
4. **Validate Pin Matching**: Confirm pins work correctly before production deployment

### Step 3: Registry Documentation
1. **Create Pin Entry**: Add new pins to your SPKI registry
2. **Include Metadata**: Document extraction date, source, and expiration
3. **Set Reminders**: Schedule pin rotation before certificate expiration
4. **Backup Pins**: Store pins securely with appropriate access controls

## Security Best Practices

### During Pin Extraction
1. **Secure Environment**: Perform extractions in isolated, secure systems
2. **Access Control**: Limit who can execute pin extraction procedures
3. **Audit Trail**: Log all pin generation activities with timestamps
4. **Clean Workspace**: Remove temporary files after extraction

### Key Management
1. **Secure Storage**: Store backup keys in HSM or encrypted storage
2. **Access Logging**: Monitor access to cryptographic materials
3. **Chain of Custody**: Maintain documentation of key handling
4. **Regular Rotation**: Follow established key rotation schedules

## Automation Considerations

For operational efficiency, consider implementing:

### Automated Pin Generation
- Integrate pin extraction into CI/CD pipelines
- Schedule regular pin validation checks
- Automate registry updates with new pins
- Set up alerts for upcoming pin expirations

### Monitoring and Alerting
- Monitor certificate expiration dates
- Alert on pin validation failures
- Track pin usage across applications
- Generate reports on pinning status

## Troubleshooting Common Issues

### Connection Problems
- Verify network connectivity to target host
- Check firewall rules and proxy settings
- Validate hostname and port configuration

### Command Execution Errors
- Ensure OpenSSL is properly installed and updated
- Check file permissions for key files
- Verify command syntax and parameter order

### Pin Validation Failures
- Compare pins extracted from different sources
- Check for certificate changes or rotations
- Validate pin format and encoding
