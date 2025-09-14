# SSL Pinning Implementation Guide

## Architecture Overview

SSL pinning architecture follows this pattern:
```
[Clients (mobile/cms)] -> [ApiGee] -> [Backend Services]
```

- **Client -> Apigee**: External connections (secured with SSL pinning)
- **Apigee -> Backend Services**: Internal connections (secured with mTLS and network controls)

## Goals

Secure the connection through SSL pinning to prevent man-in-the-middle attacks and ensure clients only communicate with legitimate servers.

## SSL Pinning Strategy

### Why Public Key Pinning?

We recommend **public key pinning** over certificate pinning because:
- Public keys remain the same even when certificates are reissued
- Reduces maintenance overhead
- Provides flexibility for certificate renewal

### Implementation Approach

1. **Extract Public Key Hash**: Generate SHA-256 hash of the server's public key
2. **Pin Multiple Keys**: Include current and backup key hashes
3. **Validate in Client**: Compare server's key against pinned hashes
4. **Graceful Degradation**: Include kill switch for emergency disabling

## Best Practices and Considerations

### Security Best Practices

1. **Use Public Key Pinning**: Pin the public key hash instead of the full certificate
   - Doesn't require client updates when certificates are renewed
   - Works as long as the key pair remains the same

2. **Pin Backup Keys**: Always pin multiple public key hashes
   - One for the current key
   - One for a backup key
   - Allows key rotation without immediate client updates

3. **Include Transition Plan**: Plan for key rotation
   - Release new app version with updated hashes
   - Ensure wide adoption before deprecating old keys
   - Maintain backward compatibility during transition

4. **Specific Hostname Pinning**: Never pin wildcard certificates
   - Pin for specific hostnames only
   - Reduces attack surface

5. **CI/CD Integration**: Automate the pinning process
   - Extract public key hash from certificates automatically
   - Inject hashes into client builds
   - Minimize human error

### Internal Connection Security

For **Apigee -> Backend Services** connections:
- Use mutual TLS (mTLS) authentication
- Implement network-level access controls
- Use IP allowlisting for trusted Apigee instances
- Keep client-facing pinning as independent security measure

### Error Handling

1. **Graceful Degradation**: Handle pinning failures appropriately
2. **Kill Switch**: Remote configuration to disable pinning in emergencies
3. **Logging**: Comprehensive logging for debugging and monitoring
4. **User Experience**: Provide clear error messages to users

### Testing Strategy

1. **Unit Tests**: Test pinning logic in isolation
2. **Integration Tests**: Test with real certificates
3. **Network Tests**: Test with various network conditions
4. **Failure Tests**: Test behavior when pinning fails

## Implementation Files

This guide includes platform-specific implementations:

1. **[Server Public Key Extraction](server-public-key-extraction.md)**: Step-by-step guide for generating public key hashes
2. **[iOS Swift Implementation](ssl-pinning-ios-swift.md)**: URLSession with custom TrustManager
3. **[Android Kotlin Implementation](ssl-pinning-android-kotlin.md)**: OkHttp with CertificatePinner
4. **[Flutter Dart Implementation](ssl-pinning-flutter-dart.md)**: Cross-platform implementation
5. **[Nuxt.js CMS Implementation](ssl-pinning-nuxtjs-cms.md)**: Server-side and client-side validation

## Key Features Across All Platforms

### Consistent Pin Format
- All implementations use the same Base64 SPKI hashes
- Standardized hash generation process
- Cross-platform compatibility

### Error Handling
- Dedicated SSL pinning failure handling
- Clear error messages and logging
- Fallback mechanisms

### Kill Switch
- Remote configuration capability
- Emergency pinning disabling
- Gradual rollout support

### Testing
- Comprehensive unit tests
- Integration testing
- Real-world scenario testing

### Security Considerations
- Best practices implementation
- Troubleshooting guides
- Performance optimization

### Real-world Usage
- Complete examples with UI integration
- Production-ready code
- Monitoring and analytics integration

## Security Monitoring

### What to Monitor

1. **Pinning Failures**: Track when pinning validation fails
2. **Certificate Changes**: Monitor for unexpected certificate changes
3. **Attack Attempts**: Log potential MITM attempts
4. **Performance Impact**: Monitor latency introduced by pinning

### Alerting

Set up alerts for:
- High pinning failure rates
- Unexpected certificate changes
- Unusual network patterns
- Performance degradation

## Troubleshooting

### Common Issues

1. **Clock Skew**: Ensure device time is accurate
2. **Certificate Renewal**: Update pinned hashes when certificates are renewed
3. **Network Proxies**: Handle corporate proxy environments
4. **Development Testing**: Use test certificates for development

### Debugging Steps

1. Verify certificate chain
2. Check pinned hash accuracy
3. Validate network connectivity
4. Review error logs
5. Test with different network conditions

## Compliance and Legal Considerations

1. **Data Privacy**: Ensure logging complies with privacy regulations
2. **Security Standards**: Meet industry security requirements
3. **Audit Trail**: Maintain security audit logs
4. **Documentation**: Keep implementation documentation current

## Conclusion

SSL pinning provides an additional layer of security for client-server communications. When implemented correctly with proper error handling, monitoring, and fallback mechanisms, it significantly enhances the security posture of your application.

Remember to:
- Plan for certificate rotation
- Test thoroughly across all platforms
- Monitor implementation in production
- Keep backup keys ready
- Document your implementation for future maintenance
