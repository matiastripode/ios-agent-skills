# App Store Rejection Prevention

## Guideline 2.1: App Completeness

**Common triggers:**
- App crashes on launch or during core flows
- Placeholder content ("Lorem ipsum", "TODO", test images)
- Broken links or dead-end screens
- Features that require login but provide no way to create an account or test credentials

**Review checklist:**
- [ ] App launches without crashing on all supported devices
- [ ] All user-facing text is finalized (no placeholder/debug text)
- [ ] All navigation paths lead somewhere (no dead ends)
- [ ] Demo/test credentials provided in App Review notes if login required

## Guideline 2.3: Accurate Metadata

- App name, description, and screenshots must match actual functionality
- Screenshots must be from the actual app, not mockups
- No references to other platforms ("also available on Android")
- Category must match primary functionality

## Guideline 4.0: Design (Minimum Functionality)

Apps that are too simple (single web view, single screen with no interaction) get rejected. Ensure the app provides meaningful native functionality beyond what a website could offer.

## Guideline 5.1.1: Data Collection and Storage

**Privacy requirements:**
- [ ] Privacy manifest (`PrivacyInfo.xcprivacy`) included (iOS 17+)
- [ ] All data collection types declared in App Store Connect
- [ ] Privacy policy URL provided and accessible
- [ ] Purpose strings for ALL used permissions (see below)

## Required Purpose Strings

If your app uses any of these, the corresponding `Info.plist` key MUST have a human-readable description:

| Capability | Info.plist Key |
|------------|---------------|
| Camera | `NSCameraUsageDescription` |
| Photo Library | `NSPhotoLibraryUsageDescription` |
| Photo Library (write) | `NSPhotoLibraryAddUsageDescription` |
| Location (always) | `NSLocationAlwaysAndWhenInUseUsageDescription` |
| Location (in use) | `NSLocationWhenInUseUsageDescription` |
| Microphone | `NSMicrophoneUsageDescription` |
| Contacts | `NSContactsUsageDescription` |
| Calendar | `NSCalendarsUsageDescription` |
| Reminders | `NSRemindersUsageDescription` |
| Bluetooth | `NSBluetoothAlwaysUsageDescription` |
| Health | `NSHealthShareUsageDescription` / `NSHealthUpdateUsageDescription` |
| Motion | `NSMotionUsageDescription` |
| Face ID | `NSFaceIDUsageDescription` |
| Tracking | `NSUserTrackingUsageDescription` |
| Local Network | `NSLocalNetworkUsageDescription` |

**Bad purpose string:** "This app needs camera access."
**Good purpose string:** "Camera access is used to scan QR codes for quick check-in at events."

## Privacy Manifest (iOS 17+)

Apps and third-party SDKs must include `PrivacyInfo.xcprivacy` declaring:
- Required reason APIs used (see ios-security skill for details)
- Data types collected
- Whether data is used for tracking

Missing or incomplete privacy manifests cause rejection starting Spring 2024.

## Required Reason APIs

These APIs require a declared reason in the privacy manifest:
- File timestamp APIs (`NSFileCreationDate`, `NSFileModificationDate`)
- System boot time APIs (`systemUptime`, `mach_absolute_time`)
- Disk space APIs (`volumeAvailableCapacityKey`)
- User defaults APIs (`UserDefaults`)
- Active keyboard APIs

## Export Compliance (IDFA / Encryption)

- [ ] If the app uses IDFA (`ASIdentifierManager`), declare it and explain the use
- [ ] If the app uses encryption beyond standard HTTPS, file for an export compliance review or declare exemption
- [ ] Set `ITSAppUsesNonExemptEncryption` to `NO` in Info.plist if only using HTTPS

## IPv6 Compatibility

Apple's review network is IPv6-only. Apps must work on IPv6 networks.

**Checklist:**
- [ ] No hardcoded IPv4 addresses in code
- [ ] Use high-level networking APIs (URLSession, not raw sockets)
- [ ] Test with "NAT64/DNS64" network condition in macOS Internet Sharing

## App Transport Security (ATS)

- [ ] No blanket `NSAllowsArbitraryLoads = YES` (almost always rejected)
- [ ] Domain-specific exceptions have justification in review notes
- [ ] All first-party APIs use HTTPS

## Common Binary Rejections

These are automated checks that fail before human review:
- Missing required device capabilities in Info.plist
- Using private/undocumented APIs
- Referencing deprecated frameworks that have been removed
- Bitcode issues (no longer required, but misconfigurations still cause problems)
- Missing required icons (App Icon in asset catalog)
- App binary too large without on-demand resources or app thinning
