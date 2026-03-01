# Keychain Usage

## When to Use What

| Data Type | Storage | Why |
|-----------|---------|-----|
| Auth tokens, passwords | Keychain | Encrypted, survives app reinstall optionally |
| API keys (if must be on device) | Keychain | Not visible in binary or backups |
| User preferences (non-sensitive) | UserDefaults | Simple, fast, not secret |
| Large files | File system + Data Protection | Keychain has size limits |
| Cached data (non-sensitive) | File system / Core Data | Performance |

## Keychain Access Levels

Choose the right `kSecAttrAccessible` value:

| Value | When Accessible | Use Case |
|-------|----------------|----------|
| `kSecAttrAccessibleWhenUnlocked` | Device unlocked | Default for most tokens |
| `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` | Device unlocked, no backup migration | Sensitive tokens |
| `kSecAttrAccessibleAfterFirstUnlock` | After first unlock until restart | Background refresh tokens |
| `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` | Same, no backup migration | Background tokens, high security |
| `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` | Only when passcode is set | Highest security requirement |

**Never use:** `kSecAttrAccessibleAlways` (deprecated and insecure).

## Secure Keychain Wrapper

```swift
enum KeychainError: Error {
    case duplicateItem
    case itemNotFound
    case unexpectedStatus(OSStatus)
}

struct KeychainService {
    static func save(key: String, data: Data, accessibility: CFString = kSecAttrAccessibleWhenUnlockedThisDeviceOnly) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: accessibility
        ]

        let status = SecItemAdd(query as CFDictionary, nil)

        if status == errSecDuplicateItem {
            let updateQuery: [String: Any] = [
                kSecClass as String: kSecClassGenericPassword,
                kSecAttrAccount as String: key
            ]
            let attributes: [String: Any] = [kSecValueData as String: data]
            let updateStatus = SecItemUpdate(updateQuery as CFDictionary, attributes as CFDictionary)
            guard updateStatus == errSecSuccess else {
                throw KeychainError.unexpectedStatus(updateStatus)
            }
        } else if status != errSecSuccess {
            throw KeychainError.unexpectedStatus(status)
        }
    }

    static func load(key: String) throws -> Data {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess, let data = result as? Data else {
            if status == errSecItemNotFound { throw KeychainError.itemNotFound }
            throw KeychainError.unexpectedStatus(status)
        }
        return data
    }

    static func delete(key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key
        ]
        let status = SecItemDelete(query as CFDictionary)
        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unexpectedStatus(status)
        }
    }
}
```

## Biometric Authentication with Keychain

```swift
import LocalAuthentication

func saveWithBiometric(key: String, data: Data) throws {
    let access = SecAccessControlCreateWithFlags(
        nil,
        kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
        .biometryCurrentSet, // invalidate if biometrics change
        nil
    )

    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: key,
        kSecValueData as String: data,
        kSecAttrAccessControl as String: access as Any
    ]

    SecItemAdd(query as CFDictionary, nil)
}

func loadWithBiometric(key: String, reason: String) throws -> Data {
    let context = LAContext()
    context.localizedReason = reason

    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: key,
        kSecReturnData as String: true,
        kSecMatchLimit as String: kSecMatchLimitOne,
        kSecUseAuthenticationContext as String: context
    ]

    var result: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &result)
    guard status == errSecSuccess, let data = result as? Data else {
        throw KeychainError.unexpectedStatus(status)
    }
    return data
}
```

## Common Mistakes to Flag

```swift
// CRITICAL: Storing tokens in UserDefaults
UserDefaults.standard.set(authToken, forKey: "token") // INSECURE

// CRITICAL: Hardcoded secrets
let apiKey = "sk-1234567890abcdef" // visible in binary

// HIGH: Logging sensitive data
print("User token: \(token)") // visible in device logs

// MEDIUM: Using kSecAttrAccessibleAlways
kSecAttrAccessible: kSecAttrAccessibleAlways // deprecated, insecure
```
