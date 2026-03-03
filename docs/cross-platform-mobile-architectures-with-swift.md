# A Comprehensive Guide to Cross-Platform Mobile Architectures

## Preface: The Central Question

When you write code on your laptop and it somehow appears as a beautiful, tappable button on a phone, what actually happens in between? This guide answers that question for four different systems, each with a fundamentally different philosophy about how to bridge that gap.

---

## Part 1: The Analogy

### React Native (Paper) — The UN Interpreter

Imagine the United Nations General Assembly. A delegate speaks French. A listener understands only English. Between them sits a **live interpreter** in a glass booth, listening to every word spoken in French and translating it into English in real time, then sending the translation back the other way when the English speaker responds.

This is React Native's old Paper architecture. Your JavaScript code speaks "JavaScript." The phone's operating system speaks "Native." Between them sits a **bridge** — a translator working in real time. Every time your app wants to do something (draw a button, respond to a tap), the message must be packaged up, passed through the booth, translated, and sent across. The interpreter is good, but they introduce a delay, and they can only handle so many messages per second.

### React Native (Fabric) — The Bilingual Employee

Now imagine the company got smart and hired a **bilingual employee** instead. This person doesn't need a separate booth or a middleman. They can switch between French and English mid-sentence, hold shared documents that both sides can read simultaneously, and respond to requests instantly without waiting for a translation queue to clear.

This is Fabric. The "bridge" has been replaced by a shared layer written in C++ that both JavaScript and Native code can touch directly. The conversation is no longer a one-way queue — it's a direct, live connection.

### Flutter — The Method Actor Who Learned the Role

Flutter ignores the translation problem entirely. Instead of hiring someone to talk to the French-speaking staff, the company **trained an entirely new employee** from scratch who doesn't work with the existing staff at all. This employee builds their own furniture, paints their own walls, and runs their own department — completely independently.

Flutter ships its own **rendering engine** (called Skia, or the newer Impeller) directly inside your app. It doesn't ask iOS or Android to draw a button. It draws the button itself, pixel by pixel, using a canvas it controls entirely. The phone's OS is reduced to just one job: "give me a blank surface to draw on." Flutter handles everything else.

### Swift SDK for Android — The Foreign Architect Who Moves In

Where the previous three architectures all grapple — in different ways — with the translation problem between a high-level language and the native platform, the Swift SDK for Android sidesteps the problem at the source. Instead of hiring a translator, training a bilingual employee, or building a new department, Swift takes a fourth path: it **learns to speak the local language at the machine code level before arriving on-site**.

Swift code is compiled by LLVM — the same compiler infrastructure that powers C, C++, Rust, and Objective-C — directly into ARM64 or x86 machine code. The output is a standard `.so` shared library, the exact same binary format that Android uses for all of its own native code. When the Android OS loads a Swift library at runtime, it cannot distinguish it from a library written in C++ or Kotlin Native. The foreign architect has moved in, and the building has no idea the blueprints were drawn in a foreign language.

---

## Part 2: Source-to-Native Translation & UI Rendering

### Architecture 1: React Native Paper (The Old Bridge)

**How your code runs:**

You write JavaScript or TypeScript. When the app launches, a JavaScript engine called **Hermes** (or previously JavaScriptCore) boots up. Think of Hermes as a JavaScript interpreter — a program that reads your JS code line by line and executes it. Your JS code lives in its own world, running on its own thread, completely isolated from the phone's native world.

**The Bridge — the central character:**

The Bridge is a serialized, asynchronous message queue. "Serialized" means every piece of data has to be converted into a text format (JSON) before crossing. "Asynchronous" means you send a message and then wait — you don't get an instant reply.

Here's what happens when your app first loads and needs to show a button:

1. Your JS code runs and says "I need a `<Button>` component."
2. React Native's JS layer converts that into a JSON message: `{"type": "createView", "id": 4, "props": {"text": "Submit", "color": "blue"}}`.
3. This JSON string is **serialized** (packaged) and placed onto the Bridge queue.
4. The Native side (running on a separate thread in Objective-C/Swift or Java/Kotlin) picks up the message, **deserializes** it (unpacks the JSON), and reads the instructions.
5. The Native side then calls the actual iOS or Android system APIs: "Create a real UIButton (iOS) with the text 'Submit'."
6. The phone's OS draws its own, genuine native button widget on screen.

**The key insight about rendering:** React Native Paper renders **real, native OS widgets**. The button you see on iOS is a genuine `UIButton`. The button on Android is a genuine `android.widget.Button`. This means your app looks and feels exactly like native apps — because it's using the same building blocks. The tradeoff is that every instruction to create or update those widgets must cross the bridge.

**The Bridge's core weakness:**

Because the Bridge is asynchronous and serialized, two problems emerge. First, there's **latency**: every interaction has a round-trip cost. Second, the layout system (called Yoga, which calculates where things should be positioned on screen) runs on the JS thread, but the actual rendering happens on the native thread. They can't share data in real time, which is why complex animations in Paper apps would sometimes stutter — the two sides had to constantly "shout across the room" at each other.

---

### Architecture 2: React Native Fabric (The New Architecture)

Fabric is a ground-up rewrite addressing every weakness of the Bridge.

**JSI — JavaScript Interface:**

The most important concept in Fabric is **JSI** (JavaScript Interface). Instead of JSON messages on a queue, JSI is a C++ layer that allows the JavaScript engine to hold **direct references** to native objects. Think of the difference between mailing a letter describing a chair versus being in the same room, reaching out, and touching the actual chair. JSI eliminates the mailroom entirely.

With JSI:

- JavaScript can call native functions **synchronously** — no waiting in a queue.
- JavaScript can hold a C++ object reference and call methods on it directly.
- Data doesn't need to be serialized into JSON and back. It can stay as native C++ data structures.

**The Fabric Renderer:**

The rendering pipeline is rebuilt in C++, which means it runs independently of both the JS thread and the native UI thread. When your code says "I need a Button," Fabric:

1. Creates a **shadow tree** — a lightweight C++ representation of your entire UI hierarchy.
2. Runs **Yoga** (the layout engine) in C++ — now much faster and runnable on any thread.
3. Calculates the final positions and sizes of every element.
4. Commits the result to the native UI thread, which then creates or updates real native widgets.

The critical difference from Paper: layout calculation and the JS logic no longer have to keep shouting at each other across a bridge. The C++ layer acts as shared memory both sides can read and write directly.

**TurboModules:**

Fabric also introduced TurboModules — a replacement for the old native module system. In Paper, all native modules (like camera access, GPS, etc.) were loaded at startup, whether you used them or not. TurboModules load lazily (only when first used) and communicate via JSI rather than the Bridge, making startup faster and interactions snappier.

**UI Rendering:** Like Paper, Fabric still renders **real native OS widgets**. The philosophy hasn't changed — only the communication mechanism has.

---

### Architecture 3: Flutter (AOT Compilation + Custom Renderer)

Flutter is architecturally unlike either React Native approach. It has two revolutionary differences: how code is compiled, and how the UI is drawn.

**AOT Compilation — Ahead of Time:**

You write Dart code. Unlike JavaScript (which is interpreted at runtime by an engine), Dart is compiled **Ahead of Time** into native machine code before the app is ever installed.

Think of the difference between a recipe book (interpreted — someone reads and cooks in real time) versus a pre-cooked meal that's already done (AOT — all the "cooking" happened in advance). When a Flutter app launches, the Dart code is already in the phone's native language. There's no interpreter, no VM startup overhead, no just-in-time translation. The CPU runs your Dart logic directly.

This is why Flutter apps tend to start quickly and run smoothly — the language barrier has been eliminated before the user ever opens the app.

**The Rendering Engine — Skia and Impeller:**

Here is Flutter's most radical design decision: **it doesn't use native OS widgets at all.**

Flutter ships with its own 2D graphics rendering engine. Originally this was Skia (the same engine that powers Google Chrome). Newer Flutter versions use **Impeller**, a purpose-built renderer designed to eliminate shader compilation jank (a technical problem where the first time certain visual effects appear, there's a brief stutter as the GPU compiles the instructions for them).

When Flutter needs to draw a button, it doesn't ask iOS to create a `UIButton`. Instead, it asks the OS for one thing only: a **blank canvas** (a raw surface — called a `FlutterView`). Then the rendering engine takes over:

1. Your Dart code builds a **Widget tree** — a description of your UI (Button, with text "Submit", with blue color, at position x=100, y=200).
2. Flutter converts that Widget tree into an **Element tree** (a live, stateful version of your widgets).
3. The Element tree is turned into a **RenderObject tree** — concrete objects that know how to calculate their own size and position.
4. The RenderObject tree is handed to the **Compositor**, which groups everything into layers.
5. The Compositor passes these layers to **Skia/Impeller**, which issues **draw calls** directly to the GPU.
6. The GPU paints pixels directly onto the screen canvas.

**The key insight about rendering:** Flutter paints every single pixel itself. A Flutter button isn't a `UIButton` or an `android.widget.Button`. It's Flutter drawing a rectangle, adding rounded corners, painting text, adding a shadow, and handling touch input — all using its own code. The OS is simply a window to display the result.

**The tradeoff:** Because Flutter draws its own widgets, it never automatically gets OS-level UI updates for free. If Apple redesigns their buttons in iOS 18, Flutter apps won't automatically look different — Flutter has its own widget library (called Material and Cupertino) that must be updated independently. On the flip side, Flutter apps look **identical** on iOS and Android by default, and pixel-perfect custom designs are significantly easier.

---

### Architecture 4: Swift SDK for Android (AOT Compilation + JNI Interoperability)

The Swift SDK for Android represents a fourth architectural philosophy — one that is distinct from all three approaches above. Where React Native (Paper and Fabric) negotiates a conversation between two worlds, and where Flutter builds a world of its own, the Swift SDK compiles away the gap entirely before the app is ever deployed.

**How your code runs:**

You write Swift code. Like Flutter's Dart, Swift is compiled **Ahead of Time** — but through LLVM, the same compiler backend that powers C, C++, Rust, and Objective-C. The output is raw ARM64 or x86 machine code packaged as a standard `.so` (shared object) dynamic library. There is no interpreter, no virtual machine, and no separate runtime engine bundled into the app. When the Android OS loads your Swift library, it is indistinguishable from any other native binary.

Unlike Flutter's AOT compilation — which still requires the Flutter engine (Skia/Impeller, the Dart runtime layers) to travel with the app — Swift's AOT output stands alone. Only the Swift standard library is bundled, and it is lean compared to a full rendering engine.

**The Three Required Components:**

Cross-compiling Swift for Android requires three pieces working in concert. Understanding what each one does clarifies the architecture as a whole.

1. **The Host Toolchain** — the `swift` compiler and associated tools installed on your development machine (macOS or Linux). This is what reads your Swift source files and produces compiled output. Crucially, the toolchain version must match the SDK version exactly — there is no version flexibility here.

2. **The Swift SDK for Android** — a pre-built sysroot bundle: a collection of Swift standard libraries, the Swift runtime (which handles ARC memory management, concurrency, and reflection), and Android API bindings — all pre-compiled for Android targets (`arm64-v8a`, `armv7`, `x86_64`, `x86`). This bundle teaches the host toolchain how to produce Android-compatible binaries instead of macOS or Linux ones. It is installed as a named SDK slot, similar to how Xcode manages iOS and macOS SDKs.

3. **The Android NDK (Native Development Kit)** — specifically version r27d. The NDK provides the C/C++ system headers (`libc`, `libdl`, `pthread`, and others), the `clang` linker, and the platform-specific sysroot that Swift's LLVM backend needs to produce a valid Android binary. Think of the NDK as the local building code that tells the foreign architect what materials and standards the structure must conform to. Without it, the compiler cannot produce a binary that Android's dynamic linker will accept.

**Cross-Compilation — The Build-Time Translation:**

Where React Native Paper translates at _runtime_ (the Bridge processes every tap and update while the user is looking at the screen), and React Native Fabric translates at _runtime_ via JSI, the Swift SDK performs its translation entirely at _build time_. By the time the app is installed on a device, the Swift source code no longer exists in any form — only compiled ARM64 machine instructions remain.

```
$ swift build --swift-sdk aarch64-unknown-linux-android28 --static-swift-stdlib
```

This single command instructs the host toolchain to produce a native Android binary. The `--swift-sdk` flag selects the Android SDK bundle; `aarch64` specifies the ARM64 architecture; `android28` targets Android API level 28. The output is a standard ELF binary — the native executable format on Android — that the Android OS can run directly.

**JNI — The Runtime Seam:**

For command-line executables, Swift code can run directly on Android with no further ceremony. But Android applications are not command-line tools. They are assembled as `.apk` archives and launched by the Android runtime, which is fundamentally a Java/Kotlin environment. This is where the **Java Native Interface (JNI)** enters the architecture.

JNI is Android's own mechanism for calling native C-ABI code from Java or Kotlin. It is not a React Native invention, not a Swift invention — it is Android's built-in bridge between the managed Java world and the native binary world. Every Android app that calls into C or C++ uses JNI. Swift, compiling to a standard `.so` library with C-compatible exports, slots directly into this existing mechanism.

The practical flow for a full Android app is:

1. Swift source is cross-compiled into one `.so` library per architecture.
2. The `.so` files are placed into the Android project's `jniLibs/` directory (e.g., `arm64-v8a/`, `x86_64/`).
3. Gradle bundles them into the `.apk` during the standard Android build.
4. At runtime, the Java/Kotlin layer calls `System.loadLibrary("MySwiftLibrary")` to load the library.
5. Java/Kotlin calls into Swift functions via JNI using normal `native` method declarations.

The JNI seam is thin — a direct C ABI function call with no serialization, no queue, and no intermediate interpreter. Compared to the Bridge in React Native Paper (which serializes every interaction to JSON and despatches it asynchronously), the JNI crossing is orders of magnitude cheaper. Compared to JSI in Fabric (which is a synchronous C++ reference), JNI is architecturally equivalent — both are direct, typed function calls into native code.

**swift-java — Automating the JNI Bindings:**

Writing JNI bindings by hand is tedious and error-prone. The `swift-java` project (part of the Swift open-source ecosystem) solves this with a code generator called `jextract`. Given your Swift source files, `jextract` automatically produces:

- **Java wrapper classes**: one per Swift class or struct, with `native` method declarations for every property and function.
- **Swift `@_cdecl` exports**: C-ABI function stubs that the JNI dispatcher can locate and call.

The generated bindings are safe and typed on both sides — Swift's type system and Java's type system are each fully satisfied. The developer writes Swift business logic and Kotlin/Java UI code; `swift-java` generates the glue layer between them automatically.

**Memory Management — The Hidden Seam:**

This is an architectural challenge that neither React Native nor Flutter faces in the same form, because both of those systems use garbage-collected languages on both sides of their respective seams. Swift uses **ARC (Automatic Reference Counting)** — deterministic, immediate deallocation when the last reference drops — while Java and Kotlin use a **garbage collector** that reclaims memory non-deterministically.

`swift-java` resolves this by making Swift the authoritative owner of all Swift objects. The Java side holds only an opaque `long` integer — a handle to the Swift object's memory address. When the Java garbage collector collects the wrapper object, it triggers a finalizer that calls back into Swift via JNI to release the ARC reference. The Swift object is then freed immediately, on Swift's own terms, according to ARC rules. Java never touches the memory directly; it only holds the ticket.

This design avoids the two classic failure modes of cross-language memory management: memory leaks (orphaned Swift objects that Java forgot to release) and use-after-free crashes (Java accessing Swift memory that ARC already freed).

**UI Rendering — A Fundamentally Different Model:**

This is where the Swift SDK diverges most sharply from both React Native and Flutter.

React Native (Paper and Fabric) renders **real native OS widgets** — the shared layer handles both UI and logic. Flutter renders **no native widgets at all** — the shared layer handles both UI and logic using its own GPU renderer.

The Swift SDK renders **nothing**. It deliberately does not participate in UI rendering. The Swift layer handles only business logic: data models, API networking, state management, validation, serialization, and concurrency. The UI is owned entirely by the platform-native layer — Jetpack Compose or XML layouts on Android, SwiftUI or UIKit on iOS.

This means the blue "Submit" button from our running example is a genuine `android.widget.Button` or Compose `Button`, drawn by Android's own rendering pipeline, with full access to OS-level accessibility, animation, and theming. Swift told it what label to display and what to do when tapped — but the button's visual existence is entirely the OS's responsibility.

The architectural consequence is that the Swift SDK is not primarily a UI framework at all. It is a **shared logic layer** — analogous in philosophy to Kotlin Multiplatform, but using Swift as the shared language. Each platform renders its own native UI; the Swift code beneath provides the brains.

---

## Part 3: Visual Flow Diagrams

### Flow 1: Tapping a Button in React Native Paper

```
USER TAPS BUTTON ON SCREEN
         │
         ▼
┌─────────────────────────┐
│   Native OS Layer       │  ← iOS/Android detects touch event
│   (Obj-C / Java)        │    on a UIButton/android.Button
└────────────┬────────────┘
             │  Touch event serialized to JSON
             │  {"type":"touch","targetId":4,"x":150,"y":300}
             ▼
┌─────────────────────────┐
│       THE BRIDGE        │  ← Asynchronous queue
│  (Serialized JSON pipe) │    Message waits in line
└────────────┬────────────┘
             │  JSON deserialized on JS thread
             ▼
┌─────────────────────────┐
│   JavaScript Thread     │  ← Hermes engine executes your
│   (Hermes Engine)       │    onPress() handler
│                         │
│  onPress={() =>         │
│    setState({count+1})} │
└────────────┬────────────┘
             │  New UI state calculated
             │  Diff computed (what changed?)
             │  Serialized to JSON
             │  {"type":"updateView","id":4,"text":"Clicked!"}
             ▼
┌─────────────────────────┐
│       THE BRIDGE        │  ← Second round-trip across bridge
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│   Native OS Layer       │  ← Native code receives update
│                         │    Calls UIButton.setTitle("Clicked!")
│                         │    OS redraws the native widget
└─────────────────────────┘
             │
             ▼
      SCREEN UPDATES

Total bridge crossings: 2 (touch in, update out)
Latency: Two async round-trips
```

---

### Flow 2: Tapping a Button in React Native Fabric

```
USER TAPS BUTTON ON SCREEN
         │
         ▼
┌─────────────────────────────────┐
│   Native OS Layer               │  ← Touch detected on native widget
│   (Obj-C / Swift / Java/Kotlin) │
└──────────────┬──────────────────┘
               │  Via JSI (direct C++ reference, NO serialization)
               │  ← Synchronous function call, not a queue message
               ▼
┌─────────────────────────────────┐
│   C++ Core (JSI Layer)          │  ← Shared memory both sides
│                                 │    can access directly
└──────────────┬──────────────────┘
               │  Direct JS function invocation
               ▼
┌─────────────────────────────────┐
│   JavaScript Thread             │  ← Hermes executes onPress()
│   (Hermes Engine)               │
│                                 │
│   onPress={() =>                │
│     setState({count+1})}        │
└──────────────┬──────────────────┘
               │  New state triggers re-render
               │  Fabric renderer updates C++ Shadow Tree
               ▼
┌─────────────────────────────────┐
│   Fabric C++ Renderer           │  ← Yoga runs layout in C++
│   + Shadow Tree                 │    Diffs old vs new tree
└──────────────┬──────────────────┘
               │  Via JSI (direct, synchronous)
               ▼
┌─────────────────────────────────┐
│   Native UI Thread              │  ← Updates real native widget
│                                 │    UIButton/android.Button redraws
└─────────────────────────────────┘
               │
               ▼
        SCREEN UPDATES

Total bridge crossings: 0 (JSI is direct, not a bridge)
Latency: Dramatically reduced, synchronous paths available
```

---

### Flow 3: Tapping a Button in Flutter

```
USER TAPS BUTTON ON SCREEN
         │
         ▼
┌──────────────────────────────────────┐
│   Flutter's own touch detection      │  ← Flutter, NOT the OS, handles
│   (GestureDetector / HitTesting)     │    touch events via its own
│                                      │    event system on the canvas
└─────────────────┬────────────────────┘
                  │  Event dispatched to Flutter's gesture system
                  ▼
┌──────────────────────────────────────┐
│   Dart Code (pre-compiled to         │  ← onPressed() runs as
│   native machine code via AOT)       │    raw native CPU instructions
│                                      │    No interpreter needed
│   onPressed: () {                    │
│     setState(() { count++; }); }     │
└─────────────────┬────────────────────┘
                  │  setState() marks widget as dirty
                  ▼
┌──────────────────────────────────────┐
│   Flutter Framework (Dart layer)     │
│                                      │
│   Widget Tree → rebuilt              │  ← Flutter rebuilds only the
│   Element Tree → reconciled          │    affected widgets, like React's
│   RenderObject Tree → updated        │    virtual DOM diffing
└─────────────────┬────────────────────┘
                  │  Paint instructions generated
                  ▼
┌──────────────────────────────────────┐
│   Skia / Impeller Renderer           │  ← Issues draw calls:
│   (C++ graphics engine)             │    "Draw rounded rect at x,y"
│                                      │    "Draw text 'Submit'"
│                                      │    "Apply shadow"
└─────────────────┬────────────────────┘
                  │  GPU commands
                  ▼
┌──────────────────────────────────────┐
│   GPU                                │  ← Renders pixels directly
│                                      │    to the screen surface
│   (OS only provided the canvas)      │    No native widgets involved
└──────────────────────────────────────┘
                  │
                  ▼
          SCREEN UPDATES

Native OS widgets involved: ZERO
Control over rendering: TOTAL
```

---

### Flow 4: Tapping a Button in a Swift SDK Android App

The Swift SDK flow has a structural difference from the other three: the translation has already happened before the user ever opens the app. There is no runtime interpretation, no bridge, no rendering engine. The conversation between Swift and Android is resolved at build time, not tap time.

```
─────────────────────── BUILD TIME (on your dev machine) ───────────────────────

Swift source code (business logic, models, handlers)
         │
         ▼
┌──────────────────────────────────────────────────┐
│   Swift LLVM Cross-Compiler                      │
│   (Host Toolchain + Swift SDK + Android NDK)     │  ← Three-component
│                                                  │    toolchain working
│   $ swift build                                  │    in concert
│     --swift-sdk aarch64-unknown-linux-android28  │
└──────────────────────────────────────────────────┘
         │
         ▼
  libMySwiftLibrary.so (ARM64 ELF binary)
  ← Native machine code. No VM. No interpreter.
         │
         ▼
┌──────────────────────────────────────────────────┐
│   swift-java jextract                            │  ← Auto-generates
│                                                  │    Java wrappers +
│   Swift structs/classes → Java wrapper classes   │    Swift @_cdecl exports
│   Swift functions → native method declarations   │
└──────────────────────────────────────────────────┘
         │
         ▼
  .so files + Java bindings bundled into .apk via Gradle

──────────────────────── RUNTIME (on the Android device) ────────────────────────

USER TAPS BUTTON ON SCREEN
         │
         ▼
┌──────────────────────────────────────────────────┐
│   Android OS                                     │  ← Real native widget
│   Jetpack Compose / android.widget.Button        │    detects the tap
└─────────────────────┬────────────────────────────┘
                      │  Kotlin onClickListener fires
                      ▼
┌──────────────────────────────────────────────────┐
│   Kotlin / Java UI Layer                         │  ← Your Android UI code
│                                                  │    handles the event
│   button.setOnClickListener {                    │
│       val result = mySwiftLib.handleTap()        │  ← Calls into Swift
│       label.text = result                        │
│   }                                              │
└─────────────────────┬────────────────────────────┘
                      │  JNI call (C ABI, direct, no serialization)
                      ▼
┌──────────────────────────────────────────────────┐
│   Swift Business Logic (.so loaded in memory)    │  ← Runs as raw
│                                                  │    ARM64 machine code
│   func handleTap() -> String {                   │    ARC manages memory
│       state.incrementCount()                     │
│       return "Tapped \(state.count) times"       │
│   }                                              │
└─────────────────────┬────────────────────────────┘
                      │  Return value via JNI
                      ▼
┌──────────────────────────────────────────────────┐
│   Kotlin / Java UI Layer                         │  ← Updates real native
│                                                  │    Android widget
│   label.text = "Tapped 1 times"                  │    OS redraws it
└──────────────────────────────────────────────────┘
                      │
                      ▼
               SCREEN UPDATES

Runtime bridge crossings: 1 (one direct JNI call, no serialization)
Translation cost: ZERO (all translation done at build time)
Widget type: Real native Android widget ✓
```

---

## Part 4: Practical Rendering Examples

### The Component: A Simple Blue Button with "Submit" Text

Let's trace what actually happens in each system when this button is first shown on screen.

---

### React Native Paper: The Button is a Native Widget in Disguise

You write this in JavaScript:

```javascript
<TouchableOpacity style={{ backgroundColor: "blue", padding: 10 }}>
  <Text style={{ color: "white" }}>Submit</Text>
</TouchableOpacity>
```

What actually happens: React Native's JS layer processes this component tree and generates a JSON message for the bridge. On the other side, the native module receives the message and creates a real `UIView` with a `UILabel` child on iOS, or a `ViewGroup` with a `TextView` on Android. These are genuine, 100% native OS objects sitting in memory, managed by the OS. The OS draws them using its own rendering pipeline. If you opened the iOS view hierarchy debugger, you'd see real UIKit objects.

The button's blue background, rounded corners, and text are all rendered by the operating system using its standard drawing routines. This is why a React Native Paper app, when built well, feels genuinely native — because it _is_ native under the hood.

---

### React Native Fabric: The Same Result, Built Better

You write the exact same JSX code. The user sees the exact same button.

The difference is entirely internal. Instead of JSON crossing a bridge, Fabric creates a C++ `ShadowNode` representing your button. Yoga calculates the layout in C++. The result is committed to the native thread via JSI with a direct function call. A real native `UIView`/`ViewGroup` is still created — Fabric doesn't change what the end result looks like to users. It changes how instructions get from your code to the OS.

Think of it as the same house being built, but with better communication between the architect and the construction crew — no more messages getting lost in translation or waiting in a queue.

---

### Flutter: The Button is a Drawing, Not a Widget

You write this in Dart:

```dart
ElevatedButton(
  onPressed: () {},
  child: Text('Submit'),
)
```

What actually happens: Flutter's `ElevatedButton` is not a native button. It's a Dart class that knows how to describe itself geometrically. When Flutter renders it, the `RenderObject` for `ElevatedButton` calculates: "I'm 88dp wide, 36dp tall, I have a Material shadow at elevation 2, my background is a filled rounded rectangle with radius 4, and my child text is centered."

Impeller then receives a series of drawing instructions: draw a rounded rectangle with color `#6200EE`, then draw the string "Submit" in white using the Roboto font, centered at these coordinates. The GPU executes those instructions and paints the pixels directly onto the screen.

If you opened a screen inspector tool, you'd see a single `FlutterView` — one big canvas. There's no button object inside it. There are just pixels that look exactly like a button.

This is why Flutter is extraordinarily powerful for custom UIs — you can make a button look _exactly_ how you want, on every platform, because you're working at the pixel level. It's also why integrating certain platform-specific UI elements (like a native map view or a native video player) requires special `PlatformView` wrappers — you're essentially embedding a native window inside Flutter's canvas, which introduces its own complexity.

---

### Swift SDK for Android: The Button is Native. Swift is Invisible.

You write this in Swift (the shared logic layer):

```swift
// Swift — shared business logic
func handleSubmit(formData: FormData) -> SubmitResult {
    guard formData.isValid else {
        return .failure("Please fill in all required fields")
    }
    return apiClient.submitForm(formData)
}
```

And this in Kotlin (the Android UI layer):

```kotlin
// Kotlin — Android UI, owns the button
val submitButton = Button(context).apply {
    text = "Submit"
    setBackgroundColor(Color.BLUE)
    setOnClickListener {
        val result = swiftLib.handleSubmit(formData)  // JNI call into Swift
        statusLabel.text = result.message
    }
}
```

What actually happens: the `Button` object is a genuine `android.widget.Button` — the same object that a pure Kotlin app would create. Android draws it using its own rendering pipeline, complete with native ripple animations, system accessibility support, and platform-appropriate theming. If you opened the Android view hierarchy inspector, you would see a real Android `Button` object with no indication that Swift is involved.

When the button is tapped and `swiftLib.handleSubmit()` is called, the Kotlin runtime makes a JNI call into the loaded `.so` library. Swift executes `handleSubmit`, which runs as compiled ARM64 machine code — no interpreter, no VM warmup, no serialization. The result is returned to Kotlin as a plain value, and Kotlin updates the UI.

The key insight about rendering: in the Swift SDK model, the "Submit" button's blue background, rounded corners, text, and touch feedback are **entirely the OS's responsibility**. Swift never sees the button. The separation of concerns is total — Swift owns the logic of what to do; Android owns the pixels of how it looks.

---

## Summary Comparison Table

```
┌─────────────────────┬────────────────────┬────────────────────┬────────────────────┬──────────────────────────┐
│                     │  RN Paper          │  RN Fabric         │  Flutter           │  Swift SDK for Android   │
├─────────────────────┼────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Language Runtime    │ JS interpreted     │ JS interpreted     │ Dart AOT compiled  │ Swift AOT compiled       │
│                     │ by Hermes          │ by Hermes          │ to machine code    │ to machine code via LLVM │
│                     │                    │                    │ + Flutter engine   │ No engine bundled        │
├─────────────────────┼────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Cross-language      │ Async JSON bridge  │ Synchronous JSI    │ N/A (no JS)        │ JNI (C ABI, synchronous, │
│ Communication       │ (slow, serialized) │ (fast, direct)     │                    │ no serialization)        │
├─────────────────────┼────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ When Translation    │ Runtime            │ Runtime            │ Build time (AOT)   │ Build time (AOT)         │
│ Happens             │ (every interaction)│ (every interaction)│ + runtime GPU calls│ JNI call at runtime      │
│                     │                    │                    │                    │ is thin and direct       │
├─────────────────────┼────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ UI Rendering        │ Real native OS     │ Real native OS     │ Flutter's own      │ Real native OS widgets   │
│                     │ widgets            │ widgets            │ GPU renderer       │ Swift does not render UI │
├─────────────────────┼────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Shared Layer        │ UI + Logic         │ UI + Logic         │ UI + Logic         │ Logic only               │
│ Covers              │                    │                    │                    │ UI is per-platform       │
├─────────────────────┼────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Layout Engine       │ Yoga (JS thread)   │ Yoga (C++ thread)  │ Flutter's own      │ Platform's own           │
│                     │                    │                    │ layout system      │ (Compose / XML)          │
├─────────────────────┼────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ OS Widget Usage     │ Yes                │ Yes                │ No                 │ Yes (UI is fully native) │
├─────────────────────┼────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Pixel Control       │ Low (OS controls)  │ Low (OS controls)  │ Total              │ Low (OS controls)        │
├─────────────────────┼────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Platform Fidelity   │ High (uses real    │ High (uses real    │ Consistent across  │ Maximum — fully native   │
│                     │ native widgets)    │ native widgets)    │ platforms, but     │ platform widgets on both │
│                     │                    │                    │ Flutter-drawn      │ iOS and Android          │
├─────────────────────┼────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Memory Management   │ JS GC              │ JS GC              │ Dart GC            │ ARC (Swift) bridged to   │
│                     │                    │                    │                    │ GC (Java) via JNI handle │
├─────────────────────┼────────────────────┼────────────────────┼────────────────────┼──────────────────────────┤
│ Closest Parallel    │ Web in a shell     │ Optimized web      │ Game engine model  │ Kotlin Multiplatform     │
│                     │                    │ in a shell         │                    │ (logic shared, UI native)│
└─────────────────────┴────────────────────┴────────────────────┴────────────────────┴──────────────────────────┘
```

---

## Closing Thought

The core philosophical divide runs deeper than three options. React Native (both Paper and Fabric) asks the question, _"how do we talk to the native platform as efficiently as possible?"_ Flutter asks a different question entirely: _"what if we didn't need to talk to the native platform at all?"_

The Swift SDK for Android asks a third question that neither of those approaches considers: _"what if we didn't need a translation layer at all — because our code becomes native before it ever arrives?"_

Neither answer is universally better. React Native gives you platform authenticity and a familiar web-developer mental model. Flutter gives you pixel-perfect cross-platform consistency and a performance floor that is very hard to crack through. The Swift SDK gives you the deepest native integration possible — real platform widgets, compiled machine code, no runtime engine overhead — at the cost of writing platform UI twice, once in SwiftUI for iOS and once in Jetpack Compose for Android. Understanding these distinctions is the foundation for every practical decision you will make when choosing between them.
