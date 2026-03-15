# Privacy Manifest (iOS 17+)

## Overview

Starting Spring 2024, Apple requires apps and third-party SDKs to include a `PrivacyInfo.xcprivacy` file declaring what data is collected, why certain APIs are used, and whether data is used for tracking.

## File Location

Add `PrivacyInfo.xcprivacy` to your app target's root in Xcode. For frameworks/SDKs, include it in the framework bundle.

## File Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>NSPrivacyTracking</key>
    <false/>
    <key>NSPrivacyTrackingDomains</key>
    <array/>
    <key>NSPrivacyCollectedDataTypes</key>
    <array/>
    <key>NSPrivacyAccessedAPITypes</key>
    <array/>
</dict>
</plist>
```

## Required Reason APIs

If your code uses any of these API categories, you must declare the reason.

### File Timestamp APIs
APIs: `NSFileCreationDate`, `NSFileModificationDate`, `NSURLContentModificationDateKey`

```xml
<dict>
    <key>NSPrivacyAccessedAPIType</key>
    <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
    <key>NSPrivacyAccessedAPITypeReasons</key>
    <array>
        <string>C617.1</string> <!-- access timestamps of files within app container -->
    </array>
</dict>
```

Common reasons:
- `C617.1` - Files in app container
- `DDA9.1` - Display to user
- `3B52.1` - Files provided by user via file picker

### System Boot Time APIs
APIs: `systemUptime`, `mach_absolute_time()`, `ProcessInfo.processInfo.systemUptime`

```xml
<dict>
    <key>NSPrivacyAccessedAPIType</key>
    <string>NSPrivacyAccessedAPICategorySystemBootTime</string>
    <key>NSPrivacyAccessedAPITypeReasons</key>
    <array>
        <string>35F9.1</string> <!-- measure elapsed time -->
    </array>
</dict>
```

### Disk Space APIs
APIs: `volumeAvailableCapacityKey`, `volumeAvailableCapacityForImportantUsageKey`

```xml
<dict>
    <key>NSPrivacyAccessedAPIType</key>
    <string>NSPrivacyAccessedAPICategoryDiskSpace</string>
    <key>NSPrivacyAccessedAPITypeReasons</key>
    <array>
        <string>E174.1</string> <!-- check if enough space to write -->
    </array>
</dict>
```

### User Defaults APIs
APIs: `UserDefaults`

```xml
<dict>
    <key>NSPrivacyAccessedAPIType</key>
    <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
    <key>NSPrivacyAccessedAPITypeReasons</key>
    <array>
        <string>CA92.1</string> <!-- access user defaults within the app -->
    </array>
</dict>
```

Common reasons:
- `CA92.1` - Read/write within the app
- `1C8F.1` - Read/write via App Groups

### Active Keyboard APIs
APIs: `activeInputModes`

Reason: `3EC4.1` - Customize UI layout based on active keyboard.

## Tracking Declaration

If the app tracks users across apps/websites (for advertising):

```xml
<key>NSPrivacyTracking</key>
<true/>
<key>NSPrivacyTrackingDomains</key>
<array>
    <string>analytics.example.com</string>
    <string>ads.example.com</string>
</array>
```

Also requires `NSUserTrackingUsageDescription` in Info.plist and ATTrackingManager prompt.

## Collected Data Types

Declare all data types your app collects:

```xml
<key>NSPrivacyCollectedDataTypes</key>
<array>
    <dict>
        <key>NSPrivacyCollectedDataType</key>
        <string>NSPrivacyCollectedDataTypeEmailAddress</string>
        <key>NSPrivacyCollectedDataTypeLinked</key>
        <true/> <!-- linked to user identity -->
        <key>NSPrivacyCollectedDataTypeTracking</key>
        <false/> <!-- not used for tracking -->
        <key>NSPrivacyCollectedDataTypePurposes</key>
        <array>
            <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
        </array>
    </dict>
</array>
```

Common data types: `EmailAddress`, `Name`, `PhoneNumber`, `Location`, `DeviceID`, `ProductInteraction`, `CrashData`, `PerformanceData`.

## Third-Party SDK Privacy Manifests

Starting Spring 2024, commonly used SDKs must include their own privacy manifests. If a dependency lacks one, you may need to:
1. Update to a version that includes it
2. File an issue with the SDK maintainer
3. Include a manifest on their behalf (not recommended)

Check Xcode's privacy report: Product > Generate Privacy Report.

## App Store Connect Validation Errors

Common errors:
- `ITMS-91053` - Missing privacy manifest
- `ITMS-91056` - Missing required reason for API usage
- Missing `NSPrivacyAccessedAPITypeReasons` for a declared API type

**Fix:** Generate the privacy report in Xcode, review all flagged APIs, and add the appropriate reasons.

## Detection Patterns

Flag during review:
1. Use of required reason APIs without corresponding `PrivacyInfo.xcprivacy` entries
2. Missing `PrivacyInfo.xcprivacy` file entirely
3. Tracking-related SDKs without `NSPrivacyTracking` declaration
4. Data collection without matching `NSPrivacyCollectedDataTypes` entries
5. Third-party SDKs without their own privacy manifests
