# iOS CI/CD

An agent skill for building, testing, archiving, and deploying iOS apps from the command line.

## When to Activate

- User asks to build, test, or archive an iOS app
- User asks about CI/CD setup for iOS
- User asks about code signing, Fastlane, or TestFlight
- User runs `/ios-build` or `/ios-archive`

## Decision Tree

```
What does the user need?
├── Build the app
│   ├── For development/testing → xcodebuild build
│   ├── For distribution → xcodebuild archive + exportArchive
│   └── Read references/xcodebuild-commands.md
├── Run tests
│   ├── Unit tests → xcodebuild test (specific test target)
│   ├── UI tests → xcodebuild test (UI test target with simulator)
│   └── Read references/xcodebuild-commands.md
├── Code signing issues
│   ├── Local development → Automatic signing in Xcode
│   ├── CI environment → Manual signing or Fastlane match
│   └── Read references/signing-guide.md
├── Automate with Fastlane
│   └── Read references/fastlane-patterns.md
└── Distribute via TestFlight
    └── Read references/testflight-checklist.md
```

## Common Workflow Sequences

**Local development build + test:**
```bash
xcodebuild build -workspace App.xcworkspace -scheme App -destination 'platform=iOS Simulator,name=iPhone 16' | xcpretty
xcodebuild test -workspace App.xcworkspace -scheme App -destination 'platform=iOS Simulator,name=iPhone 16' | xcpretty
```

**CI: Build, test, archive, upload:**
```bash
fastlane match appstore --readonly
fastlane scan
fastlane gym
fastlane pilot upload
```

## Reference Documents

- `references/xcodebuild-commands.md` - Build, test, archive commands
- `references/fastlane-patterns.md` - Fastlane setup and lanes
- `references/signing-guide.md` - Certificates, profiles, entitlements
- `references/testflight-checklist.md` - Pre-submission and distribution
