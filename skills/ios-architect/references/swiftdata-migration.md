# SwiftData Migration (from Core Data)

## When to Migrate

**Migrate if:**
- Minimum deployment target is iOS 17+
- Starting new features that need persistence
- Core Data stack is relatively simple (few entities, no complex migrations)
- You want to leverage SwiftUI integration with `@Query`

**Stay on Core Data if:**
- Need to support iOS 16 or earlier
- Have complex migration history with mapping models
- Heavily use `NSFetchedResultsController` with UIKit
- Use advanced features (derived attributes, abstract entities)
- CloudKit integration is working well with `NSPersistentCloudKitContainer`

## API Comparison

| Core Data | SwiftData | Notes |
|-----------|-----------|-------|
| `NSManagedObject` | `@Model` class | SwiftData uses macro |
| `NSManagedObjectContext` | `ModelContext` | Similar role |
| `NSPersistentContainer` | `ModelContainer` | Configuration and storage |
| `NSFetchRequest` | `FetchDescriptor` | Type-safe |
| `NSPredicate` | `#Predicate` macro | Compile-time checked |
| `@FetchRequest` (SwiftUI) | `@Query` | Automatic UI updates |
| `NSFetchedResultsController` | `@Query` | SwiftUI only for SwiftData |
| `.xcdatamodeld` | Swift code | No model editor needed |

## Coexistence Strategy

Run both stacks simultaneously during migration. Share the same SQLite store.

```swift
import CoreData
import SwiftData

// Step 1: Configure both to use the same store
let storeURL = URL.applicationSupportDirectory.appending(path: "Model.sqlite")

// Core Data setup (existing)
let coreDataContainer = NSPersistentContainer(name: "Model")
let description = NSPersistentStoreDescription(url: storeURL)
coreDataContainer.persistentStoreDescriptions = [description]
coreDataContainer.loadPersistentStores { _, error in
    if let error { fatalError("Core Data failed: \(error)") }
}

// SwiftData setup (new)
let schema = Schema([UserSD.self, PostSD.self])
let config = ModelConfiguration(url: storeURL)
let modelContainer = try ModelContainer(for: schema, configurations: [config])
```

**Important:** Entity names in SwiftData `@Model` classes must match Core Data entity names exactly for shared store access.

## Step-by-Step Migration Process

### 1. Create SwiftData models matching Core Data entities

```swift
// Core Data (existing)
// Entity: User, Attributes: name (String), email (String), createdAt (Date)

// SwiftData (new)
@Model
final class UserSD {
    var name: String
    var email: String
    var createdAt: Date

    init(name: String, email: String, createdAt: Date = .now) {
        self.name = name
        self.email = email
        self.createdAt = createdAt
    }
}
```

### 2. Migrate feature by feature

Don't migrate everything at once. Pick one feature/screen:

```swift
// Before: Core Data fetch in UIKit
let request: NSFetchRequest<User> = User.fetchRequest()
request.predicate = NSPredicate(format: "name CONTAINS[cd] %@", searchText)
request.sortDescriptors = [NSSortDescriptor(key: "name", ascending: true)]
let users = try context.fetch(request)

// After: SwiftData with SwiftUI
@Query(filter: #Predicate<UserSD> { user in
    user.name.localizedStandardContains(searchText)
}, sort: \.name)
var users: [UserSD]
```

### 3. Update the app entry point

```swift
@main
struct MyApp: App {
    let container: ModelContainer

    init() {
        do {
            container = try ModelContainer(for: UserSD.self, PostSD.self)
        } catch {
            fatalError("Failed to create ModelContainer: \(error)")
        }
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(container)
    }
}
```

### 4. Remove Core Data code after full migration

- Delete `.xcdatamodeld` file
- Remove `NSPersistentContainer` setup
- Remove `NSManagedObject` subclasses
- Remove Core Data import statements

## Relationship Mapping

```swift
// Core Data: to-many relationship defined in model editor

// SwiftData: defined in code
@Model
final class AuthorSD {
    var name: String
    @Relationship(deleteRule: .cascade, inverse: \BookSD.author)
    var books: [BookSD]

    init(name: String, books: [BookSD] = []) {
        self.name = name
        self.books = books
    }
}

@Model
final class BookSD {
    var title: String
    var author: AuthorSD?

    init(title: String, author: AuthorSD? = nil) {
        self.title = title
        self.author = author
    }
}
```

## Known Limitations of SwiftData

As of iOS 17/18:
- No equivalent to `NSFetchedResultsController` for UIKit
- No abstract entities
- No derived attributes
- Limited migration control compared to Core Data mapping models
- `#Predicate` does not support all Core Data predicate operations
- No background context equivalent (use `ModelActor` for background work)
- CloudKit sync is less mature than `NSPersistentCloudKitContainer`
- Unique constraints behave differently than Core Data

```swift
// Background work with ModelActor
@ModelActor
actor DataImporter {
    func importItems(_ items: [ImportData]) throws {
        for item in items {
            let model = ItemSD(name: item.name, value: item.value)
            modelContext.insert(model)
        }
        try modelContext.save()
    }
}
```

## Rollback Strategy

If migration fails or SwiftData has blocking issues:

1. **Keep Core Data models** in the project until SwiftData is fully validated
2. **Feature flag** the SwiftData code path
3. **Keep the same SQLite store** so rolling back doesn't lose data
4. **Test thoroughly** on device (not just simulator) before removing Core Data

```swift
// Feature flag approach
enum PersistenceMode {
    static var useSwiftData: Bool {
        #if DEBUG
        return UserDefaults.standard.bool(forKey: "useSwiftData")
        #else
        return true // or false for safe rollback
        #endif
    }
}
```
