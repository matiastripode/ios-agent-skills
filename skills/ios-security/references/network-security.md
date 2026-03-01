# Network Security

## App Transport Security (ATS)

ATS enforces HTTPS by default. Exceptions must be justified.

**Acceptable exceptions:**
- Third-party domains you don't control that only support HTTP
- Local development servers
- Media streaming with specific requirements

**Never acceptable:**
```xml
<!-- NEVER do this - will likely be rejected by App Review -->
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

**Correct domain-specific exception:**
```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSExceptionDomains</key>
    <dict>
        <key>legacy-api.example.com</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
            <key>NSExceptionMinimumTLSVersion</key>
            <string>TLSv1.2</string>
        </dict>
    </dict>
</dict>
```

## Certificate Pinning

Pin certificates for authentication and payment endpoints to prevent MITM attacks.

```swift
class PinnedSessionDelegate: NSObject, URLSessionDelegate {
    // SHA-256 hash of the server's public key
    private let pinnedKeyHashes: Set<String> = [
        "BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=" // base64 encoded
    ]

    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        // Evaluate the server trust
        let policy = SecPolicyCreateSSL(true, challenge.protectionSpace.host as CFString)
        SecTrustSetPolicies(serverTrust, policy)

        var error: CFError?
        guard SecTrustEvaluateWithError(serverTrust, &error) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        // Extract and verify the public key hash
        guard let serverCertificate = SecTrustCopyCertificateChain(serverTrust) as? [SecCertificate],
              let firstCert = serverCertificate.first,
              let publicKey = SecCertificateCopyKey(firstCert),
              let publicKeyData = SecKeyCopyExternalRepresentation(publicKey, nil) as Data? else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        let hash = SHA256.hash(data: publicKeyData)
        let hashString = Data(hash).base64EncodedString()

        if pinnedKeyHashes.contains(hashString) {
            completionHandler(.useCredential, URLCredential(trust: serverTrust))
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}
```

**Pin public keys, not certificates** - certificates rotate; public keys are more stable.

## API Key Management

**CRITICAL - Never hardcode API keys:**
```swift
// BAD: visible in binary with `strings` command
let apiKey = "sk_live_abc123"

// BAD: obfuscation is not security
let apiKey = String(data: Data(base64Encoded: "c2tfbGl2ZV9hYmMxMjM=")!, encoding: .utf8)!
```

**Better approaches:**
1. **Server-side proxy:** App calls your server, server calls third-party API with the key
2. **Configuration files excluded from source control:** Load from a `.xcconfig` not in git
3. **CloudKit or remote config:** Fetch keys at runtime from a secure source
4. **Keychain storage after initial secure delivery:** Store on first launch via secure channel

```swift
// Load from xcconfig (not committed to git)
let apiKey = Bundle.main.infoDictionary?["API_KEY"] as? String ?? ""
```

## Production Logging

**Remove or guard sensitive logging before release:**
```swift
// BAD: leaks data to Console.app and device logs
print("Auth response: \(response)")
os_log("Token: %{public}@", token)

// GOOD: use private formatting or conditional compilation
os_log("Token: %{private}@", token) // redacted in logs
#if DEBUG
print("Auth response: \(response)")
#endif
```

## URLSession Security Configuration

```swift
let configuration = URLSessionConfiguration.default
configuration.tlsMinimumSupportedProtocolVersion = .TLSv12
configuration.urlCache = nil // don't cache sensitive responses
configuration.requestCachePolicy = .reloadIgnoringLocalCacheData

// For sensitive endpoints, disable caching entirely
configuration.urlCache = URLCache(memoryCapacity: 0, diskCapacity: 0)
```

## Detection Patterns

During review, flag:
1. Any `NSAllowsArbitraryLoads = YES` without domain-specific exceptions
2. Hardcoded strings that look like API keys, tokens, or secrets
3. `print()` or `NSLog()` with sensitive data parameters
4. Missing certificate pinning on authentication endpoints
5. HTTP URLs in source code (should be HTTPS)
6. Disabled TLS validation in URLSession delegates
