# Data Protection

## iOS Data Protection Levels

Set via file attributes or entitlements. Controls when files are decryptable.

| Level | When Accessible | Use Case |
|-------|----------------|----------|
| `NSFileProtectionComplete` | Only when unlocked | Sensitive user data |
| `NSFileProtectionCompleteUnlessOpen` | Can finish writing after lock | Downloads, uploads |
| `NSFileProtectionCompleteUntilFirstUserAuthentication` | After first unlock | Default for most apps |
| `NSFileProtectionNone` | Always | Non-sensitive cache (avoid for user data) |

```swift
// Set protection on a file
try data.write(to: fileURL, options: .completeFileProtection)

// Set protection on a directory
try FileManager.default.setAttributes(
    [.protectionKey: FileProtectionType.complete],
    ofItemAtPath: directoryPath
)

// Check current protection
let attributes = try FileManager.default.attributesOfItem(atPath: path)
let protection = attributes[.protectionKey] as? FileProtectionType
```

**Enable Data Protection entitlement** in Xcode: Signing & Capabilities > Data Protection.

## Preventing Sensitive Data in Backups

Files in the Documents directory are backed up to iCloud by default.

```swift
var fileURL = documentsDirectory.appendingPathComponent("sensitive.db")
var resourceValues = URLResourceValues()
resourceValues.isExcludedFromBackup = true
try fileURL.setResourceValues(resourceValues)
```

**Exclude from backup:**
- Cached authentication tokens
- Downloaded content that can be re-fetched
- Temporary processing files

**Keep in backups:**
- User-created content (documents, photos)
- App settings and preferences

## Background Snapshot Protection

When the app enters background, iOS takes a screenshot for the app switcher. Sensitive content is visible.

```swift
// UIKit
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    private var blurView: UIVisualEffectView?

    func sceneDidEnterBackground(_ scene: UIScene) {
        let blur = UIVisualEffectView(effect: UIBlurEffect(style: .regular))
        blur.frame = window?.bounds ?? .zero
        blur.tag = 999
        window?.addSubview(blur)
        blurView = blur
    }

    func sceneWillEnterForeground(_ scene: UIScene) {
        window?.viewWithTag(999)?.removeFromSuperview()
        blurView = nil
    }
}

// SwiftUI
@Environment(\.scenePhase) var scenePhase

var body: some View {
    content
        .overlay {
            if scenePhase != .active {
                Rectangle()
                    .fill(.ultraThinMaterial)
                    .ignoresSafeArea()
            }
        }
}
```

## Clipboard Security

The clipboard is shared across apps. Sensitive data should not persist there.

```swift
// Set expiration on clipboard items (iOS 16+)
UIPasteboard.general.setItems(
    [[UIPasteboard.typeAutomatic: sensitiveText]],
    options: [.expirationDate: Date().addingTimeInterval(60)] // expires in 60 seconds
)

// For password fields, use .textContentType(.password) which handles clipboard securely
textField.textContentType = .password

// Clear clipboard when app enters background if it contains sensitive data
func sceneDidEnterBackground(_ scene: UIScene) {
    if UIPasteboard.general.hasStrings {
        UIPasteboard.general.items = []
    }
}
```

## Core Data Encryption

Core Data itself does not encrypt. Use iOS Data Protection or encrypt at the field level.

```swift
// Option 1: Rely on file-level Data Protection
let description = NSPersistentStoreDescription()
description.setOption(
    FileProtectionType.complete as NSObject,
    forKey: NSPersistentStoreFileProtectionKey
)

// Option 2: Encrypt sensitive fields before storage
extension NSManagedObject {
    func encryptedValue(_ value: String, key: SymmetricKey) throws -> Data {
        let data = Data(value.utf8)
        let sealedBox = try AES.GCM.seal(data, using: key)
        return sealedBox.combined!
    }

    func decryptedValue(_ data: Data, key: SymmetricKey) throws -> String {
        let sealedBox = try AES.GCM.SealedBox(combined: data)
        let decrypted = try AES.GCM.open(sealedBox, using: key)
        return String(data: decrypted, encoding: .utf8)!
    }
}
```

## Secure Deletion

iOS uses encryption-based "deletion" (destroying the file key), but for extra assurance:

```swift
// Overwrite before deleting for sensitive temporary files
func secureDelete(at url: URL) throws {
    let fileSize = try FileManager.default.attributesOfItem(atPath: url.path)[.size] as! UInt64
    let handle = try FileHandle(forWritingTo: url)
    let zeros = Data(count: Int(fileSize))
    handle.write(zeros)
    handle.closeFile()
    try FileManager.default.removeItem(at: url)
}
```

## Detection Patterns

Flag during review:
1. Sensitive data written to Documents/ without `isExcludedFromBackup`
2. No background blur/overlay for apps showing sensitive content
3. `NSFileProtectionNone` on user data files
4. Sensitive strings copied to `UIPasteboard` without expiration
5. Core Data stores without file protection set
6. Temporary files with sensitive content not cleaned up
