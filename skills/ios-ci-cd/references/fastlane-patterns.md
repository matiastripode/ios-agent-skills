# Fastlane Patterns

## Initial Setup

```bash
# Install Fastlane
brew install fastlane
# or
gem install fastlane

# Initialize in project directory
cd MyApp
fastlane init
```

This creates:
```
fastlane/
├── Appfile        # App identifier and Apple ID
├── Fastfile       # Lane definitions
├── Matchfile      # Code signing config (after match init)
└── .env.default   # Environment variables
```

## Appfile

```ruby
app_identifier("com.example.myapp")
apple_id("developer@example.com")
team_id("ABCDE12345")

# For multiple targets
for_platform :ios do
  for_lane :beta do
    app_identifier("com.example.myapp.beta")
  end
end
```

## Fastfile - Common Lanes

```ruby
default_platform(:ios)

platform :ios do

  desc "Run tests"
  lane :test do
    scan(
      workspace: "MyApp.xcworkspace",
      scheme: "MyApp",
      devices: ["iPhone 16"],
      clean: true,
      code_coverage: true,
      output_directory: "fastlane/test_output"
    )
  end

  desc "Build for TestFlight"
  lane :beta do
    ensure_git_status_clean

    # Increment build number
    increment_build_number(
      build_number: latest_testflight_build_number + 1
    )

    # Code signing
    match(type: "appstore", readonly: true)

    # Build
    gym(
      workspace: "MyApp.xcworkspace",
      scheme: "MyApp",
      configuration: "Release",
      export_method: "app-store",
      output_directory: "build"
    )

    # Upload to TestFlight
    pilot(
      skip_waiting_for_build_processing: true,
      distribute_external: false
    )

    # Commit version bump
    commit_version_bump(message: "Bump build number")
    push_to_git_remote
  end

  desc "Deploy to App Store"
  lane :release do
    ensure_git_status_clean

    # Run tests first
    test

    # Build
    match(type: "appstore", readonly: true)
    gym(
      workspace: "MyApp.xcworkspace",
      scheme: "MyApp",
      configuration: "Release",
      export_method: "app-store"
    )

    # Upload to App Store
    deliver(
      submit_for_review: false,
      force: true,
      skip_metadata: false,
      skip_screenshots: true
    )
  end

  desc "Register new device"
  lane :register_device do
    device_name = prompt(text: "Device name: ")
    device_udid = prompt(text: "Device UDID: ")
    register_devices(devices: { device_name => device_udid })
    match(type: "development", force_for_new_devices: true)
  end

end
```

## Match - Code Signing

```bash
# Initialize match (first time only)
fastlane match init
```

**Matchfile:**
```ruby
git_url("https://github.com/yourorg/certificates.git")
storage_mode("git")
type("appstore")
app_identifier(["com.example.myapp"])
username("developer@example.com")
```

```bash
# Generate/download certificates and profiles
fastlane match development    # for development
fastlane match adhoc          # for ad-hoc distribution
fastlane match appstore       # for App Store

# In CI (read-only, never create new)
fastlane match appstore --readonly
```

## Environment Variables

**.env.default:**
```bash
FASTLANE_USER=developer@example.com
FASTLANE_TEAM_ID=ABCDE12345
MATCH_PASSWORD=your-match-encryption-password
FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD=xxxx-xxxx-xxxx-xxxx
```

**Never commit `.env` files to git.** Add to `.gitignore`.

## GitHub Actions Integration

```yaml
name: iOS CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.0.app

      - name: Install dependencies
        run: bundle install

      - name: Run tests
        run: bundle exec fastlane test

  beta:
    needs: test
    runs-on: macos-14
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.0.app

      - name: Install dependencies
        run: bundle install

      - name: Setup SSH for match
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.MATCH_SSH_KEY }}

      - name: Deploy to TestFlight
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD: ${{ secrets.APP_SPECIFIC_PASSWORD }}
          FASTLANE_USER: ${{ secrets.APPLE_ID }}
        run: bundle exec fastlane beta
```

## Useful Actions

```ruby
# Screenshots (for App Store)
lane :screenshots do
  snapshot(
    devices: ["iPhone 16 Pro Max", "iPhone SE (3rd generation)", "iPad Pro 13-inch (M4)"],
    languages: ["en-US"],
    scheme: "MyAppUITests"
  )
end

# Increment version
lane :bump_minor do
  increment_version_number(bump_type: "minor")
  commit_version_bump(message: "Version bump")
end
```
