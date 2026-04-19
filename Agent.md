# AGENTS Guidelines for This Repository

This repository contains a **Java-based Android** application project. When working on the project interactively with an AI coding agent, please follow the guidelines below to ensure architectural consistency, maximum performance, and a smooth development experience.

This project deliberately targets **Java**, not Kotlin. Suggestions and generated code must stay in Java 17. If a library only ships a Kotlin DSL/extension, prefer its Java-compatible API or document the Java equivalent.

## 1. Project Specifications
- **Language:** Java 17 (source and target compatibility). `var`, records, switch expressions, text blocks, pattern matching for `instanceof` are all encouraged.
- **Android SDK:** `minSdk 21`, `targetSdk 35`, `compileSdk 35`.
- **Build System:** Gradle with Android Gradle Plugin 8.x. Groovy DSL (`build.gradle`) is acceptable; Kotlin DSL (`build.gradle.kts`) is preferred for new modules.
- **AndroidX / Jetpack:** Required. No legacy `android.support.*` imports.
- **Java Toolchain:** Configure via `java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }` so the build is reproducible across machines.

## 2. Architecture & Design Patterns
We follow a layered Clean Architecture approach with MVVM on the presentation side:
- **Presentation Layer:** `Activity` / `Fragment` + `ViewModel` + `LiveData` (or `Flow` exposed via Java-compatible wrappers). Unidirectional data flow. View renders immutable state; events flow one way.
- **Domain Layer:** Pure Java UseCases / Interactors and immutable entity classes (prefer `record`s on Java 17). No `android.*` or `androidx.*` imports.
- **Data Layer:** Repository pattern abstracting local (Room via Java annotations, `SharedPreferences` / `DataStore`) and remote (Retrofit + OkHttp) sources. DTOs map to domain entities.
- **Dependency Injection:** Hilt (preferred). Annotate `Application`, `Activity`, `Fragment`, and `ViewModel` with `@HiltAndroidApp`, `@AndroidEntryPoint`, `@HiltViewModel`. For modules that must not depend on Hilt, use Dagger 2 directly.

## 3. UI Framework
- **Views:** XML layouts with `ConstraintLayout` as the primary container. `findViewById` is legacy — use **ViewBinding** everywhere (`android.viewBinding true`).
- **DataBinding:** Optional. Enable only when two-way binding or binding expressions add clear value; otherwise ViewBinding is enough.
- **Navigation:** Jetpack Navigation Component with an XML nav graph and Safe Args (Java variants). Deep links via `<deepLink>` entries.
- **Theming:** Material Components for Android (`com.google.android.material`) with Material 3 `Theme.Material3.*`. Define colors, typography, and shape in `themes.xml`.
- **Images:** Glide 4 as the default image loader. Picasso is acceptable for maintenance apps; Coil is Kotlin-first and not recommended here.

## 4. Asynchronous Programming
- **Executors:** `java.util.concurrent.ExecutorService` with named thread pools from `Executors.newFixedThreadPool(...)` or a custom `ThreadPoolExecutor`. Always shut down executors in the appropriate lifecycle method.
- **Futures:** `CompletableFuture` on API 24+ (which is covered by `minSdk 21` only when guarded — prefer Guava `ListenableFuture` via `com.google.guava:guava` with `-android` variant for broad support).
- **Background Work:** `WorkManager` for deferrable, guaranteed execution (sync, upload, periodic tasks).
- **Reactive (optional):** RxJava 3 is a supported alternative for streams-of-events workflows. Do not mix RxJava 2 and RxJava 3.
- **Main Thread:** Never block. Use `Handler(Looper.getMainLooper())` only at boundaries; prefer posting results back via `MutableLiveData.postValue(...)`.

## 5. Networking & Persistence
- **Networking:** Retrofit 2 + OkHttp 4. Converters: Gson (default) or Moshi (preferred for new code). Interceptors for auth, logging (`HttpLoggingInterceptor` DEBUG-only), and retry policy.
- **Persistence:** Room 2.x with Java annotations (`@Entity`, `@Dao`, `@Database`). Expose DAOs that return `LiveData<T>`, `ListenableFuture<T>`, or `Flowable<T>` depending on the concurrency choice.
- **Preferences:** Jetpack `DataStore` (preferences) via its Java-compatible API; `SharedPreferences` only for legacy modules.
- **Serialization:** Gson or Moshi. Avoid Jackson on Android due to method count.

## 6. Testing Philosophy
- **Unit Tests (JVM):** JUnit 4 is the default runner for AndroidX Test; JUnit 5 (Jupiter) is acceptable for pure Java modules with the `android-junit5` plugin. Mockito 5 with `mockito-inline` for final classes. Parameterized tests via JUnit's `Parameterized` runner or JUnit 5 `@ParameterizedTest`.
- **Robolectric:** For JVM-side tests that need `Context`, resources, or `Looper`.
- **Instrumented Tests:** Espresso with Hilt test rules (`HiltAndroidRule`), `IdlingResource` for async work, `FragmentScenario` / `ActivityScenario` for isolation.
- **Architecture Tests:** Test ViewModels with `InstantTaskExecutorRule` for synchronous `LiveData`.

## 7. Code Style
- **Formatting:** Google Java Style via Spotless (`com.diffplug.spotless`). Enforce in CI.
- **Immutability:** Prefer `record`s for data carriers. Use `final` fields aggressively. Never expose mutable `List` / `Map` from a public API — return `List.copyOf(...)` or a read-only view.
- **Nullness:** Annotate with `androidx.annotation.NonNull` / `@Nullable`. Optional is acceptable as a return type in the domain layer; avoid it for fields and parameters.
- **Streams:** Use streams where they express intent clearly. Do not convert every loop to a stream. Prefer `Collectors.toUnmodifiableList()` on Java 10+.

## 8. External Documentation
- Android developer docs: [developer.android.com](https://developer.android.com)
- AndroidX reference: [developer.android.com/reference/androidx/packages](https://developer.android.com/reference/androidx/packages)
- Java 17 language reference: [docs.oracle.com/en/java/javase/17](https://docs.oracle.com/en/java/javase/17/)

## 9. Useful Agent Skills Recap

| Skill Folder                  | Purpose                                                           |
| ----------------------------- | ----------------------------------------------------------------- |
| `architecture/`               | Clean Architecture, MVVM, Room + Retrofit data layer in Java.     |
| `ui/`                         | XML views, ViewBinding, Navigation, Glide, a11y, animations, i18n.|
| `migration/`                  | `AsyncTask` → `ExecutorService`, `findViewById` → ViewBinding.    |
| `performance/`                | Profiler, `StrictMode`, R8, resource shrinking, Gradle cache.     |
| `concurrency_and_networking/` | `ExecutorService`, RxJava 3, Retrofit + OkHttp.                   |
| `testing_and_automation/`     | JUnit + Mockito, Espresso, Robolectric, Hilt test rules.          |
| `build_and_tooling/`          | Gradle Groovy DSL, flavors, signing, GitHub Actions CI.           |

---

Following these practices ensures that the agent-assisted development workflow stays reliable and consistent. When in doubt, always refer to the specific agent skills provided in `.github/skills/` for deeper task-specific context.

*Note to developers: Update this file whenever the project makes architectural shifts to ensure AI agents stay aligned with your conventions.*
