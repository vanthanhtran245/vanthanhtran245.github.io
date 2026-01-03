---
layout: post
title: Dynamic Framework vs Static Framework trong iOS
date: 2026-01-03 00:00:00 +0700
description: Tìm hiểu sự khác biệt giữa Dynamic Framework và Static Framework trong iOS, cách chúng được load vào app, ưu nhược điểm và khi nào nên dùng loại nào.
img: framework_comparison.png
fig-caption: Dynamic vs Static Framework
tags: [ios, framework, swift, xcode]
---

Khi phát triển ứng dụng iOS, việc hiểu rõ sự khác biệt giữa **Dynamic Framework** và **Static Framework** là điều cực kỳ quan trọng. Bài viết này sẽ giúp bạn hiểu sâu về cách chúng hoạt động, cơ chế load vào app, và khi nào nên sử dụng loại nào.

## 📚 Khái niệm cơ bản

### Static Library/Framework

**Static Library** (`.a` file) hoặc **Static Framework** là một tập hợp các object files được đóng gói lại. Khi bạn build app, **tất cả code từ static library sẽ được copy trực tiếp vào binary của app**.

### Dynamic Framework

**Dynamic Framework** (`.framework` hoặc `.dylib`) là một bundle chứa compiled code, resources, và headers. Code **không được copy vào app binary** mà được **link và load tại runtime**.

---

## 🔧 Cơ chế Load vào App

### Static Framework - Link Time Loading

![Static Framework Loading Mechanism]({{site.baseurl}}/assets/img/static_framework_diagram.png)

Với Static Framework, quá trình diễn ra như sau:

```
┌─────────────────┐     ┌──────────────┐     ┌─────────────────────┐
│ Static Library  │────▶│    Linker    │────▶│    App Binary       │
│   (.a file)     │     │  (Link Time) │     │ (Contains all code) │
└─────────────────┘     └──────────────┘     └─────────────────────┘
        │                                              │
        │           CODE IS COPIED                     │
        └──────────────────────────────────────────────┘
```

**Quá trình:**
1. **Compile Time**: Compiler biên dịch source code thành object files (`.o`)
2. **Link Time**: Linker (`ld`) lấy tất cả object files từ static library và **copy trực tiếp** vào app binary
3. **Runtime**: App chạy với tất cả code đã được nhúng sẵn - không cần load thêm gì

### Dynamic Framework - Runtime Loading

![Dynamic Framework Loading Mechanism]({{site.baseurl}}/assets/img/dynamic_framework_diagram.png)

Với Dynamic Framework, quá trình phức tạp hơn:

```
┌─────────────────┐     ┌──────────────┐     ┌─────────────────────┐
│    App Bundle   │     │     dyld     │     │ Dynamic Framework   │
│  (Executable)   │◀───▶│  (Runtime)   │◀───▶│   (.framework)      │
└─────────────────┘     └──────────────┘     └─────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │ LOAD AT RUNTIME   │
                    │ via @rpath        │
                    └───────────────────┘
```

**Quá trình:**
1. **Compile Time**: Framework được compile riêng biệt
2. **Link Time**: Linker chỉ ghi lại **reference** đến framework (không copy code)
3. **Runtime**: Khi app khởi động, `dyld` (dynamic linker) sẽ:
   - Đọc các dependency từ Mach-O header
   - Tìm framework trong `@rpath` (thường là `Frameworks/` trong app bundle)
   - Load framework vào memory
   - Resolve symbols và thực hiện binding

---

## 🔄 So sánh chi tiết: dyld Process

### Launch Time với Dynamic Framework

```
App Launch
    │
    ▼
┌─────────────────────────────────────────┐
│            dyld (Dynamic Linker)         │
├─────────────────────────────────────────┤
│ 1. Load App Executable                   │
│ 2. Parse Mach-O Header                   │
│ 3. Find LC_LOAD_DYLIB commands           │
│ 4. For each dynamic library:             │
│    ├── Search in @rpath                  │
│    ├── Map into memory                   │
│    ├── Verify code signature             │
│    └── Perform symbol binding            │
│ 5. Run initializers (+load, __attribute__)│
│ 6. Jump to main()                        │
└─────────────────────────────────────────┘
```

### Launch Time với Static Framework

```
App Launch
    │
    ▼
┌─────────────────────────────────────────┐
│         Simpler Launch Process           │
├─────────────────────────────────────────┤
│ 1. Load App Executable (contains all)    │
│ 2. Run initializers                      │
│ 3. Jump to main()                        │
└─────────────────────────────────────────┘
```

---

## ⚖️ Ưu và Nhược điểm

### Static Framework

| ✅ Ưu điểm | ❌ Nhược điểm |
|-----------|--------------|
| **Khởi động nhanh hơn** - Không cần load từ disk tại runtime | **App size lớn hơn** - Code được duplicate nếu nhiều target dùng chung |
| **Dễ distribute** - Chỉ cần 1 file binary | **Không thể share giữa apps** - Mỗi app có bản copy riêng |
| **Không có vấn đề về dyld** - Tránh được crash do missing framework | **Build time lâu hơn** - Linker phải copy và link tất cả code |
| **Dead code stripping** - Linker có thể loại bỏ code không dùng | **Update khó** - Phải rebuild toàn bộ app để update library |
| **Bảo mật hơn** - Khó extract ra framework riêng | **Xung đột symbols** - Có thể conflict nếu 2 static libs dùng chung dependency |

### Dynamic Framework

| ✅ Ưu điểm | ❌ Nhược điểm |
|-----------|--------------|
| **App size nhỏ hơn** - Có thể share giữa app và extensions | **Khởi động chậm hơn** - dyld cần thời gian để load |
| **Hot-swappable** - Có thể update framework mà không rebuild app | **Phức tạp hơn** - Cần quản lý đúng `@rpath` và embedding |
| **Memory efficient** - Framework được share trong memory | **Crash potential** - Missing framework = crash ngay lập tức |
| **App Extensions** - Bắt buộc dùng dynamic cho sharing code | **Code signature** - Mỗi framework cần sign riêng |
| **Faster incremental builds** - Chỉ rebuild framework thay đổi | **Không có dead code stripping** - Toàn bộ framework được include |

---

## 📊 Khi nào nên dùng loại nào?

### Sử dụng Static Framework khi:

```swift
// ✅ Third-party libraries nhỏ
// ✅ Core utilities được dùng mọi nơi
// ✅ Không có App Extensions
// ✅ Ưu tiên launch time
// ✅ Distributing closed-source SDK
```

- Thư viện nhỏ, được dùng ở khắp nơi trong app
- Bạn muốn tối ưu launch time
- Không cần share code với App Extensions
- Distributing SDK cho third-party (dễ integrate hơn)

### Sử dụng Dynamic Framework khi:

```swift
// ✅ Sharing code với App Extensions
// ✅ Large frameworks (SwiftUI, Combine, etc.)
// ✅ Modular architecture
// ✅ Hot-patching cần thiết
// ✅ Reducing app bundle size
```

- Bạn có App Extensions (Share, Today, Watch, etc.)
- Cần modular architecture với nhiều targets
- Thư viện lớn không cần load ngay khi app start
- Team lớn, cần build độc lập các modules

---

## 🛠 Cấu hình trong Xcode

### Tạo Static Framework

```ruby
# Podspec
Pod::Spec.new do |s|
  s.name = 'MyLibrary'
  s.static_framework = true  # 👈 Key setting
end
```

Hoặc trong Xcode:
1. Build Settings → **Mach-O Type** → `Static Library`

### Tạo Dynamic Framework

```ruby
# Podspec
Pod::Spec.new do |s|
  s.name = 'MyLibrary'
  # Default is dynamic framework
end
```

Trong Xcode:
1. Build Settings → **Mach-O Type** → `Dynamic Library`
2. General → **Frameworks, Libraries** → Embed & Sign

---

## 🔍 Kiểm tra loại Framework

Bạn có thể kiểm tra một framework là static hay dynamic bằng command:

```bash
# Kiểm tra Mach-O type
$ file MyFramework.framework/MyFramework
MyFramework.framework/MyFramework: Mach-O universal binary with 2 architectures

# Chi tiết hơn
$ otool -l MyFramework.framework/MyFramework | grep -A 2 LC_ID

# Nếu thấy LC_ID_DYLIB → Dynamic
# Nếu không thấy → Static
```

---

## 💡 Best Practices

1. **Prefer Static cho SPM packages** - Swift Package Manager mặc định link static, đây là lựa chọn tốt cho hầu hết trường hợp

2. **Dùng Dynamic cho App Extensions** - Nếu app có extensions, sử dụng dynamic framework để share code và giảm size

3. **Avoid embedding dynamic frameworks** nếu không cần thiết - Mỗi dynamic framework thêm ~100-500ms vào launch time

4. **Merge static libraries** - Nếu có nhiều static libs, consider merge chúng để giảm link time

5. **Use `@_implementationOnly`** - Khi expose API, dùng annotation này để hide internal dependencies

---

## 🎯 Kết luận

Việc chọn giữa Static và Dynamic Framework phụ thuộc vào:

- **App architecture**: Có extensions không?
- **Team size**: Cần build độc lập không?
- **Performance priority**: Launch time hay app size?
- **Distribution method**: SDK cho third-party hay internal use?

Hiểu rõ cơ chế hoạt động của cả hai loại sẽ giúp bạn đưa ra quyết định đúng đắn cho dự án của mình.

---

**Tham khảo thêm:**
- [Apple Documentation - Dynamic Library Programming Topics](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/DynamicLibraries/)
- [WWDC 2016 - Optimizing App Startup Time](https://developer.apple.com/videos/play/wwdc2016/406/)
- [dyld source code](https://opensource.apple.com/source/dyld/)

---
