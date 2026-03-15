# OWASP Mobile Top 10 - iOS Perspective

## M1: Improper Credential Usage

**Risk:** Hardcoded credentials, API keys in source code, tokens stored insecurely.

**Detection patterns:**
```swift
// FLAG: Hardcoded credentials
let apiKey = "AIzaSyB..." // visible with `strings` on binary
let password = "admin123"
let secret = "sk_live_..."

// FLAG: Credentials in UserDefaults
UserDefaults.standard.set(token, forKey: "authToken")

// FLAG: Credentials in plist files
Bundle.main.object(forInfoDictionaryKey: "ApiSecret")
```

**Fix:** Store in Keychain with appropriate access control. Use server-side proxy for third-party API keys. See `keychain-usage.md`.

## M2: Inadequate Supply Chain Security

**Risk:** Compromised dependencies, unsigned SDKs, outdated libraries with known vulnerabilities.

**Detection patterns:**
- Dependencies not pinned to specific versions in `Package.swift` or `Podfile`
- Binary SDKs without signature verification
- Dependencies from untrusted sources

**Fix:**
```swift
// Pin exact versions in Package.swift
.package(url: "https://github.com/example/lib.git", exact: "2.1.0")

// Or pin to a specific revision
.package(url: "https://github.com/example/lib.git", revision: "abc123")
```

Run `swift package audit` (if available) or review dependencies manually for known CVEs.

## M3: Insecure Authentication/Authorization

**Risk:** Missing biometric re-authentication for sensitive actions, weak session management.

**Detection patterns:**
```swift
// FLAG: No re-authentication for sensitive operations
func deleteAccount() {
    api.deleteAccount() // should require biometric/password confirmation
}

// FLAG: Tokens that never expire
struct AuthToken {
    let value: String
    // no expiry field
}
```

**Fix:**
```swift
func deleteAccount() async throws {
    let context = LAContext()
    try await context.evaluatePolicy(
        .deviceOwnerAuthentication,
        localizedReason: "Confirm account deletion"
    )
    try await api.deleteAccount()
}
```

## M4: Insufficient Input/Output Validation

**Risk:** Injection attacks through deep links, web views, or untrusted data sources.

**Detection patterns:**
```swift
// FLAG: Unvalidated deep link parameters
func handle(url: URL) {
    let path = url.path
    webView.loadFileURL(URL(fileURLWithPath: path), allowingReadAccessTo: rootURL)
    // path traversal possible
}

// FLAG: JavaScript injection via WKWebView
webView.evaluateJavaScript("setUser('\(userName)')") // injection if userName contains '
```

**Fix:**
```swift
// Validate and sanitize deep link input
func handle(url: URL) {
    guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
          let action = components.queryItems?.first(where: { $0.name == "action" })?.value,
          ["view", "share", "edit"].contains(action) else {
        return // reject unknown actions
    }
}

// Use parameterized JavaScript
webView.callAsyncJavaScript(
    "setUser(name)",
    arguments: ["name": userName],
    contentWorld: .page
)
```

## M5: Insecure Communication

**Risk:** Data transmitted over HTTP, disabled certificate validation, missing certificate pinning.

**Detection patterns:**
```swift
// FLAG: Disabled ATS
// NSAllowsArbitraryLoads = YES in Info.plist

// FLAG: Accepting all certificates
func urlSession(_ session: URLSession, didReceive challenge: URLAuthenticationChallenge,
                completionHandler: (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) {
    completionHandler(.useCredential, URLCredential(trust: challenge.protectionSpace.serverTrust!))
    // accepts ANY certificate including forged ones
}
```

**Fix:** See `network-security.md` for ATS configuration and certificate pinning implementation.

## M6: Inadequate Privacy Controls

**Risk:** Collecting data without disclosure, missing privacy manifest, no opt-out mechanism.

**Detection patterns:**
- Missing `PrivacyInfo.xcprivacy`
- Analytics SDKs initialized before user consent
- Location tracking without clear purpose string
- IDFA access without ATTrackingManager prompt

**Fix:** See `privacy-manifest.md`. Always initialize analytics after consent. Provide clear purpose strings.

## M7: Insufficient Binary Protections

**Risk:** Lack of jailbreak detection, no code obfuscation for sensitive logic, debug symbols in release.

**Detection patterns:**
```swift
// FLAG: Debug code in production
#if DEBUG
// ok
#else
print("Sensitive debug info") // should not be here
#endif
```

**Mitigations:**
```swift
// Runtime jailbreak detection (defense in depth, not foolproof)
func isJailbroken() -> Bool {
    #if targetEnvironment(simulator)
    return false
    #else
    let paths = ["/Applications/Cydia.app", "/usr/sbin/sshd", "/private/var/lib/apt/"]
    return paths.contains { FileManager.default.fileExists(atPath: $0) }
    #endif
}
```

Ensure `STRIP_SWIFT_SYMBOLS = YES` and `DEBUG_INFORMATION_FORMAT = dwarf` (not `dwarf-with-dsym` in the binary itself) for release builds.

## M8: Security Misconfiguration

**Risk:** Default configurations left in production, overly permissive ATS, debug features enabled.

**Detection patterns:**
- `NSAllowsArbitraryLoads` enabled
- Debug menus accessible in release builds
- `NSAppTransportSecurity` exceptions without justification
- Backup of sensitive files enabled (default)

**Fix:** Audit `Info.plist` and build settings for each release. Use different configurations for Debug/Release.

## M9: Insecure Data Storage

**Risk:** Sensitive data in plain text on the file system, caches, or logs.

**Detection patterns:**
```swift
// FLAG: Plaintext storage of sensitive data
try sensitiveData.write(to: fileURL) // no encryption
FileManager.default.createFile(atPath: path, contents: tokenData) // no protection

// FLAG: Sensitive data in NSURLCache
// Default URLSession caches responses including auth headers

// FLAG: SQLite without encryption
let db = try Connection(dbPath) // plain SQLite
```

**Fix:** See `data-protection.md` for file protection levels and encryption patterns. Disable caching for sensitive endpoints.

## M10: Insufficient Cryptography

**Risk:** Custom crypto implementations, weak algorithms, hardcoded keys/IVs.

**Detection patterns:**
```swift
// FLAG: Using deprecated/weak algorithms
CCCrypt(CCOperation(kCCEncrypt), CCAlgorithm(kCCAlgorithmDES), ...) // DES is weak
let digest = Insecure.MD5.hash(data: data) // MD5 is broken for security

// FLAG: Hardcoded encryption key
let key = "mysecretkey12345".data(using: .utf8)!

// FLAG: Static IV
let iv = Data(repeating: 0, count: 16) // predictable IV
```

**Fix:**
```swift
// Use Apple CryptoKit with proper key management
import CryptoKit

// Generate a random key
let key = SymmetricKey(size: .bits256)

// Encrypt with AES-GCM (authenticated encryption)
let sealed = try AES.GCM.seal(plaintext, using: key)

// Decrypt
let decrypted = try AES.GCM.open(sealed, using: key)

// Hash with SHA-256
let hash = SHA256.hash(data: data)
```

Always use `CryptoKit` or `Security.framework`. Never implement custom cryptographic algorithms.
