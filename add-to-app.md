# Flutter Add-to-App Architecture — Complete Technical Guide

## 1. What Flutter Add-to-App Actually Means

**Flutter Add-to-App** means integrating Flutter as a module/framework inside an existing native Android or iOS application instead of making Flutter the application's main entry point.

The existing application remains the host:

Flutter officially supports integrating Flutter incrementally into existing applications. On mobile, Flutter supports multiple Flutter instances using a **multi-engine** architecture. Each engine runs an independent Dart program and maintains its own UI and application state.

The main architectural idea is:

```text
Native Application
        ↕
Integration Boundary
        ↕
Flutter Runtime
```

Flutter can access native functionality through mechanisms such as:

```text
MethodChannel
EventChannel
BasicMessageChannel
Pigeon-generated APIs
Flutter Plugins
```

Communication does **not automatically synchronize state between Flutter and native**.

---

# 2. Add-to-App Is More Than Opening Flutter Screens

A common misunderstanding is:

```text
Add-to-App = Native App opens Flutter Screen
```

Technically, opening the screen is one of the easier parts.

The actual architecture involves:

```text
Navigation
State Synchronization
Local Storage
Communication
Flutter Engine Lifecycle
Dependency Injection
Authentication
Plugin Ownership
Memory
Debugging
Security
CI/CD
Release Management
Migration Strategy
```

Two of the most complex parts are generally:

```text
1. Navigation
2. Local Storage / State Synchronization
```

because these are where ownership frequently crosses the Flutter/native boundary.

---

# 3. Core Add-to-App Architecture

A production architecture can look like:

```text
┌─────────────────────────────────────────────┐
│                 NATIVE APP                  │
│                                             │
│ Native UI                                   │
│ Native Navigation                           │
│ Native Services                             │
│ Native Storage                              │
│ Native Authentication                       │
│ Native Dependency Injection                 │
│                                             │
├─────────────────────────────────────────────┤
│              INTEGRATION LAYER              │
│                                             │
│ FlutterEngine Manager                       │
│ NativeBridge                                │
│ Navigation Contract                         │
│ State Contract                              │
│ MethodChannel / Pigeon                      │
│ Lifecycle Coordination                      │
│                                             │
├─────────────────────────────────────────────┤
│                  FLUTTER                    │
│                                             │
│ Flutter Core                                │
│ Flutter Navigation                          │
│ Flutter State Management                    │
│ Flutter Networking                          │
│ Flutter Storage                             │
│ Flutter Dependency Injection                │
│ Flutter Features                            │
│                                             │
└─────────────────────────────────────────────┘
```

The **Integration Layer** is important because it prevents native implementation details from leaking throughout Flutter and Flutter implementation details from leaking throughout native code.

---

# 4. Flutter Module

you can create a Flutter module:

```bash
flutter create --template module my_flutter
```

Conceptually:

```text
Application/
│
├── NativeApp/
│   ├── Android / iOS code
│   ├── Native Navigation
│   ├── Native Services
│   └── Native DI
│
└── FlutterModule/
    ├── lib/
    ├── pubspec.yaml
    ├── Flutter Core
    └── Flutter Features
```

Flutter officially supports integrating modules into Android and iOS host applications.

On Android, Flutter supports integration directly with the Gradle build or through an Android Archive (`AAR`).

On iOS, Flutter supports integration through mechanisms including generated artifacts/build phases and Swift package integration depending on the chosen setup.

---

# 5. FlutterEngine

A Flutter Engine hosts a Flutter execution environment.

Conceptually:

```text
FlutterEngine
│
├── Dart execution
├── Dart isolate
├── Flutter framework
├── Plugin registry
├── Platform channels
├── Rendering pipeline
├── Flutter state
└── Flutter application
```

The UI container is separate from the engine.

Android:

```text
FlutterActivity or FlutterFragment
       │
       └── FlutterEngine
```

iOS:

```text
FlutterViewController
       │
       └── FlutterEngine
```

Flutter's APIs allow the engine to exist independently of the UI container, which is important when pre-warming or caching an engine.

---

# 7. Flutter Engine Lifecycle

There are multiple strategies.

## Strategy A — Create Engine When Needed

```text
Native
  ↓
User opens Flutter
  ↓
Create FlutterEngine
  ↓
Start Dart
  ↓
Initialize Flutter
  ↓
Show Flutter UI
```

When finished:

```text
Close Flutter
    ↓
Destroy FlutterEngine
```

Advantages:

```text
Simple lifecycle
Clean state
Lower persistent memory usage
```

Disadvantages:

```text
Engine startup latency
Dart startup
Plugin initialization
Flutter application bootstrap
DI initialization
```

For frequently accessed Flutter features, repeated initialization might not be desirable.

---

# 8. Pre-Warmed / Cached Flutter Engine

Another architecture is:

```text
Native Application Starts
          ↓
Create FlutterEngine
          ↓
Start Dart
          ↓
Initialize Flutter
          ↓
Cache FlutterEngine
          ↓
Continue Native Application
```

Later:

```text
User opens Flutter Feature
          ↓
Reuse FlutterEngine
          ↓
Attach FlutterActivity /
FlutterViewController
```

Android conceptually:

```kotlin
val flutterEngine = FlutterEngine(context)

flutterEngine.dartExecutor.executeDartEntrypoint(
    DartExecutor.DartEntrypoint.createDefault()
)

FlutterEngineCache
    .getInstance()
    .put("main_flutter_engine", flutterEngine)
```

Then:

```kotlin
FlutterActivity
    .withCachedEngine("main_flutter_engine")
    .build(context)
```

The important difference is:

```text
FlutterEngine lifetime
         ≠
Flutter screen lifetime
```

---

# 10. Flutter Add-to-App Modes

Flutter's current architecture distinguishes between two major Add-to-App modes.

For mobile: → Multi-engine

On Android/iOS, each Flutter instance is backed by a separate Flutter Engine and therefore runs its own Dart program/isolate state.

For web: → Multi-view

On web, multiple views can share the same Dart program and therefore share Dart objects.

Mobile Add-to-App currently does not support multi-view in the same way web does.

---

# 12. Navigation — The First Major Challenge

With Add-to-App, you effectively have:

```text
Native Navigation
        +
Flutter Navigation
```

These are separate navigation systems.

Three common flows are particularly important:

```text
1. Native → Native

2. Native → Flutter → Flutter

3. Native → Flutter → Native
```

A larger hybrid application can eventually contain:

```text
Native
  ↓
Flutter
  ↓
Native
  ↓
Flutter
  ↓
Native
```

This is where navigation architecture becomes critical.

---

# 14. Case 2 — Native → Flutter → Continue in Flutter

Example:

```text
Native Dashboard
        ↓
Flutter Orders
        ↓
Flutter Order Details
        ↓
Flutter Order History
```

The native application needs to create or reuse a Flutter Engine.

Then it presents the Flutter UI container.

On iOS:

```text
Native UIViewController
        ↓
FlutterViewController
        ↓
Flutter Navigator
```

Once Flutter has been entered:

```text
Flutter Screen A
       ↓
Flutter Router
       ↓
Flutter Screen B
       ↓
Flutter Router
       ↓
Flutter Screen C
```

Flutter should usually handle these internal transitions itself.

You should avoid sending every Flutter page transition back through native.

Bad architecture:

```text
Flutter Screen A
       ↓
Native
       ↓
Tell Flutter to open B
       ↓
Flutter B
```

Preferred architecture:

```text
Native
   ↓
Open Flutter Feature
   ↓
Flutter Navigator
   ↓
Flutter A
   ↓
Flutter B
   ↓
Flutter C
```

Native owns the boundary.

Flutter owns navigation inside its feature/application area.

---

# 15. Case 3 — Native → Flutter → Native

This flow is considerably more complicated:

```text
Native Dashboard
       ↓
Flutter Orders
       ↓
Flutter Order Details
       ↓
Native Payment
```

Conceptually:

```text
Native Screen
      ↓
FlutterViewController
      ↓
Flutter Screen
      ↓
MethodChannel / Pigeon
      ↓
Native Integration Layer
      ↓
Native Navigation Controller
      ↓
Native Screen
```

Flutter does not directly execute something like:

```text
navigationController.pushViewController(...)
```

because that belongs to the native side.

Instead Flutter sends an intent:

```text
Open native payment screen
Order ID = 123
```

Native receives this request and performs the actual navigation.

---

---

# 17. Navigation Ownership

A strong architecture has explicit ownership.

```text
Native Root Navigation
        ↓
Native

Flutter Internal Navigation
        ↓
Flutter

Flutter → Native Request
        ↓
Integration Layer

Native → Flutter Destination
        ↓
Integration Layer
```

Flutter should communicate **intent**:

```text
openNativeDestination(
    payment(orderId: 123)
)
```

rather than native implementation details:

```text
Get UINavigationController
Push PaymentViewController
```

Flutter should not know those details.

---

# 19. Native → Flutter Destination Contract

The opposite direction should also be structured.

Instead of:

```text
openOrders()
openInvoice()
openCustomer()
```

you could send:

```text
FlutterDestination
```

Example:

```json
{
  "feature": "invoice",
  "action": "details",
  "parameters": {
    "invoiceId": 1005
  }
}
```

Then:

```text
Native
   ↓
FlutterHost
   ↓
FlutterDestination
   ↓
Flutter Router
   ↓
InvoiceDetails(1005)
```

This creates a stable cross-platform navigation API.

---

# 20. Cached Engine Navigation Problem

A cached engine preserves Flutter state.

Imagine Flutter has:

```text
Home
 ↓
Orders
 ↓
OrderDetails
```

Then Flutter UI closes, but the engine remains alive.

Later native requests:

```text
Reports
```

Flutter may still have:

```text
Home
 ↓
Orders
 ↓
OrderDetails
```

inside its navigator.

Therefore the navigation command should contain a mode:

```dart
enum FlutterNavigationMode {
  push,
  replace,
  reset,
  resume,
}
```

Example:

```text
openFlutter(
    destination: Reports,
    mode: reset
)
```

Meaning:

```text
Ignore previous navigation stack
        ↓
Reset
        ↓
Open Reports
```

Another case might use:

```text
resume
```

to intentionally keep the previous Flutter navigation stack.

---

# 21. Multiple Flutter Instances

A single cached Flutter Engine is not always the correct architecture.

Consider:

```text
Native A
   ↓
Flutter A
   ↓
Native B
   ↓
Flutter B
```

You could potentially reuse one engine, but doing so means Flutter must carefully maintain and restore a shared internal navigation stack.

Another approach is:

```text
Flutter Instance A
        +
Flutter Instance B
```

Each instance is independent.

Flutter specifically identifies this architecture for cases where Flutter is not a leaf node in the navigation graph and the stack may contain a mixture such as:

```text
Native
  ↓
Flutter
  ↓
Native
  ↓
Flutter
```

Each Flutter instance can maintain its own:

```text
Navigation Stack
Application State
UI State
Dart execution
```

independently.

---

# 23. FlutterEngineGroup

Flutter provides:

```text
FlutterEngineGroup
```

specifically for creating multiple Flutter Engines efficiently.

Instead of:

```text
FlutterEngine()
FlutterEngine()
FlutterEngine()
```

the architecture becomes:

```text
FlutterEngineGroup
        │
    ┌───┼────┐
    │   │    │
    ▼   ▼    ▼
   E1  E2   E3
```

Each engine remains independent from an application-state point of view.

However, engines created from the same `FlutterEngineGroup` can share common engine-level resources.

Flutter documents examples including:

```text
GPU context
Font metrics
Isolate group snapshot
```

This improves memory usage and engine creation/rendering characteristics compared with completely independent engine initialization.

Flutter currently documents a low incremental memory cost for additional Flutter instances created using this architecture, although actual application memory should always be measured in your specific project and plugin set.

The first engine pays the normal initial Flutter-engine startup cost.

Later engines can benefit from shared resources.

Flutter's documentation also notes that the first created engine does not itself have to remain alive forever.

However, to preserve the shared-resource advantage, at least one engine from the group needs to remain alive.

---

# 31. Partial-Screen Flutter Views

Add-to-App is not limited to:

```text
Full Native Screen
       ↓
Full Flutter Screen
```

Another architecture is:

```text
Native Screen
│
├── Native Toolbar
│
├── Native Component
│
├── Flutter-rendered area
│
└── Native Button
```

Flutter officially recognizes partial-screen integration as an Add-to-App use case.

However, on mobile, multiple Flutter views should be understood in the context of the multi-engine architecture rather than web-style multi-view behavior.

---

# 38. Local Storage and State Synchronization

The second major Add-to-App challenge is shared state.

Technically, Flutter and native may be capable of accessing the same underlying persistent mechanisms when configured properly.

Examples include:

```text
Keychain
UserDefaults
SharedPreferences
Secure Storage
Database
Files
```

But sharing storage access does not mean state synchronization is automatically solved.

---

# 39. Shared Storage Problem

Example:

Native writes:

```text
token = A
```

Flutter reads:

```text
token = A
```

Later Flutter writes:

```text
token = B
```

Persistent storage becomes:

```text
token = B
```

but native may have:

```text
Memory Cache

token = A
```

Now:

```text
Persistent Storage
token = B

Native Memory
token = A

Flutter Memory
token = B
```

The application has inconsistent state.

---

# 40. Shared Storage Is Not Shared Runtime State

This distinction is important:

```text
Shared Persistent Storage
           ≠
Shared Runtime Memory
```

Even if both environments read the same Keychain/UserDefaults/etc., each runtime can independently cache data.

Therefore you need explicit rules.

For every shared value define:

```text
Who owns it?

Who may modify it?

Who may only read it?

What is the source of truth?

Is it cached?

When must caches be invalidated?

How does the other runtime know it changed?

Should the value be persisted?

Should updates be emitted as events?
```

---

---

# 45. Passing Initial Data to Flutter

When starting an engine, native may provide initial configuration.

Examples:

```text
Environment
Language
Tenant
Initial Feature
Initial Route
Feature Flags
User context
Build configuration
```

Conceptually:

```text
Native
   ↓
Initialize Engine
   ↓
Initial Parameters
   ↓
Flutter bootstrap()
```

But after Flutter is initialized:

```text
Initial Parameters
```

are not an ongoing state channel.

If state changes later, use:

```text
Platform Channel
```

or:

```text
Persistent source of truth + change notification
```

or another explicit synchronization mechanism.

---

# 47. Circular Dependency Risk

A dangerous architecture:

```text
Native Feature
      ↓
Flutter Feature
      ↓
Native Service
      ↓
Flutter Service
      ↓
Native Feature
```

Now:

```text
Native depends on Flutter
       ↓
Flutter depends on Native
       ↓
Circular ownership
```

This makes:

```text
Initialization
Lifecycle
Testing
Navigation
Release versioning
Dependency management
```

much harder.

Instead:

```text
Native Internals
        ↓
Integration Contract
        ↑
Flutter Internals
```

Both sides depend on the integration abstraction rather than deeply depending on each other's implementation.

---

# 49. Multiple Engines and Dependency Injection

With multiple Flutter Engines, DI needs extra attention.

If: Engine A starts Dart application A and: Engine B starts Dart application B,

then each will generally have its own Dart-level object graph.

Conceptually:

```text
Engine A
    ↓
get_it A
    ↓
Repositories A


Engine B
    ↓
get_it B
    ↓
Repositories B
```

Do not assume:

```text
get_it singleton
```

is global across every Flutter Engine.

If global application state must be shared, place ownership outside the independent Dart runtime or synchronize explicitly through native/another source of truth.

---

# 52. Plugin Limitations in Add-to-App

Flutter's mobile Add-to-App documentation notes that plugins which make assumptions incompatible with Add-to-App can behave unexpectedly.

For example, a plugin might incorrectly assume:

```text
A Flutter Activity is always available
```

which might not be true when the engine exists independently from an attached UI.

Therefore plugins used in Add-to-App should be tested for:

```text
Engine lifecycle
Activity attachment
Multiple engine behavior
Detach / reattach
Background behavior
```

Android Add-to-App currently expects AndroidX applications.

---

# 54. Flutter Add-to-App Mobile Limitation — Multiple Flutter Libraries

Flutter's current mobile documentation notes that packaging multiple independent Flutter libraries into one application is not supported.

The supported architecture should therefore be understood as:

```text
One Flutter module/library
       ↓
Potentially multiple FlutterEngine instances
```

rather than:

```text
Flutter Module A
+
Flutter Module B
+
Flutter Module C
```

packaged independently into one mobile application.

This is an important difference.

Multiple Flutters generally means:

```text
Multiple engine instances
```

not:

```text
Multiple unrelated packaged Flutter modules.
```

---

# 60. Security

The final application contains:

```text
Native Code
+
Flutter/Dart Code
```

Security therefore needs to cover both.

Consider:

```text
Dart obfuscation
Native obfuscation
Symbol management
Secure storage
Signing
API secrets
Sensitive logs
Environment values
Debug configuration
Build artifacts
```

Do not assume Flutter code is automatically secure because it is compiled.

Similarly:

```text
API secret inside Flutter
```

is not inherently protected just because it is Dart.

Sensitive secrets should normally not be embedded client-side if a backend can own them.

---

# 65. CI/CD

Add-to-App changes release architecture.

At minimum:

```text
Flutter Source
      ↓
Flutter Build
      ↓
Native Application
      ↓
Final Android/iOS Build
```

For larger teams:

```text
Flutter Repository
       ↓
Flutter CI
       ↓
Analyze
Test
Build
       ↓
Version Artifact
       ↓
Artifact Repository
       ↓
Native Repository
       ↓
Native CI
       ↓
Final Application
```

This provides reproducible integration.

---

# 72. Entry-Point Migration Risk

Moving from:

```text
Native Entry Point
```

to:

```text
Flutter Entry Point
```

can affect:

```text
Application startup
Deep links
Push notification routing
Authentication bootstrap
Analytics
Crash reporting
Navigation
Background execution
App lifecycle
CI/CD
Release rollback
```

Therefore this stage should usually happen after Flutter has already become sufficiently mature inside the application.

---

# 77. Recommended Large-Scale Architecture

For a large project:

```text
┌──────────────────────────────────────────────────┐
│                    NATIVE                        │
│                                                  │
│ App Lifecycle                                    │
│ Root Navigation                                  │
│ Existing Native Features                         │
│ Authentication / Session Owner                   │
│ Platform Services                                │
│ Existing Native Storage                          │
│ Native DI                                        │
│                                                  │
├──────────────────────────────────────────────────┤
│              ADD-TO-APP INTEGRATION              │
│                                                  │
│ FlutterEngineManager                             │
│ FlutterEngineGroup if required                   │
│ NativeBridge                                     │
│ FlutterDestination                               │
│ NativeDestination                                │
│ State / Event Contracts                          │
│ MethodChannel / Pigeon                           │
│ Trace / Logging                                  │
│ Bridge Version                                   │
│ Lifecycle Coordination                           │
│                                                  │
├──────────────────────────────────────────────────┤
│                    FLUTTER                       │
│                                                  │
│ Core                                             │
│ ├── NativeBridge                                 │
│ ├── Navigation                                   │
│ ├── Networking                                   │
│ ├── DI                                           │
│ ├── Storage                                      │
│ ├── Logging                                      │
│ ├── Analytics                                    │
│ ├── Localization                                 │
│ └── Design System                                │
│                                                  │
│ Features                                         │
│ ├── Orders                                       │
│ ├── Inventory                                    │
│ ├── Reports                                      │
│ ├── Customers                                    │
│ └── ...                                          │
│                                                  │
└──────────────────────────────────────────────────┘
```

---

# 120. Recommended Ownership Matrix

Before implementation, create something like:


| Responsibility              | Owner                     | Cross-boundary mechanism     |
| --------------------------- | ------------------------- | ---------------------------- |
| Application Lifecycle       | Native                    | Lifecycle bridge when needed |
| Root Navigation             | Native                    | Native coordinator           |
| Flutter Navigation          | Flutter                   | Flutter Router               |
| Native → Flutter Navigation | Integration               | FlutterDestination           |
| Flutter → Native Navigation | Integration               | NativeDestination            |
| Flutter Engine              | Native                    | FlutterEngineManager         |
| Multiple Engines            | Native                    | FlutterEngineGroup           |
| Authentication              | Native                    | NativeBridge                 |
| Access Token                | Native                    | NativeBridge                 |
| Refresh Token               | Native                    | NativeBridge                 |
| Selected Company            | Explicit owner            | State + event                |
| User                        | Explicit owner            | Bridge                       |
| Flutter Feature State       | Flutter                   | Bloc/Cubit                   |
| Flutter Storage             | Flutter                   | Flutter storage              |
| Native DB                   | Native                    | Native repository/bridge     |
| Flutter HTTP                | Flutter where appropriate | Dio                          |
| Native Platform API         | Native                    | Bridge                       |
| Push Notifications          | Explicit owner            | Bridge                       |
| Analytics                   | Explicit owner            | Bridge/direct                |
| Flutter DI                  | Flutter                   | get_it                       |
| Native DI                   | Native                    | Native container             |
| Logs/Tracing                | Shared contract           | Trace IDs                    |
| CI Artifact                 | Flutter CI                | Versioned artifact           |


