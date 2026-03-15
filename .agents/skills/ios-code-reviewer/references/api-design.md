# API Design

## Swift Naming Conventions

Follow the [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/).

### Clarity at the Point of Use

**Bad:**
```swift
func remove(_ position: Index) -> Element
list.remove(1) // remove 1? remove at 1?
```

**Good:**
```swift
func remove(at position: Index) -> Element
list.remove(at: 1) // clear
```

### Name Methods by Their Side Effects

- **No side effects** (returns a value): use noun phrases (`distance(to:)`, `suffix(3)`)
- **Has side effects** (mutates): use imperative verbs (`sort()`, `append(_:)`)
- **Mutating/non-mutating pairs:** use `-ed`/`-ing` for the non-mutating form

```swift
// Mutating
mutating func sort()
mutating func append(_ item: Element)

// Non-mutating (returns new value)
func sorted() -> [Element]
func appending(_ item: Element) -> [Element]
```

### Boolean Properties

Read as assertions: `isEmpty`, `isEnabled`, `hasContent`. Avoid `is` prefix for non-boolean properties.

## Protocol-Oriented Patterns

Prefer protocols over class inheritance for shared behavior.

```swift
// Define capability, not identity
protocol Loadable {
    associatedtype Content
    func load() async throws -> Content
}

protocol Cacheable {
    var cacheKey: String { get }
    var cacheExpiry: TimeInterval { get }
}

// Compose behaviors
struct UserProfile: Loadable, Cacheable {
    typealias Content = User
    var cacheKey: String { "user-profile" }
    var cacheExpiry: TimeInterval { 300 }

    func load() async throws -> User {
        try await api.fetchUser()
    }
}
```

Use protocol extensions for default implementations:
```swift
extension Cacheable {
    var cacheExpiry: TimeInterval { 600 } // 10 min default
}
```

## Error Type Design

Define domain-specific errors, not generic ones.

**Bad:**
```swift
enum AppError: Error {
    case failed
    case error(String)
}
```

**Good:**
```swift
enum NetworkError: Error, LocalizedError {
    case unauthorized
    case serverError(statusCode: Int)
    case decodingFailed(underlying: Error)
    case noConnection

    var errorDescription: String? {
        switch self {
        case .unauthorized: "Session expired. Please sign in again."
        case .serverError(let code): "Server error (\(code)). Please try again."
        case .decodingFailed: "Unable to process the response."
        case .noConnection: "No internet connection."
        }
    }
}
```

### Result Type vs Throws

Use `throws` by default. Use `Result` when you need to store or pass errors around.

```swift
// Preferred for most cases
func fetchUser() async throws -> User

// Use Result when storing for later
var lastResult: Result<User, NetworkError>

// Use Result in completion handlers (legacy code)
func fetchUser(completion: @escaping (Result<User, NetworkError>) -> Void)
```

## Access Control

Minimize public surface area. Default to the most restrictive access level that works.

```swift
// Public API of a module
public struct UserService {
    public init(client: APIClient) { ... }
    public func fetchUser(id: String) async throws -> User { ... }

    // Internal helpers - not exposed
    func buildRequest(for id: String) -> URLRequest { ... }
    private let client: APIClient
}
```

Rules:
- `private` for things used only within the declaration
- `fileprivate` rarely, only when multiple types in the same file need access
- `internal` (default) for module-internal use
- `public` only for module boundaries
- `open` only when subclassing across modules is intended

## Extension Organization

Group related functionality into extensions, optionally with `// MARK:`.

```swift
class ProfileViewController: UIViewController {
    // Core properties and lifecycle
}

// MARK: - UITableViewDataSource
extension ProfileViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int { ... }
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell { ... }
}

// MARK: - UITableViewDelegate
extension ProfileViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) { ... }
}
```
