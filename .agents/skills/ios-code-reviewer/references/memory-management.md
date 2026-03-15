# Memory Management

## Strong Reference Cycles in Closures

The most common memory leak in iOS. Closures capture `self` strongly by default.

**Bad:**
```swift
class ProfileViewController: UIViewController {
    var viewModel: ProfileViewModel?

    func loadData() {
        viewModel?.onUpdate = {
            self.tableView.reloadData() // strong capture of self
        }
    }
}
```

**Good:**
```swift
func loadData() {
    viewModel?.onUpdate = { [weak self] in
        self?.tableView.reloadData()
    }
}
```

**When to use `[unowned self]`:** Only when you can guarantee `self` outlives the closure (e.g., closures that are synchronous or owned by self with no possibility of outliving it). When in doubt, use `[weak self]`.

## Delegate Patterns

Delegates must be `weak` to avoid retain cycles.

**Bad:**
```swift
protocol DataManagerDelegate: AnyObject {
    func didUpdate()
}

class DataManager {
    var delegate: DataManagerDelegate? // strong reference - LEAK
}
```

**Good:**
```swift
class DataManager {
    weak var delegate: DataManagerDelegate?
}
```

Note: The protocol must be constrained to `AnyObject` (or be a class protocol) for `weak` to work.

## NotificationCenter Observers

With block-based observers, you must remove them manually.

**Bad:**
```swift
class MyViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        NotificationCenter.default.addObserver(
            forName: .dataDidChange,
            object: nil,
            queue: .main
        ) { _ in
            self.refresh() // retains self, never removed
        }
    }
}
```

**Good:**
```swift
class MyViewController: UIViewController {
    private var observer: NSObjectProtocol?

    override func viewDidLoad() {
        super.viewDidLoad()
        observer = NotificationCenter.default.addObserver(
            forName: .dataDidChange,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.refresh()
        }
    }

    deinit {
        if let observer { NotificationCenter.default.removeObserver(observer) }
    }
}
```

Note: Selector-based `addObserver(_:selector:name:object:)` auto-removes in modern iOS (iOS 9+), but block-based does not.

## Timer Retain Cycles

`Timer.scheduledTimer(withTimeInterval:repeats:block:)` retains its target until invalidated.

**Bad:**
```swift
class PollingController {
    var timer: Timer?

    func start() {
        timer = Timer.scheduledTimer(withTimeInterval: 5, repeats: true) { _ in
            self.poll() // strong capture
        }
    }
}
```

**Good:**
```swift
func start() {
    timer = Timer.scheduledTimer(withTimeInterval: 5, repeats: true) { [weak self] _ in
        self?.poll()
    }
}

deinit {
    timer?.invalidate()
}
```

## Collections Holding Strong References

Be careful with caches and collections that hold objects.

**Problem:** `NSCache` holds strong references to values. A dictionary of closures or view controllers can cause unexpected retention.

**Solution:** Use `NSMapTable` with weak values, `NSHashTable` with weak objects, or `NSPointerArray` for weak collections when needed.

```swift
// Weak-value dictionary alternative
let cache = NSMapTable<NSString, AnyObject>.strongToWeakObjects()
```

## Detection Patterns

During code review, flag these patterns:
1. Any closure assigned to a stored property without `[weak self]`
2. Any `delegate` property not marked `weak`
3. Block-based NotificationCenter observers without removal in `deinit`
4. `Timer` creation without `[weak self]` and without `invalidate()` in `deinit`
5. Completion handlers stored as properties (should be called and set to nil)
