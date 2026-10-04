# Flutter Add-to-App: إزاي تخلي الـ Native والـ Flutter يشتغلوا كتطبيق واحد؟

عندك تطبيق Android أو iOS شغال، وفيه Dashboard و Authentication و Payment معمولين Native. قررت تبني Orders بـ Flutter بدل ما تعيد كتابة التطبيق كله. أول Demo غالبًا هيبقى سهل: تدوس Orders، تفتح Flutter screen، وتقول إن الـ integration نجح.

بس المستخدم مش هيقف عند أول screen. هيفتح Order Details، يروح Native Payment، يرجع، يغيّر الـ account، وبعدها يدخل Feature تانية معمولة بـ Flutter. هنا السؤال الحقيقي: مين ماسك الـ navigation؟ مين صاحب الـ state؟ والـ FlutterEngine لسه عايش ولا اتعمله destroy؟

خلينا نمشي مع الرحلة دي خطوة خطوة. الـ topics والـ APIs هتفضل بالإنجليزي عشان تربط الشرح بالـ code والـ documentation، لكن التفسير هيبقى بالمصري. وكل الرسومات والـ snippets الموجودة في المقال الأصلي هتلاقيها هنا.

## 1. What Flutter Add-to-App Actually Means

أول حاجة لازم نتفق عليها: Add-to-App مش معناه إن Flutter بقى هو التطبيق الأساسي. الـ Native app لسه هو الـ host والـ entry point، وFlutter داخل كـ module علشان تبني Features معينة بالتدريج.

كده عندك عالمين بينهم boundary. الـ Native عنده lifecycle و services و storage، وFlutter عنده Dart runtime و UI state. تقدر توصلهم بـ MethodChannel أو EventChannel أو BasicMessageChannel أو Pigeon أو Plugins. لكن مجرد إن رسالة عدّت بينهم مش معناه إن الـ state اتزامنت تلقائي. الرسمين الجايين بيبينوا العلاقة وطرق التواصل.

```text
Native Application
        ↕
Integration Boundary
        ↕
Flutter Runtime
```

```text
MethodChannel
EventChannel
BasicMessageChannel
Pigeon-generated APIs
Flutter Plugins
```

## 2. Add-to-App Is More Than Opening Flutter Screens

فتح Flutter screen هو الجزء اللي بيبان للناس، لكنه غالبًا مش أصعب جزء. أول ما المستخدم يتحرك جوه التطبيق هتدخل في Navigation و Authentication و Local Storage و State Synchronization و Lifecycle و Memory و Debugging و Security و CI/CD.

أكتر نقطتين بيظهر فيهم الاحتكاك عادة هما Navigation و Storage/State Synchronization؛ لأن الـ ownership بيتنقل هنا بين الـ Native والـ Flutter. لو ما اتحددش بدري، هتلاقي الشاشة اتفتحت صح لكن بياناتها قديمة، أو Flutter رجع على route مش اللي المستخدم متوقعه. الأمثلة الجاية بتوضح ليه اختزال Add-to-App في شاشة واحدة مضلل.

```text
Add-to-App = Native App opens Flutter Screen
```

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

```text
1. Navigation
2. Local Storage / State Synchronization
```

## 3. Core Add-to-App Architecture

عشان ما تخليش كل Feature تعرف تفاصيل النظام التاني، حط Integration Layer واضحة في النص. الـ Native يفضل مسؤول عن Root Navigation و Authentication و Platform Services و Native Storage و DI. وFlutter يفضل مسؤول عن Flutter Navigation و State Management و Networking و Features بتاعته.

الطبقة اللي في النص تمسك FlutterEngine Manager و NativeBridge و Navigation Contract و State Contract و Channels أو Pigeon وتنسيق الـ lifecycle. كده لما Flutter يطلب Native Payment ما يحتاجش يعرف اسم UIViewController، ولما Native يفتح Invoice في Flutter ما يحتاجش يعرف Widget اسمها إيه. الرسم الجاي بيجمع التقسيمة.

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

## 4. Flutter Module

بعد ما فهمنا الحدود، نضيف Flutter للمشروع. تقدر تعمل Flutter module بالأمر الموجود تحت وتخلي NativeApp و FlutterModule جنب بعض. الـ Native app يفضل هو اللي بيبني النسخة النهائية اللي هتنزل للمستخدم.

على Android ممكن تربط الـ module في Gradle أو توزعه كـ AAR. وعلى iOS طريقة الدمج تعتمد على الـ setup، وممكن تشمل generated artifacts/build phases أو Swift package integration. اختار طريقة تناسب الفريق والـ CI، لأن Flutter code بقى جزء من release الـ Native app. المثالين تحت يوضحوا الأمر وشكل المجلدات.

```bash
flutter create --template module my_flutter
```

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

## 5. FlutterEngine

لما المستخدم يفتح Orders، اللي بيظهر قدامه UI container، لكن اللي بيشغّل Dart وFlutter فعليًا هو FlutterEngine. الـ engine فيها Dart execution و isolate و plugin registry و platform channels و rendering pipeline و state.

على Android الـ UI ممكن تبقى FlutterActivity أو FlutterFragment، وعلى iOS تبقى FlutterViewController. المهم إن الـ container غير الـ engine. تقدر تنشئ engine قبل ما تظهر الشاشة، وتقدر تقفل الشاشة وتسيب الـ engine عايشة. الفرق ده أساس قرار الـ pre-warming والـ caching اللي جاي.

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

```text
FlutterActivity or FlutterFragment
       │
       └── FlutterEngine
```

```text
FlutterViewController
       │
       └── FlutterEngine
```

## 6. Flutter Engine Lifecycle

خلينا نبدأ بأبسط اختيار، Strategy A — Create Engine When Needed: المستخدم يفتح Flutter Feature، فالـ Native ينشئ FlutterEngine، يشغّل Dart، يجهّز Flutter، وبعدها يعرض الـ UI. لما المستخدم يخلص، ممكن تعمل destroy للـ engine.

الميزة إن الـ lifecycle واضح والـ state بتبدأ نظيفة ومفيش memory محجوزة طول الوقت. المقابل إن كل دخول ممكن يدفع تكلفة startup و plugin initialization و bootstrap و DI initialization. لو الـ Feature بتتفتح كتير، التكلفة ممكن تبقى محسوسة. الرسومات الجاية بتعرض التسلسل والمزايا والعيوب.

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

```text
Close Flutter
    ↓
Destroy FlutterEngine
```

```text
Simple lifecycle
Clean state
Lower persistent memory usage
```

```text
Engine startup latency
Dart startup
Plugin initialization
Flutter application bootstrap
DI initialization
```

## 7. Pre-Warmed / Cached Flutter Engine

الاختيار التاني إنك تعمل pre-warm للـ FlutterEngine بدري وتخزّنها. المستخدم يكمل في الـ Native app عادي، ولما يفتح Flutter Feature توصل الـ UI container بالـ cached engine بدل ما تبدأ كل حاجة من الصفر.

الكود تحت تصور Android للـ FlutterEngineCache وطريقة فتح FlutterActivity بالـ cached engine. النقطة اللي لازم تفضل في بالك: عمر الـ engine مش هو عمر الـ screen. لو الشاشة اتقفلت والـ engine فضلت عايشة، الـ Dart state والـ Flutter navigator ممكن يفضلوا زي ما هم. وده هيفرق لما المستخدم يرجع بعد شوية.

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

```text
User opens Flutter Feature
          ↓
Reuse FlutterEngine
          ↓
Attach FlutterActivity /
FlutterViewController
```

```kotlin
val flutterEngine = FlutterEngine(context)

flutterEngine.dartExecutor.executeDartEntrypoint(
    DartExecutor.DartEntrypoint.createDefault()
)

FlutterEngineCache
    .getInstance()
    .put("main_flutter_engine", flutterEngine)
```

```kotlin
FlutterActivity
    .withCachedEngine("main_flutter_engine")
    .build(context)
```

```text
FlutterEngine lifetime
         ≠
Flutter screen lifetime
```

## 8. Flutter Add-to-App Modes

قبل ما نختار Engine واحدة أو أكتر، لازم نفرّق بين طريقتين Flutter بتستخدمهم في Add-to-App. على Android و iOS الـ mobile model هو Multi-engine: كل Flutter instance ليها FlutterEngine مستقل و Dart program و state خاصين بيها.

على Web فيه Multi-view: أكتر من view ممكن يشتركوا في Dart program واحد وبالتالي يشاركوا Dart objects. ما ينفعش نفترض إن Flutter views على الموبايل هتشارك نفس الـ runtime state بالطريقة دي. الفرق ده هيظهر تاني لما نناقش Mixed Navigation و FlutterEngineGroup.

## 9. Navigation — The First Major Challenge

دلوقتي نمشي مع المستخدم. عندنا Native Navigation وعندنا Flutter Navigation، ودي مش نفس الـ stack. ممكن يفضل Native من أول الرحلة لآخرها، وممكن يدخل Flutter ويكمل جواه، وممكن يدخل Flutter وبعدها يرجع على Native screen.

في تطبيق كبير الرحلة ممكن تبقى Native → Flutter → Native → Flutter. لو كل انتقال اتساب من غير قاعدة ownership، الـ back behavior والـ deep links والـ current route هيبقوا صعبين. الرسومات التالية بتوضح أشكال الرحلات اللي التصميم لازم يستحملها.

```text
Native Navigation
        +
Flutter Navigation
```

```text
1. Native → Native

2. Native → Flutter → Flutter

3. Native → Flutter → Native
```

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

## 10. Case 2 — Native → Flutter → Continue in Flutter

هنا المستخدم يفتح Orders من Native Dashboard، وبعدها يروح Order Details ثم Order History؛ الصفحات التلاتة الأخيرة كلها Flutter. الـ Native ينشئ أو يعيد استخدام الـ engine ويعرض Flutter container، وبعدها Flutter Router يدير التنقل الداخلي.

لو كل خطوة جوه Flutter رجعت تسأل Native يفتح الشاشة اللي بعدها، يبقى الـ Native مضطر يعرف تفاصيل Flutter routes. ده coupling مالوش داعي. خليه يملك نقطة الدخول، وخلي Flutter يملك الخطوات اللي جواه. الأمثلة تحت بتوضح الـ flow والفرق بين المسار المربك والمسار المفضل.

```text
Native Dashboard
        ↓
Flutter Orders
        ↓
Flutter Order Details
        ↓
Flutter Order History
```

```text
Native UIViewController
        ↓
FlutterViewController
        ↓
Flutter Navigator
```

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

```text
Flutter Screen A
       ↓
Native
       ↓
Tell Flutter to open B
       ↓
Flutter B
```

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

## 11. Case 3 — Native → Flutter → Native

دلوقتي المستخدم واقف في Flutter Order Details وداس Pay، لكن Payment لسه Native. Flutter هنا ما ينفعش يمسك UINavigationController أو Android navigation مباشرة. هو يرسل intent: افتح Native Payment للـ Order ID ده.

الـ Integration Layer يستقبل الطلب، والـ Native Navigation Controller ينفّذ الانتقال. كده Flutter عارف هو عايز إيه، من غير ما يبقى مربوط بـ Native implementation. الرسومات الجاية بتتبع الطلب من Flutter لحد الشاشة الـ Native.

```text
Native Dashboard
       ↓
Flutter Orders
       ↓
Flutter Order Details
       ↓
Native Payment
```

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

```text
navigationController.pushViewController(...)
```

```text
Open native payment screen
Order ID = 123
```

## 12. Navigation Ownership

من الأمثلة السابقة نقدر نكتب قاعدة واضحة: Native Root Navigation تبع Native، والـ Flutter Internal Navigation تبع Flutter. أي Flutter → Native Request أو Native → Flutter Destination يعدّي على Integration Layer.

يعني Flutter يبعث openNativeDestination(payment(orderId: 123)) بدل ما يقول: هات UINavigationController واعمل push لـ PaymentViewController. أول صيغة بتوصف intent، والتانية بتسرّب تفاصيل منصة للتطبيق التاني. الرسومات تحت بتحط القاعدة والصيغتين جنب بعض.

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

```text
openNativeDestination(
    payment(orderId: 123)
)
```

```text
Get UINavigationController
Push PaymentViewController
```

## 13. Native → Flutter Destination Contract

الاتجاه العكسي محتاج Contract برضه. بدل ما Native يزود method جديدة لكل Feature — openOrders و openInvoice و openCustomer — ممكن يبعث FlutterDestination منظمة فيها feature و action و parameters.

مثلًا Invoice Details بـ invoiceId 1005. الـ FlutterHost يستقبل الطلب، والـ Flutter Router يوصّل المستخدم للصفحة المطلوبة. كده الـ Native يعرف destination مستقرة، ومش محتاج يعرف Widgets أو Flutter route implementation. الـ snippets التالية بتوضح شكل الطلب ومساره.

```text
openOrders()
openInvoice()
openCustomer()
```

```text
FlutterDestination
```

```json
{
  "feature": "invoice",
  "action": "details",
  "parameters": {
    "invoiceId": 1005
  }
}
```

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

## 14. Cached Engine Navigation Problem

هنا يظهر تأثير الـ cached engine. المستخدم كان في Home → Orders → OrderDetails، قفل Flutter UI، لكن الـ engine لسه عايشة. بعد شوية Native طلب Reports. لو وصلنا نفس الـ engine بالشاشة وخلاص، ممكن يرجع المستخدم على OrderDetails القديمة.

عشان كده الـ navigation request لازم يحدد mode: push أو replace أو reset أو resume. reset معناها ابدأ stack جديدة وافتح Reports؛ resume معناها ارجع عمدًا للـ stack القديمة. مفيش mode صح في كل الحالات؛ اللي يحدد ده هو الـ user journey. الرسومات والكود التاليين بيمشوا في السيناريو بالترتيب.

```text
Home
 ↓
Orders
 ↓
OrderDetails
```

```text
Reports
```

```text
Home
 ↓
Orders
 ↓
OrderDetails
```

```dart
enum FlutterNavigationMode {
  push,
  replace,
  reset,
  resume,
}
```

```text
openFlutter(
    destination: Reports,
    mode: reset
)
```

```text
Ignore previous navigation stack
        ↓
Reset
        ↓
Open Reports
```

```text
resume
```

## 15. Multiple Flutter Instances

لما الـ navigation stack تبقى Native A → Flutter A → Native B → Flutter B، Engine واحدة cached ممكن تشتغل، بس Flutter لازم تحفظ وتسترجع stacks مختلفة بعناية. حل تاني إن Flutter A و Flutter B يبقوا instances مستقلين.

كل instance ساعتها ليها Navigation Stack و Application State و UI State و Dart execution بتوعها. Flutter بتذكر الـ hybrid stack دي كسبب مهم لاستخدام Multiple Flutter Instances، خصوصًا لما Flutter ما تبقاش leaf node في الرحلة. الرسومات الجاية بتوضح المشكلة والبديل.

```text
Native A
   ↓
Flutter A
   ↓
Native B
   ↓
Flutter B
```

```text
Flutter Instance A
        +
Flutter Instance B
```

```text
Native
  ↓
Flutter
  ↓
Native
  ↓
Flutter
```

```text
Navigation Stack
Application State
UI State
Dart execution
```

## 16. FlutterEngineGroup

لو اخترنا أكتر من engine، مش لازم كل واحدة تدفع كل تكلفة الموارد من أول وجديد. FlutterEngineGroup ينشئ FlutterEngines مستقلة على مستوى الـ app state، لكن يقدروا يشاركوا موارد داخلية زي GPU context و font metrics و isolate group snapshot.

أول engine لسه بتدفع تكلفة البداية المعتادة، والـ engines اللي بعدها ممكن تستفيد من المشاركة. أول engine مش لازم تفضل عايشة للأبد، لكن لازم تفضل engine واحدة على الأقل من الـ group عايشة عشان ميزة المشاركة تفضل موجودة. وبرضه قيس memory و startup في تطبيقك ومع Plugins بتاعتك؛ ما تعتمدش على رقم عام. الرسومات تحت بتوضح الفكرة.

```text
FlutterEngineGroup
```

```text
FlutterEngine()
FlutterEngine()
FlutterEngine()
```

```text
FlutterEngineGroup
        │
    ┌───┼────┐
    │   │    │
    ▼   ▼    ▼
   E1  E2   E3
```

```text
GPU context
Font metrics
Isolate group snapshot
```

## 17. Partial-Screen Flutter Views

Add-to-App مش شرط يملأ الشاشة كلها. ممكن Native screen فيها Native Toolbar و Native Component ومساحة Flutter-rendered وبعدها Native Button. ده يفتح الباب لنقل Feature أو جزء منها تدريجيًا.

لكن لو ظهر أكتر من Flutter view على Mobile، افتكر إن التفكير هنا مرتبط بالـ Multi-engine architecture، مش Web-style Multi-view اللي بتشارك Dart objects. المثالين التاليين بيوضحوا الفرق بين Full-screen و Partial-screen.

```text
Full Native Screen
       ↓
Full Flutter Screen
```

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

## 18. Local Storage and State Synchronization

خلصنا رحلة الـ Navigation؛ نرجع للبيانات. Flutter وNative ممكن يتجهزوا للوصول لنفس Keychain أو UserDefaults أو SharedPreferences أو Secure Storage أو Database أو Files. ده مفيد، لكنه ما يحلش مشكلة الـ shared state لوحده.

كل Runtime ممكن يكون محتفظ بنسخة في الـ memory. فلو جهة غيرت قيمة في storage، الجهة التانية مش شرط تعرف أو تحدّث الـ cache عندها. القائمة الجاية بتعرض persistent mechanisms ممكن تتشارك، وبعدها هنمشي مع Token واحد عشان المشكلة تبقى ملموسة.

```text
Keychain
UserDefaults
SharedPreferences
Secure Storage
Database
Files
```

## 19. Shared Storage Problem

افترض إن Native كتبت token = A، وFlutter قرأت A. بعدين Flutter كتبت token = B في الـ persistent storage. Flutter دلوقتي شايفة B، لكن Native ممكن لسه في الـ memory cache عندها A.

كده التطبيق ممكن يبعث requests بهويتين مختلفتين حسب الـ runtime. الـ snippets الستة التالية بتعرض القصة خطوة خطوة من أول كتابة الـ token لحد حالة الـ inconsistency. ركّز إن تغيير قيمة الـ storage ما مسحش نسخة Native اللي في الذاكرة.

```text
token = A
```

```text
token = A
```

```text
token = B
```

```text
token = B
```

```text
Memory Cache

token = A
```

```text
Persistent Storage
token = B

Native Memory
token = A

Flutter Memory
token = B
```

## 20. Shared Storage Is Not Shared Runtime State

الفرق الجوهري هنا: Shared Persistent Storage مش هي Shared Runtime Memory. حتى لو الطرفين بيقرأوا من نفس المكان، كل واحد يقدر يعمل cache مستقلة.

عشان كده لكل قيمة مشتركة اسأل: مين الـ owner؟ مين يكتب ومين يقرأ فقط؟ فين source of truth؟ هل القيمة cached؟ إيمتى نعمل invalidate؟ الطرف التاني هيعرف التغيير إزاي؟ هل هنحفظها؟ وهل هنرسل event؟ من غير إجابات واضحة، المشكلة هترجع حتى لو غيرت storage library. الرسم والقائمة الجاية بيجمعوا الأسئلة دي.

```text
Shared Persistent Storage
           ≠
Shared Runtime Memory
```

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

## 21. Passing Initial Data to Flutter

وقت تشغيل الـ engine، Native ممكن يمرر Environment أو Language أو Tenant أو Initial Feature أو Initial Route أو Feature Flags أو User Context أو Build Configuration. Flutter تستخدم القيم دي في bootstrap.

بس Initial Parameters عبارة عن snapshot وقت البداية، مش live state channel. لو المستخدم غيّر Company أو عمل logout بعدين، لازم Platform Channel أو Persistent Source of Truth مع Change Notification أو آلية تزامن واضحة. وده أهم مع cached engine لأنها ممكن تفضل عايشة بعد الـ bootstrap بفترة طويلة. الرسومات تحت تفصل بين البداية والتغييرات اللاحقة.

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

```text
Native
   ↓
Initialize Engine
   ↓
Initial Parameters
   ↓
Flutter bootstrap()
```

```text
Initial Parameters
```

```text
Platform Channel
```

```text
Persistent source of truth + change notification
```

## 22. Circular Dependency Risk

لو Native Feature نادت Flutter Feature، وFlutter نادت Native Service، والـ Native Service رجعت تعتمد على Flutter Service، بقى عندك Circular Dependency. المشكلة مش بس في رسم الأسهم؛ Initialization و Lifecycle و Testing و Navigation و Versioning هيبقوا أصعب.

الأفضل إن الطرفين يعتمدوا على Integration Contract. الـ Native Internals ما تعرفش تفاصيل Flutter Internals، والعكس صحيح. كل جهة تتعامل مع abstraction مشتركة وتنفّذ دورها. الرسومات التالية بتعرض الـ cycle وبعدين البديل المنظم.

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

```text
Native depends on Flutter
       ↓
Flutter depends on Native
       ↓
Circular ownership
```

```text
Initialization
Lifecycle
Testing
Navigation
Release versioning
Dependency management
```

```text
Native Internals
        ↓
Integration Contract
        ↑
Flutter Internals
```

## 23. Multiple Engines and Dependency Injection

لما يبقى عندك Engine A و Engine B، كل واحدة عادة بتشغّل Dart application و object graph مستقلين. لو بتستخدم get_it، singleton جوه Engine A مش global singleton لكل FlutterEngines.

لو محتاج state عامة فعلًا، خلّي ownership خارج الـ Dart runtimes المستقلة أو اعمل synchronization صريح عن طريق Native أو source of truth مشتركة. ما تبنيش التصميم على افتراض إن DI container واحدة هتظهر في كل engine. الرسمين تحت يوضحوا المثال والافتراض الغلط.

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

```text
get_it singleton
```

## 24. Plugin Limitations in Add-to-App

الـ Plugins لازم تتجرب تحت الـ lifecycle الحقيقي بتاع Add-to-App. Plugin ممكن تفترض إن Flutter Activity موجودة طول الوقت، مع إنك عامل pre-warm للـ engine من غير UI attached.

اختبر Engine Lifecycle و Activity Attachment و Multiple Engine Behavior و Detach/Reattach و Background Behavior. وافتكر إن Android Add-to-App يتوقع AndroidX. مجرد إن Plugin اشتغلت في Flutter app مستقلة ما يضمنش إنها هتتصرف صح في Hybrid App. الـ snippets التالية بتعرض الافتراض والمواقف اللي محتاجة اختبار.

```text
A Flutter Activity is always available
```

```text
Engine lifecycle
Activity attachment
Multiple engine behavior
Detach / reattach
Background behavior
```

## 25. Flutter Add-to-App Mobile Limitation — Multiple Flutter Libraries

مصطلح Multiple Flutters ممكن يتفهم غلط. على Mobile، دعم Multiple FlutterEngine Instances ما يعنيش إنك تقدر تحزم Flutter Module A و B و C كـ independent Flutter libraries في التطبيق الواحد.

التصميم المدعوم هنا هو Flutter module/library واحدة، ومنها ممكن تنشئ engines متعددة حسب الرحلة. الفرق ده مهم قبل ما تقسّم الـ repositories أو الـ build artifacts لكل Feature. الرسومات الجاية بتفصل بين الحالتين.

```text
One Flutter module/library
       ↓
Potentially multiple FlutterEngine instances
```

```text
Flutter Module A
+
Flutter Module B
+
Flutter Module C
```

```text
Multiple engine instances
```

```text
Multiple unrelated packaged Flutter modules.
```

## 26. Security

النسخة اللي هتنزل للمستخدم فيها Native Code و Flutter/Dart Code. بالتالي Security Review لازم تغطي الاتنين: Obfuscation و Symbols و Secure Storage و Signing و Logs و Environment Values و Debug Configuration و Build Artifacts.

ما تعتبرش API secret آمن لمجرد إنه مكتوب بـ Dart أو اتعمله compile. لو الـ backend يقدر يملك secret حساس، ما تحطوش جوه client app. الأمثلة التالية بتوضح مساحة المراجعة والفكرة الخاصة بالـ client-side secrets.

```text
Native Code
+
Flutter/Dart Code
```

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

```text
API secret inside Flutter
```

## 27. CI/CD

بعد ما Flutter بقت جزء من التطبيق، الـ CI/CD بقت رحلة مشتركة. في أبسط صورة: Flutter Source تتحول لـ Flutter Build، تدخل Native Application، وبعدها تطلع Final Android/iOS Build.

مع فرق أكبر، Flutter CI تعمل Analyze و Test و Build، تنتج Versioned Artifact، تنشرها في Artifact Repository، وبعدها Native CI تستخدم النسخة المحددة وتبني التطبيق النهائي. ده يخلي الـ integration قابل للإعادة والتتبع بدل ما كل release تعتمد على ملف اتنقل يدويًا. الرسمين تحت يوضحوا المستويين.

```text
Flutter Source
      ↓
Flutter Build
      ↓
Native Application
      ↓
Final Android/iOS Build
```

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

## 28. Entry-Point Migration Risk

بعد نجاح كذا Feature، ممكن الفريق يفكر ينقل الـ Entry Point من Native إلى Flutter. ده قرار أكبر بكتير من فتح شاشة Flutter جديدة.

التغيير ممكن يمس Startup و Deep Links و Push Routing و Authentication Bootstrap و Analytics و Crash Reporting و Navigation و Background Execution و Lifecycle و CI/CD و Rollback. عشان كده الأفضل تقييمه بعد ما Flutter تبقى mature جوه التطبيق وتبقى حدود المسؤوليات واضحة. الرسومات التالية بتوضح الانتقال ومساحة تأثيره.

```text
Native Entry Point
```

```text
Flutter Entry Point
```

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

## 29. Recommended Large-Scale Architecture

لو التطبيق كبر، تقدر تجمع الصورة كلها في Architecture واحدة: Native فوق للـ lifecycle والـ root navigation والـ existing features والـ session والـ platform services؛ Integration Layer في النص للـ engines والـ bridge والـ destinations والـ state events والـ tracing والـ versioning؛ وFlutter تحت للـ core والـ features.

مش المقصود إن كل مشروع لازم يستخدم نفس أسماء الـ classes أو نفس Packages. المقصود إن المسؤولية تبقى واضحة، وكل crossing يبقى له contract. الرسم التالي بيحط الـ three layers والـ feature areas في صورة واحدة.

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

## 30. Recommended Ownership Matrix

آخر خطوة قبل ما Feature جديدة تدخل: اكتب Ownership Matrix. مين ماسك Application Lifecycle؟ مين ماسك Root Navigation؟ مين صاحب Flutter Navigation؟ مين مسؤول عن Auth و Tokens و Selected Company و Push Notifications و Analytics و Logs؟ وإزاي كل قيمة أو request هتعبر الـ boundary؟

الجدول التالي مثال قرار مبدئي، مش قانون ثابت. بعض القيم زي Selected Company أو User أو Analytics محتاجة Owner صريح يحدده مشروعك. المهم إنك ما تسيبش ownership ضمني؛ لأن معظم مشاكل Add-to-App بتظهر وقت ما كل Runtime تفتكر إنها صاحبة نفس القرار.

- **Application Lifecycle** — Owner: Native; Cross-boundary mechanism: Lifecycle bridge when needed
- **Root Navigation** — Owner: Native; Cross-boundary mechanism: Native coordinator
- **Flutter Navigation** — Owner: Flutter; Cross-boundary mechanism: Flutter Router
- **Native → Flutter Navigation** — Owner: Integration; Cross-boundary mechanism: FlutterDestination
- **Flutter → Native Navigation** — Owner: Integration; Cross-boundary mechanism: NativeDestination
- **Flutter Engine** — Owner: Native; Cross-boundary mechanism: FlutterEngineManager
- **Multiple Engines** — Owner: Native; Cross-boundary mechanism: FlutterEngineGroup
- **Authentication** — Owner: Native; Cross-boundary mechanism: NativeBridge
- **Access Token** — Owner: Native; Cross-boundary mechanism: NativeBridge
- **Refresh Token** — Owner: Native; Cross-boundary mechanism: NativeBridge
- **Selected Company** — Owner: Explicit owner; Cross-boundary mechanism: State + event
- **User** — Owner: Explicit owner; Cross-boundary mechanism: Bridge
- **Flutter Feature State** — Owner: Flutter; Cross-boundary mechanism: Bloc/Cubit
- **Flutter Storage** — Owner: Flutter; Cross-boundary mechanism: Flutter storage
- **Native DB** — Owner: Native; Cross-boundary mechanism: Native repository/bridge
- **Flutter HTTP** — Owner: Flutter where appropriate; Cross-boundary mechanism: Dio
- **Native Platform API** — Owner: Native; Cross-boundary mechanism: Bridge
- **Push Notifications** — Owner: Explicit owner; Cross-boundary mechanism: Bridge
- **Analytics** — Owner: Explicit owner; Cross-boundary mechanism: Bridge/direct
- **Flutter DI** — Owner: Flutter; Cross-boundary mechanism: get_it
- **Native DI** — Owner: Native; Cross-boundary mechanism: Native container
- **Logs/Tracing** — Owner: Shared contract; Cross-boundary mechanism: Trace IDs
- **CI Artifact** — Owner: Flutter CI; Cross-boundary mechanism: Versioned artifact

## الخلاصة

نجاح Add-to-App مش بيتقاس بإن أول Flutter screen فتحت. بيتقاس لما المستخدم يدخل ويخرج بين Native و Flutter، يغيّر الـ account، يرجع لـ cached engine، أو يفتح أكتر من Flutter instance، ولسه يحس إنه في تطبيق واحد. لو كل route و value و engine و artifact ليه owner واضح، الـ migration تقدر تكبر Feature ورا Feature من غير ما الـ boundary تتحول لفوضى.
