# Section 4: Pin Extraction Commands

## Overview
This section provides the technical commands and procedures for extracting SPKI (Subject Public Key Info) hashes from certificates for pinning implementation.

## Content

### From Live Host
Extract SPKI hash from currently deployed certificate:

```bash
HOST=api.kreditplus.com
echo | openssl s_client -connect ${HOST}:443 -servername ${HOST} 2>/dev/null  | openssl x509 -pubkey -noout  | openssl pkey -pubin -outform der  | openssl dgst -sha256 -binary  | openssl base64
```

**Command Breakdown:**
- `openssl s_client`: Establishes SSL connection to retrieve certificate
- `openssl x509 -pubkey`: Extracts public key from certificate
- `openssl pkey -pubin -outform der`: Converts to DER format
- `openssl dgst -sha256 -binary`: Generates SHA-256 hash
- `openssl base64`: Encodes in Base64 format

### From Backup Keypair
Extract SPKI hash from backup private key file:

```bash
openssl rsa -in backup_key.pem -pubout -outform der  | openssl dgst -sha256 -binary | openssl base64
```

## Validation Procedures

### Verify Pin Accuracy
1. Extract pins from multiple sources (live cert + backup key)
2. Cross-validate extracted hashes
3. Document pins in SPKI registry
4. Test pins in staging environment before production use

### Security Considerations
- Store backup keys securely (HSM or encrypted storage)
- Limit access to pin extraction procedures
- Audit all pin generation activities
- Maintain chain of custody for cryptographic materials

## Automation Scripts
Consider automating pin extraction for operational efficiency:
- CI/CD pipeline integration
- Scheduled pin validation
- Automated registry updates
- Alert on pin expiration
