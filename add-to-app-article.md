# Flutter Add-to-App: From the First Screen to a Coherent Hybrid App

*How an existing native application can grow with Flutter while keeping navigation, state, engines, and releases understandable.*

Picture an established Android or iOS app. Its dashboard is native, authentication is already in place, and payment belongs to the native flow. A team adds an Orders feature in Flutter. At first, the task appears to be opening one Flutter screen. Then a customer moves from Orders to Order Details, into native payment, back to the dashboard, and later into a different Flutter feature. The architecture has to keep that journey continuous even though it crosses two runtimes.

The story below follows that journey. It begins with the boundary between native and Flutter, follows an engine as it is created and reused, tracks navigation and shared state, then reaches the build and release decisions that make the integration sustainable. Every technical example and diagram from the source guide is retained.
## 1. What Flutter Add-to-App Actually Means

The first decision is what role Flutter plays in the existing application. That definition determines who remains in control when the new feature is introduced.

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


This boundary is why Add-to-App must be planned as integration rather than treated as a screen-launch trick.

## 2. Add-to-App Is More Than Opening Flutter Screens

Once the first Flutter screen appears, the team sees that presentation was only the visible part of the work. The customer journey now crosses concerns that a standalone screen demo never exercises.

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


The next step is to give those responsibilities a place to live.

## 3. Core Add-to-App Architecture

To keep those concerns from spreading through both codebases, the host and the Flutter module need a deliberate integration boundary.

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


Once the integration layer has a purpose, the team can decide how the Flutter code enters the native build.

## 4. Flutter Module

With that boundary in mind, the team can add the Flutter module to the existing project without changing the native app into a Flutter-first app.

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


The build provides the module; the engine is what runs it when a customer reaches a Flutter feature.

## 5. FlutterEngine

The module supplies code, but the customer sees a screen only when an engine and a UI container work together. Understanding their separate roles prevents lifecycle surprises.

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


Separating engine from view makes the lifecycle choice explicit.

## 6. Flutter Engine Lifecycle

When the customer opens Orders for the first time, the host must decide when to create that engine and when to destroy it.

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


The alternative keeps the engine alive, which improves reuse while preserving state.

## 7. Pre-Warmed / Cached Flutter Engine

If Orders becomes a frequent destination, paying the full startup cost on every visit may be undesirable. A cached engine changes the experience and the state model.

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


That persistence matters even more when the app contains more than one Flutter appearance.

## 8. Flutter Add-to-App Modes

Before choosing how many engines to keep, it helps to distinguish mobile Add-to-App from the web model. The distinction determines whether Flutter instances share Dart state.

Flutter's current architecture distinguishes between two major Add-to-App modes.

For mobile: → Multi-engine

On Android/iOS, each Flutter instance is backed by a separate Flutter Engine and therefore runs its own Dart program/isolate state.

For web: → Multi-view

On web, multiple views can share the same Dart program and therefore share Dart objects.

Mobile Add-to-App currently does not support multi-view in the same way web does.


With the execution model clear, the team can design how a customer moves between screens.

## 9. Navigation — The First Major Challenge

Now follow the customer beyond the first rendered screen. Navigation is the first place where the native and Flutter worlds must agree on ownership.

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


Start with the route that enters Flutter and continues entirely inside it.

## 10. Case 2 — Native → Flutter → Continue in Flutter

The simplest cross-boundary journey enters Flutter and stays there for several pages. Native opens the feature; Flutter then guides the customer through its internal flow.

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


The next route leaves Flutter and returns to native code.

## 11. Case 3 — Native → Flutter → Native

The journey becomes more interesting when Order Details offers native payment. Flutter must hand the request back to the host without taking over native navigation.

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


These flows need a stable rule about who is allowed to perform the actual transition.

## 12. Navigation Ownership

Both directions work better when ownership is stated as a rule rather than rediscovered at every screen transition.

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


Native-to-Flutter requests deserve the same structure as Flutter-to-native requests.

## 13. Native → Flutter Destination Contract

The same contract is needed when native sends the customer to a particular Flutter destination, such as an invoice detail page.

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


A structured destination still needs to specify what happens to an engine that already holds an older route stack.

## 14. Cached Engine Navigation Problem

A cached engine makes that destination contract more important. The UI may be gone while the Flutter navigation stack is still alive.

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


If one shared stack becomes too difficult to manage, multiple Flutter instances offer a different ownership model.

## 15. Multiple Flutter Instances

As the app grows, the customer may enter Flutter, visit native, and enter Flutter again within one stack. That is when the team should revisit the one-engine assumption.

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


Flutter provides a grouped engine API for making those independent instances more efficient.

## 16. FlutterEngineGroup

Independent instances need not mean a completely separate engine cost each time. FlutterEngineGroup provides a way to create multiple engines with shared low-level resources.

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


The same engine model also enables Flutter to occupy only part of a native page.

## 17. Partial-Screen Flutter Views

Not every Flutter appearance fills a whole page. A native screen can also place a Flutter-rendered area among native components.

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


After deciding where Flutter appears, the team must decide which runtime supplies the data it displays.

## 18. Local Storage and State Synchronization

Navigation is only half of the customer journey. After entering Orders, Flutter needs to know which user, company, and session the native host considers current.

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


The following token example makes that state problem concrete.

## 19. Shared Storage Problem

A token change shows why reading and writing the same storage is not enough. Follow the value from disk into each runtime cache.

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


The solution begins by distinguishing persisted values from the values currently held in memory.

## 20. Shared Storage Is Not Shared Runtime State

The inconsistency is a design problem rather than a storage API problem. The team has to decide which side owns each shared value and how the other side hears about changes.

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


Those ownership rules also define what data can be passed once at startup and what requires live updates.

## 21. Passing Initial Data to Flutter

At engine startup, native can provide useful context. The team must still separate that initial snapshot from updates that happen later in the session.

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


A bridge for those updates should remain an abstraction, not a chain of services calling back and forth.

## 22. Circular Dependency Risk

As more features call across the boundary, a different risk appears: native and Flutter can become dependent on each other in a circle.

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


This separation becomes especially important when there is more than one Dart runtime.

## 23. Multiple Engines and Dependency Injection

Multiple engines sharpen the ownership question. Dart-level dependency injection belongs to each engine and does not silently become shared application state.

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


With object ownership established, the team can test the plugins that attach to those runtimes.

## 24. Plugin Limitations in Add-to-App

The engine lifecycle also affects plugins. A plugin that works when a Flutter screen is always visible may behave differently in a cached or multi-engine host.

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


Plugin compatibility is one constraint; packaging multiple Flutter libraries is another.

## 25. Flutter Add-to-App Mobile Limitation — Multiple Flutter Libraries

The phrase multiple Flutter instances can also mislead teams planning their repositories and build artifacts. Mobile packaging has a separate limitation.

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


These integration limits are part of the operational and security design of the final application.

## 26. Security

Once both runtimes are in the shipped app, security review must cover both native and Dart code as well as the bridge between them.

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


Security checks need to be built into the same pipeline that produces the combined application.

## 27. CI/CD

The same combined ownership applies to delivery. The Flutter module has to reach the final native build in a repeatable, versioned way.

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


A reliable pipeline also makes any later entry-point migration easier to evaluate.

## 28. Entry-Point Migration Risk

A team may eventually consider moving the application entry point to Flutter. That is a later migration decision with effects far beyond the next screen.

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


Until that migration is justified, a clear native-host architecture can scale with additional Flutter features.

## 29. Recommended Large-Scale Architecture

By this stage, the growing app needs one view of the whole architecture. The following layout puts the native host, integration layer, and Flutter features together.

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


The diagram becomes more useful when each responsibility has a named owner and a crossing mechanism.

## 30. Recommended Ownership Matrix

The final step is to turn architectural intent into an ownership agreement that engineers can use when the next feature is added.

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
