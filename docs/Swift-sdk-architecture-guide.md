# Swift SDK for Android — A Beginner's Guide

## Preface: The Central Question

You already know how an iPhone app is built — in Swift. You already know how an Android app is built — in Kotlin or Java. But what happens when you want the same app on both platforms without writing everything twice?

This guide answers that question for the Swift SDK for Android. Unlike Flutter or React Native, the Swift SDK takes a fundamentally different approach — one that is worth understanding from the ground up, even if you have never written a line of code.

---

## Part 1: The Analogy

### The Shared Recipe Book

Imagine a restaurant chain that wants to open two locations: one in Paris and one in Tokyo. The head office has spent years developing recipes — the core formulas for taste, ingredients, and proportions. These recipes are the business's most valuable asset.

Now the question is: how does each location use them?

**Option A** — Send one chef who works in both kitchens, translating every instruction in real time. (This is React Native.)

**Option B** — Build a completely new, third kitchen that doesn't follow either French or Japanese conventions — it runs entirely on its own system. (This is Flutter.)

**Option C** — Write the recipes once in a master book, then send a copy to Paris and a copy to Tokyo. Each local chef reads the same recipes but cooks using their own local ingredients, local equipment, and local presentation. The food tastes consistent. The experience in each restaurant feels completely local. (This is the Swift SDK.)

The **master recipe book** is your Swift code — the logic, the calculations, the rules, the data. The **Paris kitchen** is your iPhone app. The **Tokyo kitchen** is your Android app. Each kitchen builds its own dining room (the UI) using local materials. Only the recipes travel.

> The Swift SDK does not give you one app that runs everywhere. It gives you one brain that powers two separate, fully native apps.

---

## Part 2: What the Swift SDK Actually Is

### The Three Required Ingredients

Cross-compiling Swift for Android requires three pieces working together. None of them can be skipped.

**1. The Host Toolchain**

This is the Swift compiler installed on your development machine — the tool that reads your `.swift` files and turns them into instructions a phone can execute. The compiler runs on your laptop, not on the phone. Like a printing press that produces books before they are shipped to libraries.

**2. The Swift SDK for Android**

This is a bundle of pre-built resources that teaches the compiler how to produce Android-compatible output. Without it, the compiler only knows how to produce iPhone or Mac binaries. With it, the compiler gains a new target — Android — and knows exactly what format, what system libraries, and what conventions Android expects.

**3. The Android NDK (Native Development Kit)**

This is Google's official standards manual for native Android code. It contains the system headers and tools that any native library must conform to in order to run on Android. The Swift SDK uses it as the final verification step — confirming that the compiled Swift output meets Android's requirements before it is packaged into an app.

When these three work together, the output is a `.so` file (a shared library) — a compiled, phone-ready binary that Android loads and runs as if it were written in C++ or Kotlin Native. The phone never knows Swift was involved.

---

## Part 3: Source-to-Native Translation

### How Swift Code Becomes an Android App

Unlike React Native (which interprets JavaScript at runtime) and Flutter (which ships a Dart runtime engine inside the app), the Swift SDK compiles everything **before the app is installed**. This is called Ahead-of-Time (AOT) compilation.

Here is what happens at each stage:

**Stage 1 — You write Swift**

Your Swift code describes the app's logic: what happens when a user logs in, how data is fetched from the internet, how a form is validated, what to do when an error occurs. None of this code mentions Android, buttons, or screens. It is pure logic.

**Stage 2 — The compiler translates**

You run a single build command. The three-component toolchain reads your Swift source files and produces ARM64 machine code — raw CPU instructions — packaged as a `.so` shared library. The translation is complete. There is no interpreter, no virtual machine, no runtime engine. Just compiled instructions.

**Stage 3 — Gradle bundles the library**

The `.so` file is placed into the Android project's `jniLibs/` folder. Gradle (Android's build system) picks it up and bundles it into the final `.apk` (the app file that gets installed on a phone).

**Stage 4 — The app loads Swift at runtime**

When the Android app starts, it calls `System.loadLibrary("MySwiftLibrary")`. This loads the compiled Swift binary into memory. From this point forward, any Kotlin or Java code in the app can call Swift functions directly — with no bridge, no serialization, and no message queue.

**Stage 5 — Kotlin calls Swift through JNI**

JNI (Java Native Interface) is Android's built-in mechanism for connecting Java/Kotlin code with native compiled code. It is not a React Native invention or a Swift invention — it is how Android has always worked with C and C++ libraries. Swift, compiling to the same binary format, plugs into the same socket. The call is synchronous and direct. No translation happens at runtime because all translation happened at compile time.

---

## Part 4: Visual Flow Diagrams

### Flow 1: Building the Swift Library for Android (Build Time)

```
Your Swift source files (.swift)
         |
         v
+-------------------------------------------+
|   Swift LLVM Compiler                     |  <- Reads Swift source
|   + Swift SDK for Android (sysroot)       |     Applies Android rules
|   + Android NDK (system headers + linker) |     Links system libraries
+-------------------------------------------+
         |
         v
  libMyApp.so  (ARM64 machine code)
  <- No interpreter. No VM. Raw CPU instructions.
         |
         v
+-------------------------------------------+
|   swift-java jextract                     |  <- Reads your Swift API
|                                           |     Generates Java wrappers
|   Swift class -> Java class               |     and Swift @_cdecl exports
|   Swift function -> native method         |     automatically
+-------------------------------------------+
         |
         v
  Java wrapper classes  +  Swift @_cdecl stubs
         |
         v
+-------------------------------------------+
|   Gradle                                  |  <- Packages everything
|   jniLibs/arm64-v8a/libMyApp.so           |     into the final .apk
|   + generated Java wrappers               |
+-------------------------------------------+
         |
         v
      App installed on Android device
```

---

### Flow 2: Tapping a Button at Runtime (Android)

```
USER TAPS THE LOGIN BUTTON
         |
         v
+-------------------------------------------+
|   Android OS                              |  <- Real native widget
|   android.widget.Button / Compose Button  |     detects the tap
+-------------------------------------------+
         |
         |  Kotlin onClick listener fires
         v
+-------------------------------------------+
|   Kotlin UI Layer                         |  <- Your Android UI code
|                                           |
|   loginButton.setOnClickListener {        |
|       val result =                        |
|           swiftLib.login(email, password) |  <- Calls into Swift
|       showResult(result)                  |
|   }                                       |
+-------------------------------------------+
         |
         |  JNI call  (direct, synchronous, no serialization)
         v
+-------------------------------------------+
|   Swift Logic (.so loaded in memory)      |  <- Runs as compiled
|                                           |     ARM64 machine code
|   func login(                             |     ARC manages memory
|       email: String,                      |
|       password: String                    |
|   ) -> LoginResult {                      |
|       guard isValidEmail(email) else {    |
|           return .failure("Bad email")    |
|       }                                   |
|       return api.authenticate(            |
|           email, password                 |
|       )                                   |
|   }                                       |
+-------------------------------------------+
         |
         |  Result returned via JNI
         v
+-------------------------------------------+
|   Kotlin UI Layer                         |  <- Updates real native
|   showResult(result)                      |     Android UI components
+-------------------------------------------+
         |
         v
      SCREEN UPDATES

Runtime bridge crossings : 1 direct JNI call (no serialization)
Translation cost at runtime : ZERO (all done at build time)
Widget type : Real native Android widget
```

---

### Flow 3: The Same Swift Logic on iOS (No JNI Needed)

```
USER TAPS THE LOGIN BUTTON
         |
         v
+-------------------------------------------+
|   iOS — SwiftUI Button                    |  <- Real native iOS widget
+-------------------------------------------+
         |
         |  SwiftUI action closure fires
         v
+-------------------------------------------+
|   SwiftUI View                            |  <- Your iOS UI code
|                                           |
|   Button("Login") {                       |
|       let result = userService.login(     |  <- Direct Swift call
|           email: email,                   |     No JNI. No bridge.
|           password: password              |     Same function as Android.
|       )                                   |
|       handleResult(result)                |
|   }                                       |
+-------------------------------------------+
         |
         |  Direct function call (native Swift)
         v
+-------------------------------------------+
|   Swift Logic (same source as Android)   |  <- Runs natively
|                                           |     iOS is Swift's home
|   func login(                             |     No compilation ceremony
|       email: String,                      |     No extra toolchain
|       password: String                    |
|   ) -> LoginResult { ... }               |
+-------------------------------------------+
         |
         v
      SCREEN UPDATES

Runtime bridge crossings : 0
Translation cost at runtime : ZERO
Widget type : Real native iOS widget
```

---

## Part 5: Practical Code Examples

### The Component: A Login Form

Let's trace what actually happens in each layer when a user fills in an email and password and taps "Login."

---

### Layer 1 — The Swift Logic (Written Once, Shared Everywhere)

This code lives in a Swift Package. It knows nothing about buttons or screens. It only knows about users, validation, and authentication.

```swift
// Sources/AppLogic/UserService.swift

public struct LoginResult {
    public let success: Bool
    public let message: String

    public init(success: Bool, message: String) {
        self.success = success
        self.message = message
    }
}

public class UserService {

    public init() {}

    public func login(email: String, password: String) -> LoginResult {
        guard email.contains("@") else {
            return LoginResult(success: false, message: "Invalid email address")
        }
        guard password.count >= 8 else {
            return LoginResult(success: false, message: "Password too short")
        }
        // In a real app: call an API, check credentials
        return LoginResult(success: true, message: "Welcome back!")
    }
}
```

This is the **master recipe book** from the analogy. It is compiled once into the `.so` library. Both platforms use the same `login()` function with the same validation rules. Fix a bug here, and both apps get the fix.

---

### Layer 2a — Android UI (Kotlin, Written for Android Only)

The Kotlin code calls into Swift through the auto-generated JNI wrappers. From Kotlin's perspective, `UserService` looks like a normal Kotlin class.

```kotlin
// app/src/main/java/com/myapp/LoginScreen.kt

class LoginActivity : AppCompatActivity() {

    companion object {
        init {
            // Load the compiled Swift library once at startup
            System.loadLibrary("AppLogic")
        }
    }

    private val userService = UserService()  // Swift object via JNI

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_login)

        val emailField = findViewById<EditText>(R.id.email)
        val passwordField = findViewById<EditText>(R.id.password)
        val loginButton = findViewById<Button>(R.id.loginButton)  // Real Android Button
        val messageView = findViewById<TextView>(R.id.message)

        loginButton.setOnClickListener {
            // This line crosses the JNI boundary into Swift
            val result = userService.login(
                email = emailField.text.toString(),
                password = passwordField.text.toString()
            )
            messageView.text = result.message
        }
    }
}
```

The `Button` on screen is a genuine `android.widget.Button`. Android draws it, animates it, and handles its accessibility. Swift never sees it. Swift only handles what happens after the tap: the validation logic inside `login()`.

---

### Layer 2b — iOS UI (SwiftUI, Written for iOS Only)

The SwiftUI code calls the same Swift `UserService` directly — no JNI, no bridge, no library loading. It is a normal Swift function call.

```swift
// iOS/LoginView.swift

import SwiftUI
import AppLogic  // The shared Swift package — linked directly on iOS

struct LoginView: View {

    private let userService = UserService()  // Same Swift class, used directly
    @State private var email = ""
    @State private var password = ""
    @State private var message = ""

    var body: some View {
        VStack(spacing: 16) {
            TextField("Email", text: $email)
                .textFieldStyle(.roundedBorder)
                .keyboardType(.emailAddress)

            SecureField("Password", text: $password)
                .textFieldStyle(.roundedBorder)

            Button("Login") {
                // Direct Swift function call — no JNI, no ceremony
                let result = userService.login(
                    email: email,
                    password: password
                )
                message = result.message
            }
            .buttonStyle(.borderedProminent)

            Text(message)
        }
        .padding()
    }
}
```

The `Button("Login")` on screen is a genuine iOS `UIButton` rendered by SwiftUI. iOS draws it, applies the system font, handles Dynamic Type, and respects the user's accessibility settings. The button looks and feels exactly like every other iOS button — because it is one.

---

### What the User Experiences

Neither user knows Swift was involved. The Android user sees a button that behaves exactly like every other Android button. The iPhone user sees a button that behaves exactly like every other iPhone button. The login validation — the rule that an email must contain `@` and a password must be at least 8 characters — is identical on both, because it runs from the same compiled Swift function.

---

## Part 6: iOS vs Android — The Key Differences

| Dimension | iPhone (iOS) | Android |
|---|---|---|
| Swift's position | Native language of the platform | Compiled visitor, loaded as a library |
| How UI connects to Swift | Direct function call | JNI — Android's native code connector |
| Extra toolchain needed | None — Xcode handles everything | Swift SDK bundle + Android NDK r27d |
| JNI wrappers needed | No | Yes — auto-generated by `swift-java` |
| Memory management | ARC throughout, no seam | ARC (Swift) bridged to GC (Java/Kotlin) |
| Build output | `.framework` or static library | `.so` shared library |
| User experience | Fully native iOS | Fully native Android |

### The Memory Management Difference

This is the one technical nuance worth understanding at a conceptual level.

Swift uses a system called **ARC** (Automatic Reference Counting) to manage memory. When a Swift object is no longer needed, ARC frees its memory immediately.

Java and Kotlin use a **Garbage Collector** — a background process that periodically sweeps through memory and cleans up objects that are no longer in use. It does not free memory immediately.

These two systems have to co-exist when Swift objects are used from Kotlin. The `swift-java` tool handles this automatically: Swift owns the actual memory for Swift objects. Kotlin holds only a lightweight handle (a number that points to the Swift object). When Kotlin's garbage collector determines the handle is no longer needed, it notifies Swift, which then frees the object immediately using ARC. The developer does not manage any of this by hand.

On iOS, this problem does not exist — Swift talks to Swift, and ARC operates end-to-end without any seam.

---

## Part 7: What You Write Once vs What You Write Twice

### Written once in Swift — shared across both platforms

| Category | Example |
|---|---|
| Authentication logic | Email validation, password rules, token management |
| Data fetching | API calls, response parsing, error handling |
| Business rules | "Max 3 free orders per month", "discount expires after 30 days" |
| Data models | User, Product, Order — the structures that represent your data |
| State management | What the app remembers, how it updates when data changes |
| Calculations | Currency conversion, date formatting, sorting and filtering |

### Written separately — once for iOS, once for Android

| Category | iOS | Android |
|---|---|---|
| Screens and layouts | SwiftUI / UIKit | Jetpack Compose / XML |
| Navigation | NavigationStack, TabView | NavController, BottomNavigation |
| Animations | SwiftUI animations, UIKit animators | Compose animations, Motion |
| Platform features | Face ID, ShareSheet, Widgets | Back gesture, App Shortcuts, Widgets |
| Look and feel | Human Interface Guidelines | Material Design |

The rule of thumb: if it is something the user **sees or touches**, it belongs to the platform. If it is something the app **thinks or calculates**, it belongs in the shared Swift layer.

---

## Summary Comparison Table

```
+---------------------+--------------------+--------------------+--------------------+----------------------------+
|                     |  RN Paper          |  RN Fabric         |  Flutter           |  Swift SDK for Android     |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| Language            | JavaScript         | JavaScript         | Dart               | Swift                      |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| Runtime on device   | Hermes VM (JIT)    | Hermes VM (JIT)    | Dart AOT           | AOT machine code           |
|                     |                    |                    | + Flutter engine   | No VM, no engine           |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| Cross-language      | Async JSON bridge  | Synchronous JSI    | N/A                | JNI (direct, synchronous,  |
| communication       | (slow, serialized) | (fast, direct)     |                    | no serialization)          |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| When translation    | Runtime            | Runtime            | Build time (AOT)   | Build time (AOT)           |
| happens             | (every interaction)| (every interaction)| + GPU at runtime   | JNI call is thin           |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| UI rendering        | Real native OS     | Real native OS     | Flutter's own      | Real native OS widgets     |
|                     | widgets            | widgets            | GPU renderer       | Swift does not render UI   |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| Shared layer covers | UI + Logic         | UI + Logic         | UI + Logic         | Logic only                 |
|                     |                    |                    |                    | UI is per-platform         |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| OS widget usage     | Yes                | Yes                | No                 | Yes                        |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| Memory management   | JS GC              | JS GC              | Dart GC            | ARC (Swift) bridged to     |
|                     |                    |                    |                    | GC (Java) via JNI handle   |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| Closest parallel    | Web in a shell     | Optimized web      | Game engine model  | Kotlin Multiplatform       |
|                     |                    | in a shell         |                    | (logic shared, UI native)  |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| iOS story           | React → bridge     | React → JSI        | Flutter engine     | Swift is native iOS        |
|                     |                    |                    | on iOS             | Direct link, no JNI        |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| Status (2025)       | Stable             | Stable (default)   | Stable             | Nightly preview            |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
```

---

## Closing Thought

React Native (both Paper and Fabric) asks: *"how do we talk to the native platform as efficiently as possible?"* Flutter asks: *"what if we didn't need to talk to the native platform at all?"*

The Swift SDK asks a third question that neither of those approaches considers: *"what if the translation happened before we arrived, so there is nothing to bridge at runtime?"*

The tradeoff is real. You write more code overall — the UI must be built separately for each platform. But every user on every platform gets an app that behaves exactly as they expect, using exactly the components their platform provides, with no approximation and no layer of indirection between the user and the native experience.

---

## Further Reading

- [Swift SDK for Android — Official Announcement](https://www.swift.org/blog/nightly-swift-sdk-for-android/)
- [Getting Started with the Swift SDK for Android](https://www.swift.org/documentation/articles/swift-sdk-for-android-getting-started.html)
- [swift-java — Java & Swift Interoperability](https://github.com/swiftlang/swift-java)
- [Swift for Android Examples](https://github.com/swiftlang/swift-android-examples)
- [Android Workgroup](https://www.swift.org/android-workgroup/)
- [Swift Forums — Android Category](https://forums.swift.org/c/platform/android/115)
