# iOS Code Reviewer

An agent skill for reviewing iOS/Swift code for quality, correctness, and App Store readiness.

## When to Activate

- User asks to review iOS/Swift code
- User asks to check code before a PR or submission
- User runs `/ios-review`
- Files being reviewed are `.swift`, `.xib`, `.storyboard`, `Info.plist`, or Xcode project files

## Decision Tree

```
Is the code Swift?
├── YES
│   ├── Does it involve UI? → Check references/performance-checklist.md (main thread, cell reuse)
│   │                        → Check references/accessibility.md (VoiceOver, Dynamic Type)
│   ├── Does it use closures, delegates, or timers? → Check references/memory-management.md
│   ├── Is it a public API or protocol? → Check references/api-design.md
│   ├── Is it preparing for App Store submission? → Check references/app-store-rejection.md
│   └── General review → Check all references as applicable
├── NO (Info.plist, project files, storyboards)
│   └── Check references/app-store-rejection.md for missing keys/configurations
```

## Review Output Format

For each finding, report:

```
### [Severity] Category: Brief Description
- **File:** path/to/file.swift:line
- **Issue:** What is wrong
- **Fix:** How to fix it
- **Reference:** Which reference doc covers this
```

Severity levels: CRITICAL, WARNING, SUGGESTION

## Reference Documents

- `references/memory-management.md` - Retain cycles, ARC, delegate patterns
- `references/api-design.md` - Swift naming, protocols, error handling
- `references/performance-checklist.md` - Main thread, cell reuse, image caching
- `references/app-store-rejection.md` - Common rejection reasons and fixes
- `references/accessibility.md` - VoiceOver, Dynamic Type, tap targets
