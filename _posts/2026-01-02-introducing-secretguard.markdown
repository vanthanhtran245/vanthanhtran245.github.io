---
layout: post
title: Introducing SecretGuard - Protect Your iOS App Secrets
date: 2026-01-02 00:00:00 +0700
description: SecretGuard is a lightweight Swift library that helps protect your hardcoded API keys and secrets from reverse engineering using XOR-based obfuscation.
img: secretguard.png
fig-caption: SecretGuard - Secure Your Secrets
tags: [swift, ios, security, open-source]
---

When building iOS applications, we often need to include API keys, tokens, and other sensitive strings directly in our code. The problem? These strings can be easily extracted from your compiled binary using simple tools like `strings`:

```bash
$ strings YourApp.app/YourApp | grep "sk-"
sk-1234567890abcdef  # 😱 Your secret is exposed!
```

That's why I created **SecretGuard** - a lightweight Swift library that protects your hardcoded secrets from being easily extracted via reverse engineering.

## 🔐 What is SecretGuard?

SecretGuard uses **XOR-based obfuscation** to prevent plain-text strings from appearing in your compiled binary. It works by:

1. **Converting your secrets into obfuscated byte arrays at compile time**
2. **Decoding them back to strings at runtime**

```swift
// Before: Visible in binary!
let apiKey = "sk-1234567890abcdef"  // ❌ 

// After: No plain text in binary!
let apiKey = Secrets.apiKey  // ✅ Decoded at runtime
```

## 🚀 Getting Started

### Installation

Add SecretGuard to your project using Swift Package Manager:

```swift
dependencies: [
    .package(url: "https://github.com/vanthanhtran245/secret-guard.git", from: "1.0.0")
]
```

### Two Methods for Obfuscation

| Method | Best For | Complexity |
|--------|----------|------------|
| **Python Script** | Simple projects, CI/CD pipelines | ⭐ Easy |
| **GYB Template** | Complex projects, many secrets | ⭐⭐ Medium |

### Method 1: Python Script (Recommended)

Create a `secrets.json` file (add this to `.gitignore`!):

```json
{
    "secrets": {
        "apiKey": "your-api-key-here",
        "googleMapsKey": "your-google-maps-key",
        "stripePublicKey": "pk_live_xxxxx"
    }
}
```

Run the obfuscation script:

```bash
python3 path/to/SecretGuard/Scripts/obfuscate.py \
    --config secrets.json \
    --output Sources/Generated/Secrets.swift \
    --salt "your-unique-project-salt"
```

### Method 2: GYB Template

For more complex projects, use the GYB (Generate Your Boilerplate) template:

```python
# Your unique salt (change this!)
SALT = "my-awesome-app-2024-salt"

# Your secrets
SECRETS = [
    ("apiKey", "sk-1234567890abcdef"),
    ("googleMapsKey", "AIzaSyXXXXX"),
    ("firebaseKey", "your-firebase-key"),
]
```

Then add a Run Script Phase in Xcode to generate the Swift file during build.

## 🔍 How It Works

SecretGuard uses XOR (exclusive or) encryption with a random cipher:

### Encoding (Compile Time)
```
Original:   "API"     →  [0x41, 0x50, 0x49]
Cipher:     [random]  →  [0x12, 0x34, 0x56]
Encoded:    XOR       →  [0x53, 0x64, 0x1F]
```

### Decoding (Runtime)
```
Encoded:    [0x53, 0x64, 0x1F]
Cipher:     [0x12, 0x34, 0x56]
Decoded:    XOR       →  [0x41, 0x50, 0x49]  →  "API"
```

The XOR operation is symmetric: `A XOR B XOR B = A`

## ⚠️ Security Disclaimer

**Important**: SecretGuard provides **obfuscation**, not encryption.

### What SecretGuard Does ✅
- Prevents casual extraction of secrets via `strings` command
- Raises the bar for reverse engineering
- Stops automated scanners from finding plain-text keys

### What SecretGuard Does NOT Do ❌
- Provide enterprise-grade security
- Protect against determined attackers with debuggers
- Replace server-side API key management

### Best Practices

1. **Use Environment Variables in CI/CD** - Inject secrets at build time
2. **Rotate Keys Regularly** - Even obfuscated keys can be extracted eventually
3. **Use Backend Proxies** - For high-security scenarios, proxy API calls through your server
4. **Rate Limiting & Monitoring** - Track API usage to detect unauthorized access

## 📖 API Reference

```swift
// Basic decoding
public static func decode(_ encoded: [UInt8], cipher: [UInt8]) -> String

// Optimized for large secrets
public static func decodeOptimized(_ encoded: [UInt8], cipher: [UInt8]) -> String

// Returns raw bytes
public static func decodeBytes(_ encoded: [UInt8], cipher: [UInt8]) -> [UInt8]
```

## 🔗 Links

- **GitHub Repository**: [github.com/vanthanhtran245/secret-guard](https://github.com/vanthanhtran245/secret-guard)
- **License**: MIT

---

SecretGuard is open-source and contributions are welcome! Feel free to submit pull requests or open issues on GitHub.

**Made with ❤️ for the iOS community**
