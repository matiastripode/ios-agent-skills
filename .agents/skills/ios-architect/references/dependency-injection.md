# Dependency Injection

## Constructor Injection (Preferred)

The simplest and most testable pattern. Dependencies are required at initialization.

```swift
protocol UserRepository {
    func fetchUser(id: String) async throws -> User
}

class ProfileViewModel: ObservableObject {
    @Published var user: User?
    private let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }

    func loadUser(id: String) async {
        user = try? await repository.fetchUser(id: id)
    }
}

// Production
let viewModel = ProfileViewModel(repository: APIUserRepository())

// Tests
let viewModel = ProfileViewModel(repository: MockUserRepository())
```

## SwiftUI Environment-Based DI

For SwiftUI apps, leverage the environment system.

```swift
// Define an environment key
private struct UserRepositoryKey: EnvironmentKey {
    static let defaultValue: UserRepository = APIUserRepository()
}

extension EnvironmentValues {
    var userRepository: UserRepository {
        get { self[UserRepositoryKey.self] }
        set { self[UserRepositoryKey.self] = newValue }
    }
}

// Inject at the root
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.userRepository, APIUserRepository())
        }
    }
}

// Consume in views
struct ProfileView: View {
    @Environment(\.userRepository) private var repository

    var body: some View { ... }
}

// Override in previews/tests
struct ProfileView_Previews: PreviewProvider {
    static var previews: some View {
        ProfileView()
            .environment(\.userRepository, MockUserRepository())
    }
}
```

### @Observable with Environment (iOS 17+)

```swift
@Observable
class AppState {
    var currentUser: User?
    var isAuthenticated: Bool = false
}

// Inject
ContentView()
    .environment(AppState())

// Consume
@Environment(AppState.self) private var appState
```

## DI Container (No Third-Party Libraries)

For larger apps, a simple container centralizes dependency creation.

```swift
@MainActor
final class AppContainer {
    // Singletons
    lazy var httpClient: HTTPClient = URLSessionHTTPClient()
    lazy var authService: AuthService = KeychainAuthService()

    // Transient (new instance each time)
    func makeUserRepository() -> UserRepository {
        APIUserRepository(client: httpClient, auth: authService)
    }

    func makeProfileViewModel(userId: String) -> ProfileViewModel {
        ProfileViewModel(
            repository: makeUserRepository(),
            userId: userId
        )
    }
}

// Usage in app
@main
struct MyApp: App {
    let container = AppContainer()

    var body: some Scene {
        WindowGroup {
            ProfileView(viewModel: container.makeProfileViewModel(userId: "123"))
        }
    }
}
```

## Factory Pattern

When you need to create instances with runtime parameters.

```swift
protocol ViewModelFactory {
    func makeProfileViewModel(userId: String) -> ProfileViewModel
    func makeSettingsViewModel() -> SettingsViewModel
}

struct DefaultViewModelFactory: ViewModelFactory {
    let container: AppContainer

    func makeProfileViewModel(userId: String) -> ProfileViewModel {
        ProfileViewModel(repository: container.makeUserRepository(), userId: userId)
    }

    func makeSettingsViewModel() -> SettingsViewModel {
        SettingsViewModel(settingsStore: container.settingsStore)
    }
}
```

## Protocol-Based Abstractions for Testability

Define protocols at module boundaries for easy mocking.

```swift
// In Networking module - public protocol
public protocol APIClient: Sendable {
    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T
}

// In Networking module - production implementation
public struct URLSessionAPIClient: APIClient {
    public func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T { ... }
}

// In test target - mock
final class MockAPIClient: APIClient {
    var mockResult: Any?
    var mockError: Error?

    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        if let error = mockError { throw error }
        return mockResult as! T
    }
}
```

## Service Locator (Anti-Pattern)

Avoid this pattern - it hides dependencies and makes testing harder.

```swift
// BAD: Hidden dependency, hard to test
class ProfileViewModel {
    func loadUser() {
        let repo = ServiceLocator.shared.resolve(UserRepository.self) // hidden
    }
}

// GOOD: Explicit dependency
class ProfileViewModel {
    private let repository: UserRepository

    init(repository: UserRepository) { // visible, testable
        self.repository = repository
    }
}
```

## Guidelines

1. Prefer constructor injection for most cases
2. Use `@Environment` for SwiftUI view dependencies
3. Use a container for apps with 10+ dependencies
4. Define protocols at module boundaries
5. Avoid singletons for stateful services (use DI container instead)
6. Keep the dependency graph shallow (max 3 levels deep)
