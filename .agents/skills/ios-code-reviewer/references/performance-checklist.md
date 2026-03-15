# Performance Checklist

## Main Thread Violations

All UI updates must happen on the main thread. Heavy computation must happen off it.

**Detection patterns:**
```swift
// BAD: Network callback updating UI directly
URLSession.shared.dataTask(with: url) { data, _, _ in
    self.label.text = String(data: data!, encoding: .utf8) // off main thread
}

// GOOD:
URLSession.shared.dataTask(with: url) { data, _, _ in
    DispatchQueue.main.async {
        self.label.text = String(data: data!, encoding: .utf8)
    }
}

// GOOD (async/await):
func loadData() async {
    let (data, _) = try await URLSession.shared.data(from: url)
    await MainActor.run {
        self.label.text = String(data: data, encoding: .utf8)
    }
}
```

**Heavy work to move off main thread:**
- JSON parsing of large payloads
- Image resizing/processing
- Core Data imports
- File I/O
- Sorting/filtering large collections

## Image Loading and Caching

**Bad:**
```swift
// In cellForRowAt - blocks main thread, no caching, reloads every scroll
cell.imageView?.image = UIImage(contentsOfFile: imagePath)
```

**Good patterns:**
```swift
// Use system caching with UIImage(named:) for bundled assets only
// For remote/dynamic images, use async loading with caching:

func loadImage(from url: URL) async throws -> UIImage {
    if let cached = ImageCache.shared[url] {
        return cached
    }
    let (data, _) = try await URLSession.shared.data(from: url)
    guard let image = UIImage(data: data) else { throw ImageError.invalid }
    ImageCache.shared[url] = image
    return image
}
```

**Rules:**
- Never load remote images synchronously in `cellForRowAt`
- Cancel in-flight image loads when cells are reused
- Downscale images to the display size before rendering
- Use `preparingThumbnail(of:)` (iOS 15+) for thumbnail generation

## Table/Collection View Cell Reuse

**Common issues:**
```swift
// BAD: Adding subviews in cellForRowAt
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
    let label = UILabel() // new label every reuse!
    cell.contentView.addSubview(label)
    return cell
}
```

**Checklist:**
- Configure subviews in `init` or `awakeFromNib`, not in `cellForRowAt`
- Reset cell state in `prepareForReuse()`
- Cancel async operations (image loads, network calls) in `prepareForReuse()`
- Use `UICollectionView.CellRegistration` (iOS 14+) for type-safe cell configuration
- Use diffable data sources instead of `reloadData()` for animated updates

## Lazy Loading

```swift
// Load expensive resources only when needed
class DetailViewController: UIViewController {
    lazy var heavyFormatter: DateFormatter = {
        let f = DateFormatter()
        f.dateStyle = .full
        f.timeStyle = .medium
        return f
    }()

    lazy var mapView: MKMapView = {
        let map = MKMapView()
        map.translatesAutoresizingMaskIntoConstraints = false
        return map
    }()
}
```

**Apply lazy loading to:**
- Formatters (DateFormatter, NumberFormatter)
- Heavy views not immediately visible
- Database connections
- Large data structures built from disk

## Core Animation Performance

**Offscreen rendering triggers (avoid when possible):**
- `cornerRadius` + `masksToBounds` on layers with content
- `shadow` without a `shadowPath`
- `shouldRasterize` on frequently changing layers

**Fixes:**
```swift
// BAD: forces offscreen rendering
view.layer.cornerRadius = 10
view.layer.masksToBounds = true
view.layer.shadowColor = UIColor.black.cgColor
view.layer.shadowOffset = CGSize(width: 0, height: 2)

// GOOD: provide shadowPath to avoid offscreen pass
view.layer.shadowPath = UIBezierPath(
    roundedRect: view.bounds,
    cornerRadius: 10
).cgPath
```

**Color blending:**
- Set `backgroundColor` on all views (avoid transparent backgrounds stacking)
- Use `isOpaque = true` where possible
- Debug with Instruments > Core Animation > Color Blended Layers

## Memory Leak Detection

**Instruments workflow:**
1. Profile with Leaks instrument
2. Use Allocations to track growth over repeated actions
3. In Debug navigator, watch memory graph for unexpected growth

**In-code detection:**
```swift
// Add deinit logging during development
deinit {
    #if DEBUG
    print("\(type(of: self)) deallocated")
    #endif
}
```

**Common leak sources to check:**
- Closures capturing `self` strongly (see memory-management.md)
- Circular references between view controllers
- Singletons holding references to view controllers
- DispatchQueue blocks retaining objects longer than expected
