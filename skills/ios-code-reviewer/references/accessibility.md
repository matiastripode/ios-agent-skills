# Accessibility

## VoiceOver Support

Every interactive element must have meaningful accessibility properties.

```swift
// UIKit
button.accessibilityLabel = "Add to favorites"
button.accessibilityHint = "Double tap to add this item to your favorites list"
button.accessibilityTraits = .button

// Image-only buttons need labels
heartButton.accessibilityLabel = "Favorite"
heartButton.isAccessibilityElement = true

// SwiftUI
Button(action: toggleFavorite) {
    Image(systemName: "heart.fill")
}
.accessibilityLabel("Add to favorites")
.accessibilityHint("Adds this item to your favorites list")
```

**Rules:**
- Every `UIButton`, `UIControl`, and interactive element needs an `accessibilityLabel`
- Icons without text must have explicit labels
- Decorative images should set `isAccessibilityElement = false`
- Container views that group content should use `accessibilityElements` to define reading order
- Use `.accessibilityAddTraits(.isHeader)` for section headers

## Dynamic Type

Support the user's preferred text size throughout the app.

```swift
// UIKit
label.font = UIFont.preferredFont(forTextStyle: .body)
label.adjustsFontForContentSizeCategory = true
label.numberOfLines = 0 // allow wrapping

// SwiftUI (automatic with system fonts)
Text("Hello")
    .font(.body) // responds to Dynamic Type automatically

// Custom fonts with scaling
Text("Hello")
    .font(.custom("Avenir", size: 17, relativeTo: .body))
```

**Checklist:**
- [ ] All text uses text styles (`.body`, `.headline`, `.caption`) not fixed point sizes
- [ ] Labels allow multiline wrapping (`numberOfLines = 0`)
- [ ] Layout adapts when text size increases (test with Accessibility Inspector)
- [ ] No text is truncated at the largest accessibility sizes
- [ ] Scroll views accommodate expanded text

## Minimum Tap Target Sizes

Apple HIG requires minimum 44x44 point tap targets.

```swift
// UIKit - ensure minimum size
button.frame = CGRect(x: 0, y: 0, width: 44, height: 44)

// SwiftUI
Button("Action") { }
    .frame(minWidth: 44, minHeight: 44)

// For small icons, expand the tappable area
Image(systemName: "xmark")
    .frame(width: 44, height: 44)
    .contentShape(Rectangle()) // expand tap area
```

**Review for:**
- Small icon buttons (close, info, navigation arrows)
- Custom tab bar items
- Segmented controls with many segments
- List accessories

## Color Contrast

WCAG 2.1 requires:
- **Normal text:** 4.5:1 contrast ratio minimum
- **Large text (18pt+ or 14pt+ bold):** 3:1 contrast ratio minimum

```swift
// Use semantic colors that adapt to accessibility settings
label.textColor = .label           // adapts to light/dark
view.backgroundColor = .systemBackground

// SwiftUI
Text("Hello")
    .foregroundStyle(.primary)     // adapts automatically
```

**Rules:**
- Never rely on color alone to convey information (add icons, patterns, or labels)
- Use semantic/system colors that adapt to increased contrast settings
- Test with Settings > Accessibility > Increase Contrast enabled

## Reduce Motion

Respect the user's motion preference.

```swift
// UIKit
if UIAccessibility.isReduceMotionEnabled {
    // Use fade instead of slide animation
} else {
    // Full animation
}

// SwiftUI
@Environment(\.accessibilityReduceMotion) var reduceMotion

var body: some View {
    content
        .animation(reduceMotion ? .none : .spring(), value: isExpanded)
}
```

## Accessibility Identifiers for UI Testing

Separate from accessibility labels. These do not affect VoiceOver.

```swift
// UIKit
loginButton.accessibilityIdentifier = "login_button"
emailField.accessibilityIdentifier = "email_text_field"

// SwiftUI
TextField("Email", text: $email)
    .accessibilityIdentifier("email_text_field")

// In UI tests
let loginButton = app.buttons["login_button"]
loginButton.tap()
```

**Convention:** Use `snake_case` for identifiers. Keep them stable across releases for reliable tests.

## Review Checklist

- [ ] All interactive elements have `accessibilityLabel`
- [ ] Decorative images are hidden from VoiceOver
- [ ] All text supports Dynamic Type
- [ ] Tap targets are at least 44x44 points
- [ ] Color is not the sole indicator of state
- [ ] Animations respect Reduce Motion
- [ ] Screen reader navigation order is logical
- [ ] UI test identifiers are set on key elements
