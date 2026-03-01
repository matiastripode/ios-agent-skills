# TestFlight Checklist

## Pre-Submission Checks

### Version and Build Number
```bash
# Check current version
agvtool what-marketing-version
agvtool what-version

# Increment build number
agvtool next-version -all
# or set specific
agvtool new-version -all 42

# Increment marketing version
agvtool new-marketing-version 1.2.0

# With Fastlane
fastlane run increment_build_number
fastlane run increment_version_number bump_type:minor
```

**Rules:**
- Marketing version (`CFBundleShortVersionString`): semantic versioning (1.2.3)
- Build number (`CFBundleVersion`): must be unique per marketing version, always increasing
- TestFlight rejects builds with a build number already used for that version

### Code Quality
- [ ] All tests pass
- [ ] No compiler warnings in Release configuration
- [ ] No `TODO` or `FIXME` in production code paths
- [ ] Debug logging disabled or guarded with `#if DEBUG`
- [ ] Analytics pointing to production endpoints
- [ ] API base URLs pointing to production

### App Store Requirements
- [ ] App icon set in asset catalog (all required sizes)
- [ ] Launch screen configured (storyboard or SwiftUI)
- [ ] Privacy manifest included (`PrivacyInfo.xcprivacy`)
- [ ] All purpose strings present in Info.plist
- [ ] Export compliance declared (`ITSAppUsesNonExemptEncryption`)

## App Store Connect API

Automate uploads and management via API key (no password needed).

```bash
# Create API key at: App Store Connect > Users and Access > Integrations > App Store Connect API

# Use with Fastlane
# In Appfile or Fastfile:
```

```ruby
# API key authentication (recommended over password)
lane :beta do
  api_key = app_store_connect_api_key(
    key_id: "YOUR_KEY_ID",
    issuer_id: "YOUR_ISSUER_ID",
    key_filepath: "fastlane/AuthKey.p8"
  )

  pilot(api_key: api_key)
end
```

```bash
# Or via environment variables
export APP_STORE_CONNECT_API_KEY_KEY_ID=YOUR_KEY_ID
export APP_STORE_CONNECT_API_KEY_ISSUER_ID=YOUR_ISSUER_ID
export APP_STORE_CONNECT_API_KEY_KEY_FILEPATH=fastlane/AuthKey.p8
```

## Internal vs External Testing

| Aspect | Internal | External |
|--------|----------|----------|
| Testers | Team members (up to 100) | Up to 10,000 |
| Review required | No | Yes (first build, or significant changes) |
| Available after | Processing (minutes to hours) | Beta App Review (usually < 24h) |
| Groups | Automatic "App Store Connect Users" | Custom groups |

## Beta App Review

Required for external testers. Provide:
- **Beta App Description:** What the app does
- **What to Test:** Specific areas for feedback
- **Contact Information:** Email for review team
- **Sign-In Information:** Test account credentials if login required
- **Demo Account:** Same as sign-in if app requires authentication

## Export Compliance

Most apps only use HTTPS and can declare exemption.

```xml
<!-- In Info.plist -->
<key>ITSAppUsesNonExemptEncryption</key>
<false/> <!-- Set to YES only if using custom encryption beyond HTTPS -->
```

If set to `YES`, you need to provide export compliance documentation or select an exemption category in App Store Connect.

## IDFA (Advertising Identifier)

If your app or any SDK uses `ASIdentifierManager`:
- Declare IDFA usage in App Store Connect
- Implement ATTrackingManager prompt
- Declare in privacy manifest

If you don't use IDFA, ensure no third-party SDK accesses it without your knowledge. Check with:
```bash
# Search binary for IDFA usage
grep -r "ASIdentifierManager\|advertisingIdentifier" Pods/ Packages/
```

## Build Processing

After upload, Apple processes the build (typically 10-30 minutes):
- Automated checks run (signing, entitlements, binary analysis)
- Processing status visible in App Store Connect or via API

**Common processing failures:**
| Error | Fix |
|-------|-----|
| `ITMS-90034: Missing or invalid signature` | Re-sign with valid distribution certificate |
| `ITMS-90046: Invalid Code Signing Entitlements` | Match entitlements with provisioning profile |
| `ITMS-90096: Bundle identifier mismatch` | Bundle ID must match App Store Connect |
| `ITMS-90189: Redundant binary upload` | Increment build number |
| `ITMS-90717: Invalid Info.plist` | Check for required keys |
| `ITMS-91053: Missing privacy manifest` | Add PrivacyInfo.xcprivacy |

## Managing Test Groups

```ruby
# Fastlane: Add external testers
lane :add_testers do
  pilot(
    distribute_external: true,
    groups: ["Beta Testers"],
    changelog: "Bug fixes and performance improvements"
  )
end

# Add specific tester
lane :add_tester do
  pilot(
    distribute_external: true,
    email: "tester@example.com",
    first_name: "Test",
    last_name: "User"
  )
end
```

## Transitioning to App Store

When ready to submit the tested build:

1. In App Store Connect, select the build from TestFlight for the App Store version
2. Complete all App Store metadata (description, keywords, screenshots)
3. Set pricing and availability
4. Submit for App Review

```ruby
# With Fastlane
lane :release do
  deliver(
    build_number: latest_testflight_build_number.to_s,
    submit_for_review: true,
    automatic_release: false, # manually release after approval
    submission_information: {
      add_id_info_uses_idfa: false
    }
  )
end
```

## Crash Monitoring

- TestFlight crash reports are available in App Store Connect > TestFlight > Crashes
- Integrate a crash reporter (Firebase Crashlytics, Sentry) for richer data
- Monitor crash-free rate before promoting to App Store
- Address any crashers before wider distribution
