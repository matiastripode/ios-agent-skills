---
name: ios-architect
description: Helps design and evolve iOS app architecture — project structure, modularization, dependency injection, navigation patterns, and SwiftData migration
---

# iOS Architect

An agent skill for making iOS architecture decisions: project structure, module design, navigation patterns, and data layer choices.

## When to Activate

- User asks about project structure or architecture
- User is starting a new iOS project
- User asks about modularization, navigation, or dependency injection
- User asks about migrating from Core Data to SwiftData
- Code review reveals architectural concerns

## Decision Tree

```
What is the architectural question?
├── Project Structure / Modularization
│   ├── Solo developer, small app → Single target, folder-based organization
│   ├── Solo developer, growing app → Extract shared modules (networking, design system)
│   ├── Team, medium app → Feature modules via SPM
│   └── Team, large app → Full modular architecture
│   → Read references/modular-architecture.md
├── Dependency Management
│   ├── How to inject dependencies? → references/dependency-injection.md
│   ├── SwiftUI app → @Environment, constructor injection
│   └── UIKit app → Constructor injection, DI container
├── Navigation
│   ├── Simple linear flow → NavigationStack
│   ├── Complex multi-flow app → Coordinator pattern
│   ├── Deep linking required → Router + URL handling
│   └── Tab-based with nested nav → NavigationStack per tab
│   → Read references/navigation-patterns.md
└── Data Layer
    ├── New project → SwiftData (if iOS 17+ minimum)
    ├── Existing Core Data → Evaluate migration
    └── Migration path? → references/swiftdata-migration.md
```

## Questions to Ask Before Recommending

1. What is the deployment target (minimum iOS version)?
2. Solo developer or team?
3. Expected app size (number of screens/features)?
4. Is this a new project or refactoring an existing one?
5. Any existing architectural patterns already in use?

## Reference Documents

- `references/modular-architecture.md` - SPM modules, feature modules, dependency rules
- `references/dependency-injection.md` - DI patterns, containers, testability
- `references/navigation-patterns.md` - NavigationStack, Coordinator, deep linking
- `references/swiftdata-migration.md` - Core Data to SwiftData migration path
