# Navigation Patterns

## NavigationStack (iOS 16+)

Value-based, type-safe navigation. The recommended approach for new projects.

```swift
struct ContentView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            HomeView()
                .navigationDestination(for: User.self) { user in
                    ProfileView(user: user)
                }
                .navigationDestination(for: Settings.self) { settings in
                    SettingsView(settings: settings)
                }
        }
    }

    // Programmatic navigation
    func showProfile(_ user: User) {
        path.append(user)
    }

    func popToRoot() {
        path = NavigationPath()
    }
}
```

**Navigation values must be `Hashable`:**
```swift
struct User: Hashable, Identifiable {
    let id: String
    let name: String
}
```

## Tab-Based Navigation with Nested Stacks

Each tab gets its own independent `NavigationStack`.

```swift
struct MainTabView: View {
    @State private var selectedTab = 0

    var body: some View {
        TabView(selection: $selectedTab) {
            NavigationStack {
                HomeView()
            }
            .tabItem { Label("Home", systemImage: "house") }
            .tag(0)

            NavigationStack {
                SearchView()
            }
            .tabItem { Label("Search", systemImage: "magnifyingglass") }
            .tag(1)

            NavigationStack {
                ProfileView()
            }
            .tabItem { Label("Profile", systemImage: "person") }
            .tag(2)
        }
    }
}
```

## Coordinator Pattern in SwiftUI

Separates navigation logic from views. Useful for complex multi-step flows.

```swift
@Observable
class AppCoordinator {
    var path = NavigationPath()
    var sheet: Sheet?
    var fullScreenCover: FullScreenCover?

    enum Sheet: Identifiable {
        case settings
        case newPost

        var id: String { String(describing: self) }
    }

    enum FullScreenCover: Identifiable {
        case onboarding
        case login

        var id: String { String(describing: self) }
    }

    // Navigation actions
    func showProfile(_ user: User) {
        path.append(user)
    }

    func showSettings() {
        sheet = .settings
    }

    func showOnboarding() {
        fullScreenCover = .onboarding
    }

    func popToRoot() {
        path = NavigationPath()
    }

    func dismissSheet() {
        sheet = nil
    }
}

// In the root view
struct RootView: View {
    @State private var coordinator = AppCoordinator()

    var body: some View {
        NavigationStack(path: $coordinator.path) {
            HomeView()
                .navigationDestination(for: User.self) { user in
                    ProfileView(user: user)
                }
        }
        .sheet(item: $coordinator.sheet) { sheet in
            switch sheet {
            case .settings: SettingsView()
            case .newPost: NewPostView()
            }
        }
        .fullScreenCover(item: $coordinator.fullScreenCover) { cover in
            switch cover {
            case .onboarding: OnboardingView()
            case .login: LoginView()
            }
        }
        .environment(coordinator)
    }
}

// Any child view can navigate
struct HomeView: View {
    @Environment(AppCoordinator.self) private var coordinator

    var body: some View {
        Button("Show Profile") {
            coordinator.showProfile(user)
        }
    }
}
```

## Deep Linking

Handle URLs and Universal Links to navigate directly to content.

```swift
@Observable
class DeepLinkHandler {
    let coordinator: AppCoordinator

    init(coordinator: AppCoordinator) {
        self.coordinator = coordinator
    }

    func handle(_ url: URL) {
        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false) else { return }

        // myapp://profile/123
        // https://myapp.com/profile/123
        let pathComponents = components.path.split(separator: "/").map(String.init)

        switch pathComponents.first {
        case "profile":
            if let userId = pathComponents.dropFirst().first {
                coordinator.popToRoot()
                coordinator.showProfile(User(id: userId, name: ""))
            }
        case "settings":
            coordinator.showSettings()
        default:
            break
        }
    }
}

// In App
@main
struct MyApp: App {
    @State private var coordinator = AppCoordinator()

    var body: some Scene {
        WindowGroup {
            RootView()
                .onOpenURL { url in
                    DeepLinkHandler(coordinator: coordinator).handle(url)
                }
        }
    }
}
```

**Universal Links setup:**
1. Add Associated Domains entitlement: `applinks:yourdomain.com`
2. Host `apple-app-site-association` file at `https://yourdomain.com/.well-known/apple-app-site-association`

```json
{
    "applinks": {
        "apps": [],
        "details": [{
            "appID": "TEAMID.com.example.myapp",
            "paths": ["/profile/*", "/settings"]
        }]
    }
}
```

## Router Pattern

Decouples view creation from navigation decisions. Useful with DI containers.

```swift
enum Route: Hashable {
    case profile(userId: String)
    case settings
    case postDetail(postId: String)
}

struct AppRouter {
    let container: AppContainer

    @ViewBuilder
    func view(for route: Route) -> some View {
        switch route {
        case .profile(let userId):
            ProfileView(viewModel: container.makeProfileViewModel(userId: userId))
        case .settings:
            SettingsView(viewModel: container.makeSettingsViewModel())
        case .postDetail(let postId):
            PostDetailView(viewModel: container.makePostDetailViewModel(postId: postId))
        }
    }
}

// Usage with NavigationStack
NavigationStack(path: $path) {
    HomeView()
        .navigationDestination(for: Route.self) { route in
            router.view(for: route)
        }
}
```

## Choosing a Pattern

| App Complexity | Pattern |
|---------------|---------|
| Simple (3-5 screens, linear flow) | NavigationStack directly |
| Medium (10-20 screens, tabs + modals) | Coordinator |
| Complex (deep linking, multi-flow, DI) | Coordinator + Router |
| UIKit legacy with SwiftUI | UIKit Coordinator wrapping SwiftUI views |
