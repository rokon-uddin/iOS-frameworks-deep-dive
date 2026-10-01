# Deep Dive: iOS Libraries, Frameworks, Bundles, XCFrameworks, Linking, Code Signing & Runtime Loading

A practical and technical reference for understanding how reusable code is built, packaged, linked, signed, distributed, loaded, and executed on Apple platforms.

---

## Table of Contents

1. [The Big Picture](#1-the-big-picture)
2. [Core Terminology](#2-core-terminology)
3. [Source Code to Executable: What Actually Happens](#3-source-code-to-executable-what-actually-happens)
4. [Compiler Pipeline: Swift to Machine Code](#4-compiler-pipeline-swift-to-machine-code)
5. [Object Files](#5-object-files)
6. [What the Linker Does](#6-what-the-linker-does)
7. [Libraries](#7-libraries)
8. [Static vs Dynamic Libraries](#8-static-vs-dynamic-libraries)
9. [Frameworks](#9-frameworks)
10. [Framework vs Library](#10-framework-vs-library)
11. [Bundles](#11-bundles)
12. [Framework vs Bundle](#12-framework-vs-bundle)
13. [Mach-O](#13-mach-o)
14. [XCFramework](#14-xcframework)
15. [Swift Package Manager](#15-swift-package-manager)
16. [Embedding](#16-embedding)
17. [Runtime Paths: `@rpath`, `@loader_path`, `@executable_path`](#17-runtime-paths-rpath-loader_path-executable_path)
18. [Code Signing](#18-code-signing)
19. [Certificates, Keys, Provisioning Profiles & Entitlements](#19-certificates-keys-provisioning-profiles--entitlements)
20. [What Happens When an iOS App Launches](#20-what-happens-when-an-ios-app-launches)
21. [dyld](#21-dyld)
22. [Swift & Objective-C Runtime Initialization](#22-swift--objective-c-runtime-initialization)
23. [Static vs Dynamic: Performance Trade-offs](#23-static-vs-dynamic-performance-trade-offs)
24. [Real-World SDK Architecture](#24-real-world-sdk-architecture)
25. [SDK API Design Best Practices](#25-sdk-api-design-best-practices)
26. [Resource Packaging Best Practices](#26-resource-packaging-best-practices)
27. [Binary Compatibility & Library Evolution](#27-binary-compatibility--library-evolution)
28. [Debugging & Inspection Tools](#28-debugging--inspection-tools)
29. [Common Problems](#29-common-problems)
30. [Decision Guide](#30-decision-guide)
31. [Interview-Ready Answers](#31-interview-ready-answers)
32. [Mental Model Summary](#32-mental-model-summary)

---

# 1. The Big Picture

The following is the most useful mental model:

```text
Swift / Obj-C / C / C++
        │
        ▼
     Compiler
        │
        ▼
    Object files
       (.o)
        │
        ├──────────────┐
        │              │
        ▼              ▼
 Static Library      Other Libraries /
     (.a)            Frameworks
        │              │
        └──────┬───────┘
               ▼
             Linker
               │
               ▼
          Mach-O Binary
               │
        ┌──────┴────────┐
        │               │
        ▼               ▼
     Executable      Dynamic Library
        │               │
        └──────┬────────┘
               ▼
            Framework
      code + metadata + resources
               │
               ▼
          XCFramework
  multiple platform/architecture variants
               │
               ▼
         App Integration
               │
        Link / Embed / Sign
               │
               ▼
            .app Bundle
               │
               ▼
             iOS
               │
               ▼
             dyld
               │
               ▼
          Runtime Setup
               │
               ▼
            Execution
```

Important:

> This is not a perfectly linear pipeline.

For example:

- A static library is normally **input to the linker**
- A framework is a **packaging format**
- An XCFramework is a **distribution container**
- A bundle is a **general packaging concept**
- Static vs dynamic describes **linking behavior**
- Mach-O describes the **binary file format**

---

# 2. Core Terminology

| Term | What it is | Contains executable code? | Can contain resources? | Primary purpose |
|---|---|---:|---:|---|
| `.swift` | Source code | No | No | Human-written Swift |
| `.m` / `.mm` | Obj-C / Obj-C++ source | No | No | Human-written Obj-C |
| `.o` | Object file | Yes | No | Compiler output for linker |
| `.a` | Static library archive | Yes | Usually no | Collection of `.o` files |
| `.dylib` | Dynamic library | Yes | Usually no | Runtime-loadable compiled code |
| `.framework` | Specialized bundle | Usually yes | Yes | Code + module metadata + resources |
| `.xcframework` | Distribution container | Indirectly | Yes | Multiple framework/library variants |
| `.bundle` | Resource-oriented bundle | Usually no | Yes | Package resources |
| `.app` | App bundle | Yes | Yes | Installable application |
| `.appex` | App extension bundle | Yes | Yes | Widget, extension, etc. |
| Mach-O | Binary file format | Yes | N/A | Executables/libraries on Apple platforms |
| Swift Package | Build/distribution description | Depends | Yes | Dependency management/distribution |
| SDK | Developer-facing product | Depends | Yes | Frameworks + docs + APIs + tools + samples |

---

# 3. Source Code to Executable: What Actually Happens

Suppose you write:

```swift
public struct Calculator {
    public init() {}

    public func add(_ a: Int, _ b: Int) -> Int {
        a + b
    }
}
```

The CPU cannot execute Swift directly.

An iPhone CPU executes ARM64 machine instructions.

Conceptually:

```text
Swift source
   ↓
Compiler frontend
   ↓
Intermediate representations
   ↓
Machine code
   ↓
Object files
   ↓
Linker
   ↓
Mach-O executable/library
```

Eventually:

```swift
a + b
```

may become something roughly equivalent to:

```asm
add x0, x0, x1
ret
```

---

# 4. Compiler Pipeline: Swift to Machine Code

Swift compilation is approximately:

```text
Swift Source
    │
    ▼
Lexer / Parser
    │
    ▼
AST
    │
    ▼
Type Checking
    │
    ▼
SIL
    │
    ▼
LLVM IR
    │
    ▼
ARM64 Machine Code
    │
    ▼
Object File (.o)
```

## 4.1 AST

AST = Abstract Syntax Tree.

This:

```swift
func add(_ a: Int, _ b: Int) -> Int {
    a + b
}
```

becomes conceptually:

```text
FunctionDecl
├── name: add
├── param: a : Int
├── param: b : Int
├── return: Int
└── BinaryExpression
    ├── a
    ├── +
    └── b
```

---

## 4.2 Type Checking

The compiler resolves:

- overloaded methods
- generics
- protocol conformances
- async/throws semantics
- actor isolation
- Sendable rules
- function return types

Example:

```swift
let value: Int = "hello"
```

fails during semantic/type checking.

---

## 4.3 SIL

SIL = Swift Intermediate Language.

Swift uses SIL because LLVM does not directly model high-level Swift concepts such as:

- ARC
- ownership
- borrowing
- protocol dispatch
- generics
- async functions
- actors
- closures
- value semantics

Inspect SIL:

```bash
swiftc -emit-sil Calculator.swift
```

---

## 4.4 LLVM IR

SIL is lowered to LLVM IR.

Conceptually:

```llvm
define i64 @add(i64 %a, i64 %b) {
    %result = add i64 %a, %b
    ret i64 %result
}
```

LLVM then generates target-specific machine code.

For iPhone:

```text
Target: ARM64
```

---

## 4.5 Compiler Optimizations

The compiler may perform:

- constant folding
- dead-code elimination
- function inlining
- ARC optimization
- copy elimination
- generic specialization
- devirtualization
- bounds-check elimination

Example:

```swift
func value() -> Int {
    let x = 10
    let y = 20
    return x + y
}
```

can potentially become effectively:

```swift
return 30
```

in an optimized build.

---

# 5. Object Files

A compiler normally emits `.o` files.

Example:

```text
Payment.o
Network.o
Crypto.o
Models.o
```

An object file contains:

```text
Object File
├── machine code
├── defined symbols
├── undefined symbols
├── relocation information
├── sections
└── metadata
```

An object file is usually **not yet a complete executable**.

---

## 5.1 Defined vs Undefined Symbols

Suppose:

```swift
func loadUser() async throws -> User {
    try await APIClient.shared.fetchUser()
}
```

The object file may conceptually contain:

```text
Defined:
    UserService.loadUser

Undefined:
    APIClient.shared
    APIClient.fetchUser
```

The linker later resolves those references.

---

# 6. What the Linker Does

The linker combines compiled pieces into a complete binary.

Inputs may include:

```text
AppDelegate.o
PaymentView.o
UserService.o
libPaymentSDK.a
Foundation.framework
Security.framework
```

The linker performs several important jobs.

---

## 6.1 Symbol Resolution

If:

```text
PaymentView.o
requires:
    PaymentSDK.pay()
```

and:

```text
Payment.o
provides:
    PaymentSDK.pay()
```

the linker connects them.

---

## 6.2 Relocation

At compile time, the final memory address of a function may not be known.

The object file may effectively contain:

```text
call ????
```

The linker determines the correct relative location and patches references.

This is **relocation**.

---

## 6.3 Dead-Code Stripping

If a library contains:

```text
Payments
Analytics
QRScanner
DebugTools
Crypto
```

but the app only uses:

```text
Payments
Crypto
```

the linker may remove unused code.

Common build setting:

```text
DEAD_CODE_STRIPPING = YES
```

---

# 7. Libraries

A **library** is reusable compiled code intended to be consumed by another binary.

Two major forms:

```text
Library
├── Static Library
│   └── .a
└── Dynamic Library
    └── .dylib / dynamic framework binary
```

---

# 8. Static vs Dynamic Libraries

This is one of the most important distinctions.

---

## 8.1 Static Library

A static library is often:

```text
libPaymentSDK.a
```

It is essentially an archive:

```text
libPaymentSDK.a
├── Payment.o
├── Network.o
├── Crypto.o
└── Models.o
```

During linking, required code is copied into the final executable.

```text
MyApp.o
+
libPaymentSDK.a
      │
      ▼
    Linker
      │
      ▼
MyApp Mach-O
├── App code
└── PaymentSDK code
```

At runtime, there is no separate PaymentSDK image for dyld to load.

---

## 8.2 Dynamic Library

With dynamic linking:

```text
MyApp
+
PaymentSDK.framework
```

the app stores a runtime dependency instead of copying all SDK code into the app executable.

Conceptually:

```text
MyApp Mach-O
└── LC_LOAD_DYLIB
    @rpath/PaymentSDK.framework/PaymentSDK
```

At runtime:

```text
dyld loads:

MyApp
+
PaymentSDK
```

---

## 8.3 Static vs Dynamic Comparison

| Area | Static | Dynamic |
|---|---|---|
| Linking | Build time | Build + runtime resolution |
| SDK code location | Copied into consumer binary | Separate Mach-O image |
| dyld work | Less | More |
| Dead stripping | Often strong | More limited at library boundary |
| Executable size | Larger | Smaller |
| App bundle size | Depends | Framework included separately |
| Shared code across executables | Can duplicate | Can sometimes reduce duplication |
| Launch overhead | Usually lower | Usually higher |
| Runtime modularity | Lower | Higher |

Important:

> Static is not always better. Dynamic is not always worse. Measure based on architecture.

---

# 9. Frameworks

A framework is a specialized Apple bundle for packaging reusable code.

Example:

```text
PaymentSDK.framework/
├── PaymentSDK
├── Info.plist
├── Modules/
├── Headers/
├── PrivacyInfo.xcprivacy
└── Resources/
```

The file:

```text
PaymentSDK.framework/PaymentSDK
```

is the actual compiled binary.

The outer:

```text
PaymentSDK.framework
```

is the package.

---

## 9.1 A Framework Can Be Static or Dynamic

This is critical:

```text
.framework ≠ dynamic
```

You can have:

```text
Static Framework
Dynamic Framework
```

So:

```text
Framework = packaging concept
Static/Dynamic = linking concept
```

---

## 9.2 Static Framework

A static framework is:

```text
Framework packaging
+
static linking behavior
```

The code is merged into the consumer binary at link time.

---

## 9.3 Dynamic Framework

A dynamic framework remains separate.

Example app bundle:

```text
MyApp.app/
├── MyApp
└── Frameworks/
    └── PaymentSDK.framework/
        └── PaymentSDK
```

dyld loads it at runtime.

---

# 10. Framework vs Library

The simplest distinction:

> A library is primarily reusable compiled code.  
> A framework is structured packaging around reusable code plus metadata and resources.

| Aspect | Library | Framework |
|---|---|---|
| Main role | Reusable compiled code | Reusable code + structured package |
| Common form | `.a`, `.dylib` | `.framework` |
| Resources | Usually separate | Can be included |
| Headers | Often external | Can be included |
| Swift modules | Separate handling | Included |
| Info.plist | No | Yes |
| Privacy manifest | Awkward with raw `.a` | Natural |
| Static | Yes | Yes |
| Dynamic | Yes | Yes |
| SDK distribution | Lower-level | Better suited |

A raw static SDK might need:

```text
libPaymentSDK.a
Headers/
Resources.bundle
PrivacyInfo.xcprivacy
```

A framework can package these more naturally.

---

# 11. Bundles

A **bundle** is a general Apple packaging concept.

A bundle is a directory with a defined structure that Apple APIs understand as one logical unit.

Examples:

```text
MyApp.app
PaymentSDK.framework
Widget.appex
Resources.bundle
```

All are bundles.

---

## 11.1 Typical Bundle Types

```text
Bundle
├── Application Bundle     (.app)
├── Framework Bundle       (.framework)
├── Extension Bundle       (.appex)
├── Resource Bundle        (.bundle)
└── Plug-in Bundle
```

---

## 11.2 `.bundle` vs `Bundle`

These are different concepts.

### `.bundle`

A filesystem/package type:

```text
PaymentResources.bundle/
├── Info.plist
├── icon.png
└── Localizable.strings
```

### `Bundle`

Foundation API:

```swift
let main = Bundle.main
```

`Bundle.main` normally represents:

```text
MyApp.app
```

not specifically a `.bundle` file.

---

# 12. Framework vs Bundle

Yes:

> A framework is a specialized type of bundle.

Therefore:

```text
Every framework is a bundle.
Not every bundle is a framework.
```

Think of it like:

```text
Bundle
│
├── .app
│
├── .framework
│
├── .appex
└── .bundle
```

A framework is roughly:

```text
Framework
=
Bundle
+
Library binary
+
Module metadata
+
Optional headers
+
Optional resources
```

---

# 13. Mach-O

Mach-O = Mach Object.

It is Apple's executable binary format.

Mach-O can represent:

- executable
- dynamic library
- object file
- bundle executable

Simplified:

```text
Mach-O
├── Header
├── Load Commands
├── __TEXT
├── __DATA
├── __LINKEDIT
└── Code Signature
```

---

## 13.1 Mach-O Header

Contains information such as:

- CPU type
- file type
- number of load commands
- flags

Typical file types:

```text
MH_EXECUTE
MH_DYLIB
MH_BUNDLE
```

---

## 13.2 Load Commands

Examples:

```text
LC_SEGMENT_64
LC_LOAD_DYLIB
LC_RPATH
LC_MAIN
LC_UUID
LC_BUILD_VERSION
LC_CODE_SIGNATURE
```

For a dynamic framework dependency:

```text
LC_LOAD_DYLIB
@rpath/PaymentSDK.framework/PaymentSDK
```

---

## 13.3 `__TEXT`

Usually contains read-only executable data.

Examples:

```text
__TEXT
├── __text        executable machine instructions
├── __cstring     C strings
├── __const       constants
└── __swift5_*    Swift metadata
```

---

## 13.4 `__DATA`

Usually writable runtime data.

Examples:

- globals
- static variables
- runtime pointers
- lazy initialization state

---

# 14. XCFramework

An XCFramework is **not a new linking model**.

It is a distribution container for multiple platform/architecture variants.

Example:

```text
PaymentSDK.xcframework/
├── Info.plist
├── ios-arm64/
│   └── PaymentSDK.framework
├── ios-arm64_x86_64-simulator/
│   └── PaymentSDK.framework
└── macos-arm64_x86_64/
    └── PaymentSDK.framework
```

Xcode selects the appropriate variant.

---

## 14.1 Device vs Simulator

Even if both use ARM64:

```text
iOS device arm64
iOS Simulator arm64
```

they are not interchangeable.

Platform matters in addition to architecture.

XCFramework keeps those variants separate.

---

## 14.2 Creating an XCFramework

Typical workflow:

```bash
xcodebuild archive \
  -scheme PaymentSDK \
  -destination "generic/platform=iOS" \
  -archivePath ./build/PaymentSDK-iOS \
  SKIP_INSTALL=NO \
  BUILD_LIBRARY_FOR_DISTRIBUTION=YES
```

Simulator:

```bash
xcodebuild archive \
  -scheme PaymentSDK \
  -destination "generic/platform=iOS Simulator" \
  -archivePath ./build/PaymentSDK-Simulator \
  SKIP_INSTALL=NO \
  BUILD_LIBRARY_FOR_DISTRIBUTION=YES
```

Create:

```bash
xcodebuild -create-xcframework \
-framework ./build/PaymentSDK-iOS.xcarchive/Products/Library/Frameworks/PaymentSDK.framework \
-framework ./build/PaymentSDK-Simulator.xcarchive/Products/Library/Frameworks/PaymentSDK.framework \
-output PaymentSDK.xcframework
```

---

# 15. Swift Package Manager

Swift Package Manager is primarily:

- dependency management
- build configuration
- source distribution
- binary distribution

Source package:

```swift
.library(
    name: "PaymentSDK",
    targets: ["PaymentSDK"]
)
```

Binary distribution:

```swift
.binaryTarget(
    name: "PaymentSDK",
    url: "https://example.com/PaymentSDK.xcframework.zip",
    checksum: "..."
)
```

A package may contain:

```text
source
or
binary XCFramework
```

---

# 16. Embedding

Linking and embedding are different.

## Linking

Makes symbols available to the executable.

## Embedding

Copies a runtime dependency into the final app bundle.

Dynamic framework:

```text
Link
+
Embed
+
Sign
```

Static framework:

```text
Link
```

because its code already became part of the application binary.

---

# 17. Runtime Paths: `@rpath`, `@loader_path`, `@executable_path`

Dynamic frameworks must be discoverable at runtime.

Instead of storing a hardcoded path:

```text
/private/var/.../PaymentSDK.framework
```

the app can record:

```text
@rpath/PaymentSDK.framework/PaymentSDK
```

Typical runtime search path:

```text
@executable_path/Frameworks
```

For an app:

```text
@executable_path
=
MyApp.app
```

Therefore:

```text
@executable_path/Frameworks
=
MyApp.app/Frameworks
```

---

## 17.1 `@loader_path`

`@loader_path` refers to the directory containing the Mach-O image currently doing the loading.

Useful for nested framework dependency relationships.

---

# 18. Code Signing

Code signing gives two major guarantees:

```text
Identity
+
Integrity
```

Conceptually:

```text
Executable pages
      │
      ▼
   Hashes
      │
      ▼
 CodeDirectory
      │
      ▼
 Signed with private key
      │
      ▼
 Digital signature
```

If executable code changes after signing:

```text
expected hash ≠ actual hash
```

verification fails.

---

## 18.1 What Is Protected?

Code signing may cryptographically bind:

- executable code
- bundle identifier
- entitlements
- CodeDirectory
- signer/team identity
- nested executable content

---

# 19. Certificates, Keys, Provisioning Profiles & Entitlements

These are related but not the same thing.

---

## 19.1 Certificate

A certificate effectively says:

```text
Public key X belongs to Developer/Team Y
```

---

## 19.2 Private Key

Stored in Keychain.

Used to create signatures.

```text
Private key → sign
Public key  → verify
```

---

## 19.3 Provisioning Profile

Connects authorization/configuration details such as:

- Team ID
- App ID
- signing certificate
- enabled entitlements
- device IDs in some distribution modes

---

## 19.4 Entitlements

Entitlements describe privileged capabilities.

Examples:

```text
Push Notifications
App Groups
Associated Domains
Keychain Groups
iCloud
```

Example:

```xml
<key>com.apple.security.application-groups</key>
<array>
    <string>group.com.company.bank</string>
</array>
```

Because entitlements are tied into signing, they cannot simply be modified after signing.

---

# 20. What Happens When an iOS App Launches

High level:

```text
User taps app
    │
    ▼
iOS/kernel creates process
    │
    ▼
Mach-O header is inspected
    │
    ▼
Segments are mapped into virtual memory
    │
    ▼
dyld loads dependencies
    │
    ▼
Symbols/fixups are resolved
    │
    ▼
Swift runtime initializes
    │
    ▼
Objective-C runtime initializes
    │
    ▼
Initializers run
    │
    ▼
main / @main
    │
    ▼
UIApplication / SwiftUI lifecycle
    │
    ▼
Your code executes
```

---

# 21. dyld

`dyld` = dynamic loader.

Its responsibilities include:

- finding dependent dynamic libraries
- mapping Mach-O images
- resolving runtime paths
- applying fixups
- binding symbols
- participating in runtime initialization

Suppose the app has:

```text
LC_LOAD_DYLIB
@rpath/PaymentSDK.framework/PaymentSDK
```

dyld locates:

```text
MyApp.app/Frameworks/PaymentSDK.framework/PaymentSDK
```

and maps it into the process address space.

---

## 21.1 Virtual Memory Mapping

The entire binary does not necessarily get copied into RAM immediately.

Conceptually:

```text
Process Virtual Address Space

0x100000000  MyApp __TEXT
0x100800000  MyApp __DATA
0x101000000  PaymentSDK __TEXT
0x101600000  PaymentSDK __DATA
...
```

Pages are mapped and may be faulted in on demand.

---

## 21.2 ASLR

ASLR = Address Space Layout Randomization.

The same framework can load at different addresses on different launches.

Example:

```text
Launch 1:
PaymentSDK → 0x105000000

Launch 2:
PaymentSDK → 0x10A000000
```

Runtime relocation/fixup machinery makes this possible.

---

## 21.3 System Frameworks

UIKit, Foundation, SwiftUI, etc. are provided by the OS.

Many system libraries participate in Apple's shared-cache infrastructure.

Therefore:

```text
UIKit.framework
```

is not equivalent to an app-embedded third-party dynamic framework from a launch-performance point of view.

---

# 22. Swift & Objective-C Runtime Initialization

Before your app starts normal business logic, runtime systems initialize metadata.

---

## 22.1 Swift Runtime

Supports concepts such as:

- protocol conformances
- type metadata
- generic metadata
- ARC
- reflection
- dynamic casts
- async tasks
- actors

Example:

```swift
struct Box<T> {
    let value: T
}
```

Concrete generic metadata may be produced/managed by the runtime.

---

## 22.2 Objective-C Runtime

Handles:

- classes
- metaclasses
- selectors
- method lists
- categories
- protocols
- dynamic dispatch

Objective-C message dispatch commonly involves:

```text
objc_msgSend
```

---

## 22.3 Global and Static Initialization

Avoid expensive launch-time work in:

- Objective-C `+load`
- C/C++ global constructors
- heavyweight static initialization
- eager SDK startup
- synchronous file/database/network work

---

# 23. Static vs Dynamic: Performance Trade-offs

There is no universal winner.

---

## 23.1 Static Advantages

Potential benefits:

- fewer runtime-loaded images
- lower dyld overhead
- strong dead stripping
- simpler runtime dependency graph

---

## 23.2 Static Disadvantages

If the same static library is used by:

```text
Main App
Widget
Notification Extension
```

the code can be duplicated into each executable.

Example:

```text
MyApp executable
    + CommonSDK code

Widget executable
    + CommonSDK code

NotificationService executable
    + CommonSDK code
```

This can increase installed/package size.

---

## 23.3 Dynamic Advantages

Potential benefits:

- distinct runtime module
- can reduce some duplicated code
- can support certain modular architectures
- useful when runtime separability matters

---

## 23.4 Dynamic Disadvantages

More dynamic frameworks mean more work for the dynamic loader.

Potential startup work:

- mapping
- dependency discovery
- fixups
- binding
- initializers
- metadata processing

Therefore:

```text
50 source modules
≠
50 dynamic frameworks
```

You can have strong modularity while still statically linking many modules.

---

## 23.5 Launch Performance Priorities

Investigate:

1. excessive dynamic framework count
2. expensive static/global initialization
3. Objective-C `+load`
4. synchronous disk I/O
5. database startup
6. image decoding
7. dependency initialization
8. huge metadata graphs
9. main-thread blocking
10. expensive SDK `configure()`

---

# 24. Real-World SDK Architecture

Imagine a closed-source payment SDK.

Requirements:

- Swift
- iOS 16+
- binary distribution
- resources
- localization
- privacy manifest
- SwiftPM support
- simulator support

Recommended shape:

```text
PaymentSDK Swift Package
│
└── binaryTarget
      │
      ▼
PaymentSDK.xcframework
│
├── ios-arm64/
│   └── PaymentSDK.framework
└── ios-arm64_x86_64-simulator/
    └── PaymentSDK.framework
```

Framework:

```text
PaymentSDK.framework
├── PaymentSDK
├── Info.plist
├── Modules/
├── PrivacyInfo.xcprivacy
├── Assets.car
└── Resources/
```

---

## 24.1 API Surface

Public:

```text
PaymentClient
PaymentConfiguration
PaymentRequest
PaymentResult
PaymentError
```

Internal:

```text
HTTPClient
Authentication
Storage
RetryEngine
Telemetry
Crypto
```

---

## 24.2 Example API

```swift
public protocol PaymentClient: Sendable {
    func authorize(
        _ request: PaymentRequest
    ) async throws -> PaymentResult
}
```

Internal:

```swift
internal actor DefaultPaymentClient: PaymentClient {
    private let transport: NetworkTransport

    init(transport: NetworkTransport) {
        self.transport = transport
    }

    func authorize(
        _ request: PaymentRequest
    ) async throws -> PaymentResult {
        // SDK implementation
        fatalError("Example")
    }
}
```

---

# 25. SDK API Design Best Practices

## 25.1 Keep the Public Surface Small

Prefer:

```swift
public struct PaymentRequest { }
public enum PaymentError: Error { }
public protocol PaymentClient { }
```

Keep implementation details:

```swift
internal
private
```

Rule:

> Internal by default. Public intentionally.

---

## 25.2 Avoid Exposing Third-Party Types

Bad:

```swift
public func configure(
    session: Alamofire.Session
)
```

This makes Alamofire part of your API contract.

Better:

```swift
public protocol NetworkTransport: Sendable {
    func send(
        _ request: SDKRequest
    ) async throws -> SDKResponse
}
```

---

## 25.3 Avoid Unnecessary `open`

```swift
public class PaymentProcessor
```

is a smaller compatibility commitment than:

```swift
open class PaymentProcessor
```

Use `open` only when consumer subclassing/overriding is intentionally supported.

---

## 25.4 Use Third-Party Dependencies Carefully

Every dependency may introduce:

- version conflicts
- binary size
- privacy implications
- security patching obligations
- transitive dependencies
- symbol collisions
- build-time cost

Prefer system APIs where practical.

Example:

```swift
URLSession
```

may be preferable to forcing a networking framework on every SDK consumer.

---

# 26. Resource Packaging Best Practices

Never assume host-app resource lookup.

Risky:

```swift
UIImage(named: "checkmark")
```

The host app may also contain:

```text
checkmark.png
```

Prefer SDK-specific bundle lookup.

For SwiftPM:

```swift
Bundle.module
```

Also namespace:

```text
PaymentSDK_success
PaymentSDK_card_visa
PaymentSDK_default_config
```

Do the same for:

- UserDefaults keys
- Keychain service names
- NotificationCenter names
- database names
- logging categories
- URL schemes

Rule:

> An SDK lives inside someone else's application. Behave like a guest.

---

# 27. Binary Compatibility & Library Evolution

Important concepts:

```text
ABI Stability
Module Stability
Library Evolution
```

---

## 27.1 ABI Stability

ABI = Application Binary Interface.

Concerns:

- calling conventions
- runtime layout contracts
- symbol interaction
- binary compatibility

Swift achieved ABI stability on Apple platforms starting with Swift 5.

---

## 27.2 Module Stability

Answers:

> Can a module built with one Swift compiler version be imported by another compatible compiler version?

For binary distribution:

```text
BUILD_LIBRARY_FOR_DISTRIBUTION = YES
```

commonly produces textual interfaces such as:

```text
.swiftinterface
```

---

## 27.3 `@frozen`

Example:

```swift
@frozen
public enum PaymentStatus {
    case pending
    case success
    case failed
}
```

This makes a stronger binary-layout/API evolution promise.

Do not use casually.

---

## 27.4 `@inlinable`

Example:

```swift
@inlinable
public func calculateFee(...) -> Decimal {
    ...
}
```

Allows cross-module optimization opportunities but exposes implementation constraints.

Use only when there is a measured reason.

---

## 27.5 `@usableFromInline`

Lets `@inlinable` code reference internal symbols with special ABI visibility.

Advanced SDK feature; not a general optimization switch.

---

# 28. Debugging & Inspection Tools

Become comfortable with these.

---

## 28.1 `file`

Inspect binary type and architecture.

```bash
file PaymentSDK
```

Example:

```text
Mach-O 64-bit dynamically linked shared library arm64
```

---

## 28.2 `otool -L`

Inspect dynamic dependencies.

```bash
otool -L MyApp
```

May show:

```text
@rpath/PaymentSDK.framework/PaymentSDK
/System/Library/Frameworks/UIKit.framework/UIKit
```

A statically linked SDK normally does not appear as its own runtime dynamic dependency.

---

## 28.3 `otool -l`

Inspect Mach-O load commands.

```bash
otool -l MyApp
```

Useful for:

- `LC_RPATH`
- `LC_LOAD_DYLIB`
- segment information

---

## 28.4 `nm`

Inspect symbols.

```bash
nm -gU PaymentSDK
```

For Swift symbols:

```bash
nm PaymentSDK | swift-demangle
```

---

## 28.5 `size`

Inspect binary section sizes.

```bash
size PaymentSDK
```

---

## 28.6 `codesign`

Inspect signature:

```bash
codesign -dvvv PaymentSDK.framework
```

Inspect entitlements:

```bash
codesign -d --entitlements :- MyApp.app
```

---

## 28.7 `dwarfdump`

Inspect UUID/debug-symbol metadata.

```bash
dwarfdump --uuid PaymentSDK
```

---

## 28.8 `lipo`

Inspect architectures:

```bash
lipo -info PaymentSDK
```

For modern XCFramework workflows, architecture alone is not enough; platform also matters.

---

# 29. Common Problems

## 29.1 Undefined Symbols

Typical cause:

```text
object references symbol
but linker cannot find implementation
```

Possible reasons:

- missing library/framework
- wrong target membership
- wrong architecture
- wrong library order/configuration
- static Obj-C category linking issue

---

## 29.2 Duplicate Symbols

Typical cause:

```text
same symbol included more than once
```

Possible reasons:

- dependency linked twice
- static library duplication
- overlapping source targets
- aggressive linker flags

---

## 29.3 dyld "Library Not Loaded"

Typical form:

```text
Library not loaded:
@rpath/PaymentSDK.framework/PaymentSDK
```

Possible reasons:

- framework not embedded
- bad runtime search path
- wrong framework variant
- missing signature
- corrupted bundle structure

---

## 29.4 Wrong Architecture

Example:

```text
building iOS app for arm64
but SDK contains only simulator binary
```

Use XCFramework variants correctly.

---

## 29.5 Obj-C Static Library Categories

Historical issue:

Static libraries containing Objective-C categories may require linker flags such as:

```text
-ObjC
```

Avoid blindly using:

```text
-all_load
```

because it can:

- increase binary size
- trigger duplicate symbols
- increase link work

---

# 30. Decision Guide

| Situation | Strong candidate |
|---|---|
| Small internal reusable module | Static |
| General-purpose binary SDK | Static framework often a strong default |
| Many small app modules | Static or merged |
| Launch-sensitive app | Minimize dynamic images |
| Shared code across app + several extensions | Measure static duplication vs dynamic |
| Plugin-style runtime modularity | Dynamic |
| System framework | Dynamic |
| Need resources with static linking | Static framework |
| Raw C library | `.a` may be enough |
| Multi-platform binary SDK | XCFramework |
| Closed-source Swift SDK | XCFramework + SwiftPM binary target |
| Open-source Swift library | Source Swift Package often simplest |

Always measure:

```text
launch time
download size
installed size
memory
clean build time
incremental build time
integration complexity
```

---

# 31. Interview-Ready Answers

## 31.1 Framework vs Library

> A library is primarily reusable compiled code, while a framework is an Apple bundle that packages reusable code together with module metadata, headers, resources, and configuration. A framework can itself be statically or dynamically linked, so framework vs library describes packaging, while static vs dynamic describes linking behavior.

---

## 31.2 Framework vs Bundle

> A framework is a specialized type of bundle. Bundle is the broader Apple packaging concept used by `.app`, `.framework`, `.appex`, and `.bundle`. Every framework is a bundle, but not every bundle is a framework.

---

## 31.3 Static vs Dynamic Library

> A static library is linked into the consuming executable at build time, so its required machine code becomes part of the final binary. A dynamic library remains a separate Mach-O image; the executable records a dependency on it, and dyld maps and binds it at runtime.

---

## 31.4 Framework vs XCFramework

> A framework packages one library/module variant, while an XCFramework is a distribution container that packages multiple platform and architecture variants of frameworks or libraries. XCFramework does not introduce a new linking mechanism.

---

## 31.5 Linking vs Embedding

> Linking makes symbols available to an executable. Embedding copies runtime dynamic dependencies into the final application bundle. Static libraries need linking; dynamic frameworks generally need both linking and embedding.

---

## 31.6 What dyld Does

> dyld is Apple's dynamic loader. At launch it reads Mach-O dependency information, locates required dynamic libraries, maps them into the process address space, applies fixups/bindings, and participates in runtime initialization before application execution reaches normal app lifecycle code.

---

## 31.7 What Code Signing Does

> Code signing cryptographically binds executable content, identity, and entitlements so iOS can verify who signed the code and detect whether protected executable content changed after signing.

---

# 32. Mental Model Summary

The most important thing is to separate **different dimensions**:

```text
Source Language
    Swift / Obj-C / C / C++

Compiler Output
    .o object files

Reusable Code
    .a / .dylib

Binary Format
    Mach-O

Packaging
    .framework / .app / .appex / .bundle

Multi-Platform Distribution
    .xcframework

Dependency Management
    Swift Package Manager

Security
    Code Signing

Runtime Loading
    dyld

Developer Product
    SDK
```

Or in one compact view:

```text
SOURCE
│
├── Swift
├── Objective-C
└── C/C++
      │
      ▼
COMPILER
      │
      ▼
OBJECT FILES (.o)
      │
      ├──────────────┐
      │              │
      ▼              ▼
STATIC LIBRARY     OTHER LIBRARIES
    (.a)           / FRAMEWORKS
      │              │
      └──────┬───────┘
             ▼
           LINKER
             │
             ▼
          MACH-O
             │
      ┌──────┴────────┐
      │               │
      ▼               ▼
 EXECUTABLE       DYNAMIC LIBRARY
      │               │
      └──────┬────────┘
             ▼
          FRAMEWORK
             │
             ▼
        XCFRAMEWORK
             │
             ▼
      LINK / EMBED / SIGN
             │
             ▼
          APP BUNDLE
             │
             ▼
            iOS
             │
             ▼
            dyld
             │
             ▼
   Swift / Obj-C Runtime
             │
             ▼
        main / @main
             │
             ▼
          EXECUTION
```

---

# Final Rules of Thumb

1. **Framework is packaging, not a synonym for dynamic linking.**
2. **A framework is a specialized bundle.**
3. **A static library is usually an archive of object files.**
4. **A dynamic framework remains a separate Mach-O image at runtime.**
5. **XCFramework is for distribution across platforms/architectures.**
6. **Linking and embedding are different operations.**
7. **dyld matters only for runtime-loaded dynamic images.**
8. **Code signing protects identity, integrity, and signed entitlements.**
9. **Fewer dynamic images can help launch performance, but measure before deciding.**
10. **For SDK design, public API stability is usually more important than packaging details.**
11. **Resources must be namespaced and loaded from the correct bundle.**
12. **Avoid leaking third-party dependency types through the SDK public API.**
13. **Use `BUILD_LIBRARY_FOR_DISTRIBUTION=YES` when shipping stable binary Swift SDKs where library evolution/module stability is required.**
14. **Use tools like `otool`, `nm`, `codesign`, `file`, and `dwarfdump` to understand what Xcode actually built.**

---

## Recommended Hands-On Exercise

Create one Xcode workspace containing:

```text
DemoApp
StaticSDK
DynamicSDK
```

Then inspect the generated `.app`.

Run:

```bash
file DemoApp
otool -L DemoApp
otool -l DemoApp
nm DemoApp | swift-demangle
codesign -dvvv DemoApp.app
codesign -d --entitlements :- DemoApp.app
```

Compare:

```text
StaticSDK
```

with:

```text
DynamicSDK
```

You should observe:

```text
StaticSDK:
code becomes part of DemoApp executable
no independent runtime dylib dependency

DynamicSDK:
appears in otool -L
exists under DemoApp.app/Frameworks
loaded by dyld at runtime
```

That experiment makes the entire compilation → linking → packaging → signing → loading model concrete.
