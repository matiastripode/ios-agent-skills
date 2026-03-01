# iOS Security

An agent skill for reviewing iOS/Swift code for security vulnerabilities, data protection issues, and privacy compliance.

## When to Activate

- User asks for a security review of iOS code
- User asks about secure storage, networking, or authentication
- User runs `/ios-security-check`
- Code involves: Keychain, UserDefaults with sensitive data, networking, authentication, file storage, or privacy-related APIs

## Decision Tree

```
What area of code is being reviewed?
├── Data Storage
│   ├── Credentials, tokens, secrets → references/keychain-usage.md
│   ├── Files, databases, backups → references/data-protection.md
│   └── UserDefaults → Flag if storing sensitive data
├── Networking
│   ├── API calls, URLSession → references/network-security.md
│   ├── Hardcoded URLs, API keys → references/network-security.md
│   └── ATS configuration → references/network-security.md
├── Privacy
│   ├── Privacy manifest → references/privacy-manifest.md
│   ├── Tracking, analytics → references/privacy-manifest.md
│   └── Required reason APIs → references/privacy-manifest.md
└── General Security Audit
    └── Walk through references/owasp-mobile-top10.md
```

## Severity Levels

- **CRITICAL**: Direct data exposure, hardcoded secrets, no encryption on sensitive data
- **HIGH**: Missing certificate pinning on auth endpoints, weak Keychain configuration
- **MEDIUM**: Missing privacy manifest entries, ATS exceptions without justification
- **LOW**: Debug logging of sensitive data, clipboard exposure

## Output Format

```
### [SEVERITY] Finding Title
- **Category:** Storage / Networking / Privacy / Authentication
- **File:** path/to/file.swift:line
- **Risk:** What could go wrong
- **Fix:** How to remediate
- **Reference:** Which reference doc
```

## Reference Documents

- `references/keychain-usage.md` - Secure storage with Keychain
- `references/network-security.md` - ATS, certificate pinning, API keys
- `references/data-protection.md` - File encryption, backups, clipboard
- `references/privacy-manifest.md` - iOS 17+ privacy requirements
- `references/owasp-mobile-top10.md` - OWASP Mobile Top 10 for iOS
