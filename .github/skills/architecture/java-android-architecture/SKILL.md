---
name: java-android-architecture
description: Expert guidance on structuring a Java-based Android application using Clean Architecture, feature modularization, and dependency injection with Hilt. Use this when asked about project structure, module layout, or DI in a Java codebase.
---

# Java Android Clean Architecture & Modularization

## Instructions

When designing or refactoring a Java-based Android app, follow **Clean Architecture** principles. Dependencies must flow **inwards** toward pure Java domain code. The repository standard is Java 17 with AndroidX + Hilt.

### 1. High-Level Layers

- **Presentation Layer**
  - **Responsibility:** Rendering views and handling user input.
  - **Components:** `Activity`, `Fragment`, `ViewModel` (Jetpack), `LiveData`, adapters.
  - **Dependencies:** Domain interfaces only (UseCases or repository interfaces).
- **Domain Layer (Pure Java)**
  - **Responsibility:** Business rules and domain entities.
  - **Components:** UseCases (e.g. `GetArticles`), immutable entities (prefer `record`s), repository **interfaces**.
  - **Rule:** MUST NOT import `android.*` or `androidx.*`. Keep under `:domain` module or `com.example.app.domain`.
- **Data Layer**
  - **Responsibility:** Fetching, caching, and persisting data.
  - **Components:** Repository **implementations**, Retrofit services, Room DAOs, DTOs + mappers.

### 2. Package-by-Feature vs Package-by-Layer

For anything larger than a sample, prefer **package-by-feature** at the top level, with internal layering:

```
com.example.app/
├── core/                 // shared: theme, errors, network, DI
├── di/                   // Hilt @Module classes
├── features/
│   ├── articles/
│   │   ├── data/         // ArticleRepositoryImpl, ArticleApi, ArticleDao
│   │   ├── domain/       // Article (record), ArticleRepository, GetArticles
│   │   └── ui/           // ArticleListFragment, ArticleListViewModel
│   └── profile/
│       └── ...
└── App.java              // @HiltAndroidApp
```

Package-by-layer (`ui/`, `domain/`, `data/` at the root) is acceptable for small apps but does not scale past ~3 features.

### 3. Dependency Injection with Hilt

```java
@HiltAndroidApp
public class App extends Application { }

@Module
@InstallIn(SingletonComponent.class)
public abstract class RepositoryModule {
    @Binds
    abstract ArticleRepository bindArticleRepository(ArticleRepositoryImpl impl);
}

@HiltViewModel
public class ArticleListViewModel extends ViewModel {
    private final GetArticles getArticles;

    @Inject
    public ArticleListViewModel(GetArticles getArticles) {
        this.getArticles = getArticles;
    }
}
```

Never use service locators in production code. `@Inject` on the constructor is the default; `@Provides` modules are only for types you do not own.

### 4. Modularization Strategy

For apps with more than ~5 features, split into Gradle modules:

```
settings.gradle
├── :app              // wiring, Application class, navigation host
├── :core:ui          // common widgets, theme resources
├── :core:network     // OkHttp, Retrofit, interceptors
├── :core:database    // Room database + base DAOs
├── :feature:articles
│   ├── :feature:articles:api       // public interfaces
│   └── :feature:articles:impl      // implementation, UI
└── :domain           // pure-Java module (no android plugin)
```

`:domain` uses `apply plugin: 'java-library'` so it cannot accidentally depend on Android types. Feature `impl` modules depend on feature `api` modules, never on each other.

### 5. Enforcing Architecture

Use `com.tngtech.archunit:archunit-junit5` to fail the build on violations:

```java
@AnalyzeClasses(packages = "com.example.app")
class ArchitectureTest {

    @ArchTest
    static final ArchRule domainHasNoAndroidDeps =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("android..", "androidx..");

    @ArchTest
    static final ArchRule featuresDoNotDependOnEachOther =
        noClasses().that().resideInAPackage("..features.articles..")
            .should().dependOnClassesThat()
            .resideInAPackage("..features.profile..");
}
```

## Checklist

- [ ] Domain layer has zero `android.*` / `androidx.*` imports (enforced by ArchUnit or a pure-Java Gradle module).
- [ ] Repositories expose domain entities; never leak Retrofit `Response` or Room entities to the UI.
- [ ] `ViewModel`s receive dependencies via `@Inject` constructor, never via static singletons.
- [ ] Features do not import sibling features; communication goes through `:core` or navigation.
- [ ] `:domain` module is a `java-library` module, not an `android-library`.
