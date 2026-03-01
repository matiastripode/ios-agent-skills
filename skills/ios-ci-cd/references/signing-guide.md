# Code Signing Guide

## Certificate Types

| Certificate | Purpose | Expires |
|-------------|---------|---------|
| Apple Development | Run on devices during development | 1 year |
| Apple Distribution | App Store and TestFlight submission | 1 year |

Each team has a limit of 2 distribution certificates. Share them via Fastlane match or export manually.

## Provisioning Profile Types

| Profile | Use Case | Devices |
|---------|----------|---------|
| Development | Run on registered devices via Xcode | Specific UDIDs |
| Ad Hoc | Distribute to registered devices (outside App Store) | Specific UDIDs (max 100) |
| App Store | Submit to App Store / TestFlight | Any device (via App Store) |
| Enterprise | In-house distribution (Enterprise account only) | Any device |

## Automatic vs Manual Signing

**Automatic (recommended for solo developers):**
- Xcode manages certificates and profiles
- Set in project: Signing & Capabilities > Automatically manage signing
- Works well for development, less reliable in CI

**Manual (recommended for teams and CI):**
- You control exactly which certificate and profile is used
- Use Fastlane match to sync across team and CI
- More predictable and reproducible

## Fastlane Match Workflow

Match stores certificates and profiles in a git repo or cloud storage, encrypted.

```bash
# First-time setup
fastlane match init
# Choose storage: git, google_cloud, or s3

# Generate new certificates and profiles
fastlane match development
fastlane match appstore

# On CI (never create, only download)
fastlane match appstore --readonly

# After adding a new device
fastlane match development --force_for_new_devices
```

**In Xcode project settings (manual signing):**
- Uncheck "Automatically manage signing"
- Select the provisioning profile that match created

## Keychain Management for CI

On CI, certificates need to be in a keychain that xcodebuild can access.

```bash
# Create a temporary keychain
security create-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
security default-keychain -s build.keychain
security unlock-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
security set-keychain-settings -t 3600 -u build.keychain

# Import certificate
security import certificate.p12 \
    -k build.keychain \
    -P "$CERT_PASSWORD" \
    -T /usr/bin/codesign

# Allow codesign to access keychain without prompt
security set-key-partition-list -S apple-tool:,apple:,codesign: \
    -s -k "$KEYCHAIN_PASSWORD" build.keychain
```

Fastlane match handles all of this automatically.

## Entitlements

Configured in `.entitlements` file and must match provisioning profile capabilities.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- Push Notifications -->
    <key>aps-environment</key>
    <string>production</string>

    <!-- App Groups (shared data between app and extensions) -->
    <key>com.apple.security.application-groups</key>
    <array>
        <string>group.com.example.myapp</string>
    </array>

    <!-- Associated Domains (Universal Links) -->
    <key>com.apple.developer.associated-domains</key>
    <array>
        <string>applinks:example.com</string>
    </array>

    <!-- Keychain Access Groups -->
    <key>keychain-access-groups</key>
    <array>
        <string>$(AppIdentifierPrefix)com.example.myapp</string>
    </array>
</dict>
</plist>
```

**Each entitlement must be enabled in:**
1. Apple Developer Portal (App ID capabilities)
2. Provisioning profile (regenerate after adding)
3. Xcode project (Signing & Capabilities tab)
4. `.entitlements` file

## Common Signing Errors

| Error | Fix |
|-------|-----|
| `No signing certificate "iOS Distribution" found` | Install certificate or run `fastlane match appstore` |
| `Provisioning profile doesn't include signing certificate` | Regenerate profile or use matching certificate |
| `No provisioning profiles with a valid signing identity` | Download profiles in Xcode or run match |
| `The executable was signed with invalid entitlements` | Entitlements in code don't match profile capabilities |
| `ITMS-90046: Invalid Code Signing Entitlements` | Remove entitlements not in provisioning profile |
| `Code Signing Error: No profiles for 'com.example.app'` | Bundle ID mismatch between project and profile |

## Handling Expired Certificates

```bash
# Check certificate expiry
security find-certificate -a -c "Apple Distribution" -p ~/Library/Keychains/login.keychain-db \
    | openssl x509 -noout -enddate

# With Fastlane match: nuke and recreate
fastlane match nuke distribution  # removes old
fastlane match appstore           # creates new

# Then regenerate provisioning profiles
# (match does this automatically)
```

After renewing certificates, all team members need to run `fastlane match appstore --readonly` to get the new certificate.
