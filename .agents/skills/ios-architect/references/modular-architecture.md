# Modular Architecture

## When to Modularize

| Situation | Recommendation |
|-----------|---------------|
| Solo, < 10 screens | Folder-based organization in a single target |
| Solo, 10-30 screens | Extract 2-3 shared modules (networking, design system) |
| Team of 2-4, medium app | Feature modules + shared modules |
| Team of 5+, large app | Full modular architecture with clear boundaries |

## Module Types

### Feature Modules
One module per user-facing feature. Contains views, view models, and feature-specific models.

```
FeatureProfile/
├── Sources/
│   ├── ProfileView.swift
│   ├── ProfileViewModel.swift
│   ├── EditProfileView.swift
│   └── Models/
│       └── ProfileState.swift
├── Tests/
│   └── ProfileViewModelTests.swift
└── Package.swift
```

### Shared Modules
Cross-cutting concerns used by multiple features.

```
Networking/        - API client, request building, response handling
DesignSystem/      - Colors, typography, reusable components
Core/              - Extensions, utilities, protocols
Domain/            - Shared domain models, business logic interfaces
Persistence/       - Data layer abstraction
```

## Dependency Rules

1. **Acyclic:** No circular dependencies between modules
2. **Unidirectional:** Features depend on shared modules, never on each other
3. **App shell:** Only the main app target depends on feature modules

```
App (main target)
├── FeatureProfile
│   ├── Networking
│   ├── DesignSystem
│   └── Domain
├── FeatureSettings
│   ├── Networking
│   ├── DesignSystem
│   └── Domain
├── Networking
│   └── Core
├── DesignSystem
│   └── Core
└── Domain
    └── Core
```

## Package.swift Example

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "FeatureProfile",
    platforms: [.iOS(.v17)],
    products: [
        .library(name: "FeatureProfile", targets: ["FeatureProfile"])
    ],
    dependencies: [
        .package(path: "../Networking"),
        .package(path: "../DesignSystem"),
        .package(path: "../Domain")
    ],
    targets: [
        .target(
            name: "FeatureProfile",
            dependencies: ["Networking", "DesignSystem", "Domain"]
        ),
        .testTarget(
            name: "FeatureProfileTests",
            dependencies: ["FeatureProfile"]
        )
    ]
)
```

## Workspace Organization

```
MyApp/
├── MyApp/                    # Main app target (app shell)
│   ├── App.swift
│   ├── AppCoordinator.swift
│   └── DI/
│       └── AppContainer.swift
├── Packages/
│   ├── Core/
│   ├── Networking/
│   ├── DesignSystem/
│   ├── Domain/
│   ├── Persistence/
│   ├── FeatureProfile/
│   ├── FeatureSettings/
│   └── FeatureOnboarding/
├── MyApp.xcodeproj
└── Package.swift             # Root package that references local packages
```

## Build Time Optimization

Modules compile in parallel. Benefits:
- Changed module + dependents recompile, not the whole app
- Feature teams can work independently
- Test targets run faster (only test one module)

**Tips:**
- Keep module dependency depth shallow (max 3-4 levels)
- Avoid large "God modules" that everything depends on
- Use protocols at module boundaries to minimize recompilation cascading

## When to Extract vs Keep in Main Target

**Extract when:**
- Code is used by 2+ features
- Code has its own test suite
- Code changes independently of the app
- You want to enforce access control at module boundaries

**Keep in main target when:**
- Code is app-specific glue (AppDelegate, SceneDelegate)
- One-off utilities used in a single place
- Extraction would create a module with 1-2 files
