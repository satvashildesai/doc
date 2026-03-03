# Kotlin Multiplatform — A Beginner's Guide

## Preface: The Central Question

You already know how an Android app is built — in Kotlin. You already know how an iPhone app is built — in Swift. But what happens when you want the same app on both platforms without writing everything twice?

This guide answers that question for Kotlin Multiplatform (KMP). Unlike Flutter or React Native, Kotlin Multiplatform takes a philosophy of sharing only what is worth sharing and letting each platform be itself. Understanding how it works — not just that it works — is the foundation for using it well.

---

## Part 1: The Analogy

### The Headquarters and the Local Offices

Imagine a company with a headquarters in one city and offices in Paris and Tokyo. The headquarters writes all the company policies: how employees are evaluated, how budgets are calculated, how customers are handled, what counts as a successful outcome. These policies apply equally to both offices.

But each office runs its own daily operations. The Paris office decorates its walls, chooses its furniture, and greets clients in French. The Tokyo office does the same in Japanese. Neither office rents its lobby from headquarters. Each one builds its own front-of-house experience using local conventions, local materials, and local taste.

The **headquarters** is your `commonMain` — the shared Kotlin code. The **Paris office** is your Android app. The **Tokyo office** is your iOS app. Headquarters ships policy documents; each office implements them in whatever way fits their local environment.

> Kotlin Multiplatform does not give you one app that runs everywhere. It gives you one set of rules that both apps follow — implemented natively on each platform.

### The Promise and the Fulfilment

There is one more layer to the analogy. Sometimes a policy from headquarters mentions something that works differently in each country — tax rules, for example. The policy document says: "calculate the applicable tax." It does not specify how. Each office fills in that blank according to local law.

This is the `expect` / `actual` mechanism — the single most important concept in Kotlin Multiplatform. The shared code makes a **promise** (`expect`): "there will be a function called `getPlatformName` and it will return a String." Each platform makes a **fulfilment** (`actual`): "here is the concrete implementation using Android's or iOS's own APIs."

The compiler enforces the contract. If you write an `expect` declaration and forget to provide an `actual` on one platform, the project will not compile. The promise must be kept on every platform.

---

## Part 2: What Kotlin Multiplatform Actually Is

### The Source Set Architecture

A Kotlin Multiplatform project is organised around **source sets** — named folders of Kotlin source files, each compiled to a specific target.

```
shared/
  src/
    commonMain/        <- Compiled to ALL platforms
      kotlin/
        UserService.kt
        LoginResult.kt
    androidMain/       <- Compiled to Android only
      kotlin/
        Platform.android.kt
    iosMain/           <- Compiled to iOS only
      kotlin/
        Platform.ios.kt
```

**`commonMain`** is the headquarters. Code here must work on every declared target. It cannot import Android SDK classes, iOS frameworks, or anything platform-specific. It can only use Kotlin's standard library and multiplatform-compatible libraries (Ktor, Coroutines, SQLDelight, etc.).

**`androidMain`** is the Android office. Code here can use the full Android SDK — `android.os`, `android.content`, Jetpack libraries, anything Gradle provides.

**`iosMain`** is the iOS office. Code here can use iOS frameworks through Kotlin/Native's automatic Objective-C bindings — `platform.UIKit`, `platform.Foundation`, and so on.

### How Compilation Works

When Gradle builds a KMP project, it invokes the Kotlin compiler twice — once per target — and each compilation selects a different set of source files.

```
Android build:
  commonMain + androidMain  ->  .class files (JVM bytecode)
                                bundled as an Android library (.aar)

iOS build:
  commonMain + iosMain      ->  native binary (via Kotlin/Native + LLVM)
                                packaged as an .xcframework
                                importable from Swift and Xcode
```

For Android, the output is standard JVM bytecode — identical to any other Kotlin library. The Android runtime runs it directly with no additional steps.

For iOS, the Kotlin/Native compiler uses the LLVM toolchain — the same backend used by Swift and Clang — to produce a native binary. There is no JVM on iOS, no interpreter, and no virtual machine. The output is compiled ARM64 machine code wrapped in an `.xcframework` that Xcode can import as if it were any other Apple framework.

### The expect / actual Mechanism

This is the escape hatch that allows shared code to describe platform-specific behaviour without implementing it.

In `commonMain`, you declare a contract using the `expect` keyword:

```kotlin
// commonMain/Platform.kt
expect fun getPlatformName(): String
```

In `androidMain`, you fulfill the contract using the `actual` keyword:

```kotlin
// androidMain/Platform.android.kt
import android.os.Build
actual fun getPlatformName(): String = "Android ${Build.VERSION.SDK_INT}"
```

In `iosMain`, you fulfill the same contract differently:

```kotlin
// iosMain/Platform.ios.kt
import platform.UIKit.UIDevice
actual fun getPlatformName(): String =
    UIDevice.currentDevice.systemName()
```

When the compiler builds for Android, it merges `getPlatformName` from `commonMain` and `androidMain` into a single function. When it builds for iOS, it merges the `commonMain` version with the `iosMain` version. The `expect` declaration and its corresponding `actual` must live in the same package — they are literally merged into one declaration in the output.

---

## Part 3: Source-to-Native Translation

### How Kotlin Code Becomes an App on Each Platform

**Stage 1 — You write shared Kotlin in `commonMain`**

Your shared code describes logic that applies on both platforms: validation rules, API calls, data models, state management. It uses only the Kotlin standard library and multiplatform libraries. It knows nothing about Android Views, UIKit, or any platform-specific concept.

**Stage 2 — You add platform-specific code where needed**

Where the shared code needs to call something platform-specific — a device identifier, a local file path, a platform notification system — you use `expect` / `actual`. The shared code calls the `expect` function. Each platform provides the `actual` implementation.

**Stage 3 — Gradle compiles each target separately**

For Android: `commonMain` + `androidMain` are compiled together into JVM bytecode, then packaged as a Gradle module that the Android app imports directly.

For iOS: `commonMain` + `iosMain` are compiled together by Kotlin/Native into a `.xcframework`. The iOS Xcode project adds this framework as a build phase dependency. Every time the shared code changes, the framework is recompiled before the iOS app builds.

**Stage 4 — Each platform app uses the shared code natively**

On Android, the shared module is a standard Gradle dependency. Kotlin classes from `commonMain` are available as ordinary Kotlin classes in the Android app. No bridge, no wrapper, no special ceremony.

On iOS, Swift imports the generated framework with `import shared`. Kotlin classes appear as Objective-C compatible types in Swift. Basic types (String, Int, List) map cleanly. Coroutines and Flows require an additional tool called SKIE (Swift Kotlin Interface Enhancer) to expose them as natural Swift async patterns.

---

## Part 4: Visual Flow Diagrams

### Flow 1: Building the Shared Module (Build Time)

```
Your Kotlin source files
  commonMain/UserService.kt
  androidMain/Platform.android.kt
  iosMain/Platform.ios.kt
         |
         v
+------------------------------------------+
|   Kotlin Gradle Plugin                   |
|   reads build.gradle.kts targets         |
|                                          |
|   kotlin {                               |
|       androidTarget()                    |
|       iosX64()                           |
|       iosArm64()                         |
|       iosSimulatorArm64()                |
|   }                                      |
+------------------------------------------+
         |
         +----------------------------+
         |                            |
         v                            v
+------------------+       +-------------------+
|  Android build   |       |  iOS build        |
|                  |       |                   |
|  commonMain      |       |  commonMain       |
|  + androidMain   |       |  + iosMain        |
|                  |       |                   |
|  Kotlin compiler |       |  Kotlin/Native    |
|  -> JVM bytecode |       |  + LLVM           |
|  -> .aar library |       |  -> ARM64 binary  |
+------------------+       |  -> .xcframework  |
         |                 +-------------------+
         v                            v
  Android app imports          Xcode imports
  as Gradle dependency         as framework dependency
```

---

### Flow 2: A Login Request at Runtime on Android

```
USER TAPS LOGIN BUTTON
         |
         v
+------------------------------------------+
|   Android UI (Jetpack Compose)            |
|                                          |
|   Button(onClick = {                     |
|       viewModel.login(email, password)   |
|   }) { Text("Login") }                   |
+------------------------------------------+
         |
         |  Direct Kotlin function call
         |  (no bridge, no serialization)
         v
+------------------------------------------+
|   ViewModel (androidMain or commonMain)  |
|                                          |
|   fun login(email: String,               |
|             password: String) {          |
|       viewModelScope.launch {            |
|           val result =                   |
|               userService.login(         |
|                   email, password        |
|               )                          |
|           _uiState.value = result        |
|       }                                  |
|   }                                      |
+------------------------------------------+
         |
         |  Direct Kotlin call into commonMain
         v
+------------------------------------------+
|   UserService (commonMain)               |
|                                          |
|   suspend fun login(                     |
|       email: String,                     |
|       password: String                   |
|   ): LoginResult {                       |
|       if (!email.contains("@"))          |
|           return LoginResult.Failure(    |
|               "Invalid email"            |
|           )                             |
|       return api.authenticate(           |
|           email, password               |
|       )                                  |
|   }                                      |
+------------------------------------------+
         |
         v
+------------------------------------------+
|   Android UI updates                     |
|   Real native Compose widgets redraw     |
+------------------------------------------+

Bridge crossings : 0
Serialization    : none
Widget type      : Real native Compose widget
```

---

### Flow 3: The Same Login Request at Runtime on iOS

```
USER TAPS LOGIN BUTTON
         |
         v
+------------------------------------------+
|   iOS UI (SwiftUI)                        |
|                                          |
|   Button("Login") {                      |
|       viewModel.login(                   |
|           email: email,                  |
|           password: password             |
|       )                                  |
|   }                                      |
+------------------------------------------+
         |
         |  Swift method call into Kotlin framework
         |  (no JNI, no bridge — direct framework call)
         v
+------------------------------------------+
|   iOS ViewModel (Swift, in iosApp)       |
|   or shared ViewModel (commonMain)       |
|                                          |
|   Calls userService.login()              |
|   via the imported shared framework      |
+------------------------------------------+
         |
         |  Kotlin/Native compiled function call
         v
+------------------------------------------+
|   UserService (commonMain)               |
|   Same source file as Android.           |
|   Compiled to ARM64 native code          |
|   by Kotlin/Native + LLVM.              |
|                                          |
|   suspend fun login(...): LoginResult    |
+------------------------------------------+
         |
         v
+------------------------------------------+
|   iOS UI updates                         |
|   Real native SwiftUI views redraw       |
+------------------------------------------+

Bridge crossings : 0
Serialization    : none
Widget type      : Real native SwiftUI widget
```

---

## Part 5: Practical Code Examples

### The Component: A Login Form

Let's trace what actually happens across all layers when a user fills in an email and password and taps "Login."

---

### Layer 1 — The Shared Kotlin Logic (commonMain, written once)

This code lives in `commonMain`. It cannot import Android or iOS APIs. It only uses Kotlin's own constructs.

```kotlin
// commonMain/kotlin/auth/LoginResult.kt

sealed class LoginResult {
    data class Success(val welcomeMessage: String) : LoginResult()
    data class Failure(val errorMessage: String) : LoginResult()
}
```

```kotlin
// commonMain/kotlin/auth/UserService.kt

class UserService {

    suspend fun login(email: String, password: String): LoginResult {
        if (!email.contains("@")) {
            return LoginResult.Failure("Invalid email address")
        }
        if (password.length < 8) {
            return LoginResult.Failure("Password must be at least 8 characters")
        }
        // In a real app: call an API using Ktor
        return LoginResult.Success("Welcome back!")
    }
}
```

This is the headquarters policy document. It is compiled into both the Android library and the iOS framework. Fix a bug in `login()`, and both platforms get the fix on the next build.

---

### Layer 2 — Platform-Specific Behaviour Using expect / actual

Sometimes shared code needs to do something that works differently per platform — for example, reading a device identifier.

```kotlin
// commonMain/kotlin/platform/DeviceInfo.kt

expect fun getDeviceName(): String
```

```kotlin
// androidMain/kotlin/platform/DeviceInfo.android.kt

import android.os.Build

actual fun getDeviceName(): String = Build.MODEL
```

```kotlin
// iosMain/kotlin/platform/DeviceInfo.ios.kt

import platform.UIKit.UIDevice

actual fun getDeviceName(): String =
    UIDevice.currentDevice.name
```

The `commonMain` code can call `getDeviceName()` freely. The compiler guarantees that a real implementation exists for every declared platform. If you add a new platform target and forget to add the `actual`, the build fails with a clear error.

---

### Layer 3a — Android UI (Jetpack Compose, written for Android only)

```kotlin
// androidApp/src/main/kotlin/LoginScreen.kt

import androidx.compose.runtime.*
import androidx.compose.material3.*
import com.myapp.auth.UserService
import com.myapp.auth.LoginResult

@Composable
fun LoginScreen() {
    val userService = remember { UserService() }
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    var message by remember { mutableStateOf("") }
    val scope = rememberCoroutineScope()

    Column {
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("Email") }
        )
        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("Password") },
            visualTransformation = PasswordVisualTransformation()
        )
        Button(
            onClick = {
                scope.launch {
                    // Direct Kotlin call into commonMain — no bridge
                    message = when (val result = userService.login(email, password)) {
                        is LoginResult.Success -> result.welcomeMessage
                        is LoginResult.Failure -> result.errorMessage
                    }
                }
            }
        ) {
            Text("Login")  // Real Compose Button — Android native
        }
        Text(message)
    }
}
```

The `Button` is a genuine Jetpack Compose composable rendered by Android. The call to `userService.login()` is a direct Kotlin function call — the shared code compiled into the same process. There is no bridge, no serialization, no intermediate format.

---

### Layer 3b — iOS UI (SwiftUI, written for iOS only)

```swift
// iosApp/iosApp/LoginView.swift

import SwiftUI
import shared  // The compiled .xcframework from KMP

struct LoginView: View {

    private let userService = UserService()  // Kotlin class, used from Swift
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
                // Calling Kotlin suspend function from Swift
                // SKIE converts it to a Swift async Task automatically
                Task {
                    let result = try await userService.login(
                        email: email,
                        password: password
                    )
                    if let success = result as? LoginResultSuccess {
                        message = success.welcomeMessage
                    } else if let failure = result as? LoginResultFailure {
                        message = failure.errorMessage
                    }
                }
            }
            .buttonStyle(.borderedProminent)

            Text(message)
        }
        .padding()
    }
}
```

The `Button("Login")` is a genuine SwiftUI button rendered by iOS. The `UserService` instance is a Kotlin class that was compiled to native ARM64 code by Kotlin/Native. Swift calls it through the generated framework interface. With SKIE installed, Kotlin coroutines (`suspend` functions) become Swift `async` functions, and Kotlin `Flow` becomes Swift `AsyncStream`.

---

### What the User Experiences

Neither user knows Kotlin was shared. The Android user sees a Compose button that behaves exactly like every other Compose button. The iOS user sees a SwiftUI button that behaves exactly like every other SwiftUI button. The validation rules — email must contain `@`, password must be at least 8 characters — are identical on both, because they run from the same compiled Kotlin function.

---

## Part 6: Android vs iOS — The Key Differences

| Dimension | Android | iOS |
|---|---|---|
| Kotlin's position | Native language of the platform | Visitor, exposed via a compiled framework |
| Shared code format | JVM bytecode compiled into `.aar` | Native binary compiled into `.xcframework` |
| How the app accesses shared code | Direct Gradle dependency — no ceremony | `import shared` in Swift, framework linked in Xcode |
| Coroutine/Flow interop | Native — Kotlin Coroutines work directly | Requires SKIE or manual wrapping to feel natural in Swift |
| Platform-specific APIs in Kotlin | Full Android SDK available in `androidMain` | iOS Obj-C frameworks available in `iosMain` via K/N bindings |
| Build toolchain | Gradle + Kotlin compiler | Gradle + Kotlin/Native + LLVM + Xcode build phase |
| UI framework | Jetpack Compose (Android) | SwiftUI or UIKit (iOS) |

### The Coroutine Bridging Problem

Kotlin Multiplatform's most significant friction point for iOS developers is how Kotlin's concurrency model (Coroutines, Flow) interacts with Swift's concurrency model (async/await, AsyncStream).

Kotlin `suspend` functions and `Flow` do not automatically become Swift `async` functions and `AsyncStream`. Without extra tooling, a `Flow<Boolean>` in Kotlin becomes an opaque `Kotlinx_coroutines_coreFlow` type in Swift — essentially unusable without writing adapter code by hand.

The standard solution is **SKIE** (Swift Kotlin Interface Enhancer), an open-source Gradle plugin from Touchlab. When SKIE is applied to a KMP project, it rewrites the generated Swift interface so that:

- Kotlin `suspend fun` becomes Swift `async func`
- Kotlin `Flow<T>` becomes Swift `AsyncStream<T>`
- Kotlin sealed classes become Swift enums with associated values

SKIE is not part of the Kotlin Multiplatform compiler itself, but it is the de facto standard for production iOS projects.

---

## Part 7: What You Write Once vs What You Write Twice

### Written once in Kotlin (commonMain) — shared across both platforms

| Category | Example |
|---|---|
| Authentication logic | Email validation, password rules, token management |
| Data fetching | Ktor HTTP client, response parsing, error handling |
| Business rules | "Max 3 free orders per month", "discount expires after 30 days" |
| Data models | User, Product, Order — data classes shared across both apps |
| State management | ViewModel (Jetpack ViewModel supports KMP), StateFlow |
| Local persistence | Room Multiplatform or SQLDelight — same schema on both platforms |
| Serialization | kotlinx.serialization — same JSON parsing on both platforms |

### Written separately — once for Android, once for iOS

| Category | Android | iOS |
|---|---|---|
| Screens and layouts | Jetpack Compose | SwiftUI or UIKit |
| Navigation | NavController, Navigation Compose | NavigationStack, UINavigationController |
| Animations | Compose animations, Motion | SwiftUI animations, UIKit animators |
| Platform features | Notifications, Widgets, Back gesture | Face ID, ShareSheet, App Clips |
| Look and feel | Material Design 3 | Human Interface Guidelines |
| Concurrency wiring | Direct coroutine usage | async/await via SKIE |

The rule of thumb is the same as for the Swift SDK: if it is something the user **sees or touches**, it belongs to the platform. If it is something the app **thinks or calculates**, it belongs in `commonMain`.

---

## Part 8: Compose Multiplatform — The Optional UI Layer

Kotlin Multiplatform for logic sharing is stable and production-ready. There is a separate, optional layer on top of it called **Compose Multiplatform** (developed by JetBrains) that allows the UI to be shared as well.

With Compose Multiplatform, the same Jetpack Compose UI code can run on Android, iOS, desktop, and web. This shifts the architecture from "shared logic, native UI" to "shared logic and shared UI" — the same philosophy as Flutter, but using Kotlin and Compose rather than Dart.

| Approach | What is shared | UI framework |
|---|---|---|
| KMP alone | Logic only | Native per platform (Compose on Android, SwiftUI on iOS) |
| KMP + Compose Multiplatform | Logic + UI | Compose on all platforms |

Compose Multiplatform on iOS became significantly more stable in 2025. It is suitable for many production use cases, though some teams still prefer to keep the iOS UI in SwiftUI for the deepest native feel.

The decision between "KMP logic only" and "KMP + Compose Multiplatform" is a genuine trade-off, not a clear winner. Native SwiftUI gives the most authentic iOS experience. Compose Multiplatform gives the fastest cross-platform delivery.

---

## Summary Comparison Table

```
+---------------------+--------------------+--------------------+--------------------+----------------------------+
|                     |  RN Paper          |  RN Fabric         |  Flutter           |  Kotlin Multiplatform      |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| Language            | JavaScript         | JavaScript         | Dart               | Kotlin                     |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| Runtime on device   | Hermes VM (JIT)    | Hermes VM (JIT)    | Dart AOT           | JVM bytecode (Android)     |
|                     |                    |                    | + Flutter engine   | Native binary via LLVM     |
|                     |                    |                    |                    | (iOS)                      |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| Cross-language      | Async JSON bridge  | Synchronous JSI    | N/A                | Direct Gradle dep          |
| communication       | (slow, serialized) | (fast, direct)     |                    | (Android)                  |
|                     |                    |                    |                    | Framework import (iOS)     |
|                     |                    |                    |                    | No bridge on either side   |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| When translation    | Runtime            | Runtime            | Build time (AOT)   | Build time                 |
| happens             | (every interaction)| (every interaction)| + GPU at runtime   | JVM bytecode or LLVM       |
|                     |                    |                    |                    | native binary              |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| UI rendering        | Real native OS     | Real native OS     | Flutter's own      | Real native OS widgets     |
|                     | widgets            | widgets            | GPU renderer       | (or Compose Multiplatform  |
|                     |                    |                    |                    | if opted in)               |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| Shared layer covers | UI + Logic         | UI + Logic         | UI + Logic         | Logic only by default      |
|                     |                    |                    |                    | UI optional via CMP        |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| OS widget usage     | Yes                | Yes                | No                 | Yes (KMP alone)            |
|                     |                    |                    |                    | No (with CMP)              |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| Platform-specific   | expect/actual not  | expect/actual not  | Platform channels  | expect/actual — first-     |
| APIs                | applicable         | applicable         | (manual wiring)    | class language feature     |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| Closest parallel    | Web in a shell     | Optimized web      | Game engine model  | Swift SDK for Android      |
|                     |                    | in a shell         |                    | (same philosophy,          |
|                     |                    |                    |                    | different home platform)   |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| iOS story           | React -> bridge    | React -> JSI       | Flutter engine     | Kotlin/Native -> LLVM ->   |
|                     |                    |                    | on iOS             | .xcframework -> Swift      |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
| Status (2025)       | Stable             | Stable (default)   | Stable             | Stable (KMP)               |
|                     |                    |                    |                    | Maturing (CMP on iOS)      |
+---------------------+--------------------+--------------------+--------------------+----------------------------+
```

---

## Closing Thought

React Native (both Paper and Fabric) asks: *"how do we talk to the native platform as efficiently as possible?"* Flutter asks: *"what if we didn't need to talk to the native platform at all?"*

Kotlin Multiplatform asks a third question — the same question as the Swift SDK for Android, but from the opposite direction: *"what if the code was already native on both platforms, because we compiled it to whatever each platform requires?"*

The tradeoff is the same as the Swift SDK. You write more total code — the UI must be built separately for each platform, unless you opt into Compose Multiplatform. But every user on every platform gets an app that uses the real tools, the real patterns, and the real components of their platform — not an approximation layered on top.

The difference from the Swift SDK is the home platform. KMP is natural and zero-friction on Android, and a compiled visitor on iOS. The Swift SDK is natural and zero-friction on iOS, and a compiled visitor on Android. Architecturally, they are mirrors of each other.

---

## Further Reading

- [Kotlin Multiplatform — Official Site](https://kotlinlang.org/multiplatform/)
- [Get Started with Kotlin Multiplatform](https://kotlinlang.org/docs/multiplatform/get-started.html)
- [The Basics of KMP Project Structure](https://kotlinlang.org/docs/multiplatform-discover-project.html)
- [Expected and Actual Declarations](https://www.jetbrains.com/help/kotlin-multiplatform-dev/multiplatform-expect-actual.html)
- [Compose Multiplatform](https://www.jetbrains.com/compose-multiplatform/)
- [SKIE — Swift Kotlin Interface Enhancer](https://skie.touchlab.co)
- [KMP on Android Developers](https://developer.android.com/kotlin/multiplatform)
- [Swift SDK for Android — Beginner's Guide](./swift-sdk-beginners-guide.md)
