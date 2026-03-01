# iOS Agent Skills

Reusable [Claude Code agent skills](https://docs.anthropic.com/en/docs/claude-code/skills) for iOS development. Each skill provides domain-specific knowledge and decision trees that help Claude Code assist you with building, reviewing, securing, and shipping iOS apps.

## Available Skills

| Skill | Description |
|-------|-------------|
| **ios-code-reviewer** | Memory management, API design, performance, accessibility, App Store rejection prevention |
| **ios-security** | Keychain, ATS, certificate pinning, privacy manifests, OWASP Mobile Top 10 |
| **ios-architect** | Modular architecture, dependency injection, navigation patterns, SwiftData migration |
| **ios-ci-cd** | xcodebuild, Fastlane, code signing, TestFlight, GitHub Actions |

## Installation

Install individual skills into your iOS project:

```bash
# Install a specific skill
npx skills add https://github.com/matiastripode/ios-agent-skills --skill ios-code-reviewer
npx skills add https://github.com/matiastripode/ios-agent-skills --skill ios-security
npx skills add https://github.com/matiastripode/ios-agent-skills --skill ios-architect
npx skills add https://github.com/matiastripode/ios-agent-skills --skill ios-ci-cd

# Or install all at once
npx skills add https://github.com/matiastripode/ios-agent-skills
```

## What Are Agent Skills?

Agent skills are structured knowledge packages that Claude Code loads on demand. Each skill contains:

- **SKILL.md** — When to activate, decision trees, output format
- **references/** — Detailed reference documents with code examples, checklists, and patterns

Skills are only loaded when relevant to your question, keeping context focused and efficient.

## Skill Details

### ios-code-reviewer

Reviews Swift/iOS code for common issues:
- Strong reference cycles in closures and delegates
- Main thread violations
- Cell reuse problems in UITableView/UICollectionView
- Missing accessibility labels and Dynamic Type support
- App Store rejection risks (missing privacy keys, export compliance)

### ios-security

Audits iOS apps against security best practices:
- Keychain usage vs insecure storage (UserDefaults for secrets)
- App Transport Security configuration
- Certificate pinning implementation
- Privacy manifests (iOS 17+ requirement)
- OWASP Mobile Top 10 coverage

### ios-architect

Helps design and evolve iOS app architecture:
- SPM modular architecture (when and how to split)
- Dependency injection without third-party frameworks
- NavigationStack, Coordinator, and Router patterns
- SwiftData migration from Core Data

### ios-ci-cd

Automates building, testing, and shipping:
- xcodebuild commands (build, test, archive, export)
- Fastlane setup (match, gym, scan, pilot, deliver)
- Code signing (certificates, profiles, entitlements)
- TestFlight submission checklist
- GitHub Actions workflow templates

## Recommended Companion Skills

These community skills complement the ones in this repo:

```bash
# SwiftUI patterns and best practices
npx skills add https://github.com/AvdLee/SwiftUI-Agent-Skill

# Swift Concurrency (async/await, actors, Sendable)
npx skills add https://github.com/AvdLee/Swift-Concurrency-Agent-Skill

# Swift Testing framework
npx skills add https://github.com/AvdLee/Swift-Testing-Agent-Skill

# Core Data patterns
npx skills add https://github.com/AvdLee/Core-Data-Agent-Skill

# iOS UI design patterns
npx skills add https://github.com/wshobson/agents --skill mobile-ios-design
```

## Contributing

Contributions are welcome! To add or improve a skill:

1. Fork this repo
2. Edit or add files under `skills/<skill-name>/`
3. Follow the existing structure: `SKILL.md` + `references/*.md`
4. Submit a PR with a description of what changed and why

## License

MIT
