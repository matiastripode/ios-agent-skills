# xcodebuild Commands

## Building

```bash
# Build with workspace (CocoaPods or SPM with workspace)
xcodebuild build \
    -workspace MyApp.xcworkspace \
    -scheme MyApp \
    -destination 'platform=iOS Simulator,name=iPhone 16' \
    -configuration Debug \
    | xcpretty

# Build with project (no workspace)
xcodebuild build \
    -project MyApp.xcodeproj \
    -scheme MyApp \
    -destination 'platform=iOS Simulator,name=iPhone 16' \
    | xcpretty

# Build for a specific iOS version
xcodebuild build \
    -workspace MyApp.xcworkspace \
    -scheme MyApp \
    -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.0' \
    | xcpretty

# Clean build
xcodebuild clean build \
    -workspace MyApp.xcworkspace \
    -scheme MyApp \
    -destination 'platform=iOS Simulator,name=iPhone 16' \
    | xcpretty
```

## Testing

```bash
# Run all tests
xcodebuild test \
    -workspace MyApp.xcworkspace \
    -scheme MyApp \
    -destination 'platform=iOS Simulator,name=iPhone 16' \
    -resultBundlePath TestResults.xcresult \
    | xcpretty --report junit

# Run specific test target
xcodebuild test \
    -workspace MyApp.xcworkspace \
    -scheme MyApp \
    -destination 'platform=iOS Simulator,name=iPhone 16' \
    -only-testing:MyAppTests \
    | xcpretty

# Run specific test class
xcodebuild test \
    -workspace MyApp.xcworkspace \
    -scheme MyApp \
    -destination 'platform=iOS Simulator,name=iPhone 16' \
    -only-testing:MyAppTests/UserServiceTests \
    | xcpretty

# Run specific test method
xcodebuild test \
    -workspace MyApp.xcworkspace \
    -scheme MyApp \
    -destination 'platform=iOS Simulator,name=iPhone 16' \
    -only-testing:MyAppTests/UserServiceTests/testFetchUser \
    | xcpretty

# Skip UI tests (faster CI)
xcodebuild test \
    -workspace MyApp.xcworkspace \
    -scheme MyApp \
    -destination 'platform=iOS Simulator,name=iPhone 16' \
    -skip-testing:MyAppUITests \
    | xcpretty

# Test on multiple destinations in parallel
xcodebuild test \
    -workspace MyApp.xcworkspace \
    -scheme MyApp \
    -destination 'platform=iOS Simulator,name=iPhone 16' \
    -destination 'platform=iOS Simulator,name=iPad Pro 13-inch (M4)' \
    -parallel-testing-enabled YES \
    | xcpretty
```

## Archiving

```bash
# Step 1: Create archive
xcodebuild archive \
    -workspace MyApp.xcworkspace \
    -scheme MyApp \
    -archivePath build/MyApp.xcarchive \
    -destination 'generic/platform=iOS' \
    -configuration Release

# Step 2: Export IPA
xcodebuild -exportArchive \
    -archivePath build/MyApp.xcarchive \
    -exportPath build/output \
    -exportOptionsPlist ExportOptions.plist
```

## ExportOptions.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store</string> <!-- app-store, ad-hoc, development, enterprise -->
    <key>teamID</key>
    <string>YOUR_TEAM_ID</string>
    <key>signingStyle</key>
    <string>manual</string>
    <key>provisioningProfiles</key>
    <dict>
        <key>com.example.myapp</key>
        <string>MyApp AppStore Profile</string>
    </dict>
    <key>uploadBitcode</key>
    <false/>
    <key>uploadSymbols</key>
    <true/>
</dict>
</plist>
```

## Derived Data Management

```bash
# Custom derived data path (useful for CI)
xcodebuild build \
    -workspace MyApp.xcworkspace \
    -scheme MyApp \
    -derivedDataPath build/DerivedData \
    -destination 'platform=iOS Simulator,name=iPhone 16'

# Clean derived data
rm -rf ~/Library/Developer/Xcode/DerivedData/MyApp-*

# Show build settings
xcodebuild -workspace MyApp.xcworkspace -scheme MyApp -showBuildSettings
```

## Simulator Management (xcrun simctl)

```bash
# List available simulators
xcrun simctl list devices available

# Boot a simulator
xcrun simctl boot "iPhone 16"

# Install app on simulator
xcrun simctl install booted build/Debug-iphonesimulator/MyApp.app

# Launch app on simulator
xcrun simctl launch booted com.example.myapp

# Take screenshot
xcrun simctl io booted screenshot screenshot.png

# Record video
xcrun simctl io booted recordVideo video.mp4
# Press Ctrl+C to stop recording

# Reset simulator (clean state)
xcrun simctl erase "iPhone 16"

# Open URL in simulator (deep link testing)
xcrun simctl openurl booted "myapp://profile/123"

# Set status bar (for screenshots)
xcrun simctl status_bar "iPhone 16" override \
    --time "09:41" --batteryState charged --batteryLevel 100

# Show available destinations
xcodebuild -workspace MyApp.xcworkspace -scheme MyApp -showdestinations
```

## Common Errors

| Error | Fix |
|-------|-----|
| `No signing certificate` | Install certificates or use `fastlane match` |
| `No profiles found` | Download profiles in Xcode or use `fastlane match` |
| `Simulator not found` | Run `xcrun simctl list` and use exact device name |
| `Module not found` | Clean derived data, resolve packages |
| `Build input file not found` | Run `xcodebuild -resolvePackageDependencies` first |
