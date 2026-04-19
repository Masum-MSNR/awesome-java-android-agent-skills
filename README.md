# Awesome Java Android Agent Skills

[![Agent Skills](https://img.shields.io/badge/Agent-Skills-blue?style=flat&logo=github)](https://agentskills.io)
[![Java 17](https://img.shields.io/badge/Java-17-orange?style=flat&logo=openjdk)](https://openjdk.org/projects/jdk/17/)
[![Android minSdk 21](https://img.shields.io/badge/Android-minSdk%2021-3DDC84?style=flat&logo=android)](https://developer.android.com)
[![License: MIT](https://img.shields.io/badge/Code-MIT-green?style=flat)](LICENSE-CODE)
[![License: CC BY 4.0](https://img.shields.io/badge/Docs-CC%20BY%204.0-lightgrey?style=flat)](LICENSE-DOCS)

A collection of **20 agent skills** that give AI coding assistants (GitHub Copilot, Claude, Gemini, Cursor, etc.) expert-level knowledge of modern **Java-based Android** development.

This repository targets teams that must stay on Java (not Kotlin) — typical reasons include legacy codebases, corporate toolchain constraints, SDK/library authors that ship Java artifacts, or heterogeneous teams where Java remains the common denominator.

Drop the `.github/skills/` folder into your project and your agent automatically follows your architecture, uses current best practices, and ships code that passes review.

> Learn more about the Agent Skills standard at [agentskills.io](https://agentskills.io).

---

## Why use this

- **Java-first, modern Java** -- Java 17 language level, `var`, records, switch expressions, `Optional`, streams, `CompletableFuture`.
- **Current Android defaults** -- minSdk 21, targetSdk 35, AGP 8.x, Gradle with Groovy or Kotlin DSL, AndroidX, Jetpack libraries.
- **Production architecture** -- Clean Architecture + MVVM with `ViewModel`, `LiveData`, Hilt DI, Room, Retrofit, WorkManager.
- **Opinionated but swappable** -- each skill explains the preferred tool and a documented alternative (RxJava 3 vs `ExecutorService`, Gson vs Moshi, Espresso vs Robolectric).
- **Actionable checklists** -- every skill ends with a verification checklist the agent can self-audit against.

---

## How it works

The `Agent.md` file is the foundation. Every AI agent reads it first to understand your project conventions. Individual skills layer on top for specific tasks.

```text
       ┌───────────────────────────────────────────┐
       │                  Agent.md                 │
       │     (read by all AI agents implicitly)    │
       └─────────────────────┬─────────────────────┘
                             │
     ┌───────────────┬───────┴───────┬───────────────┐
     ▼               ▼               ▼               ▼
┌──────────┐   ┌───────────┐   ┌────────────┐  ┌────────────┐
│ Architect│   │  UI & UX  │   │ Migration  │  │  Delivery  │
├──────────┤   ├───────────┤   ├────────────┤  ├────────────┤
│clean-arch│   │views-xml  │   │asynctask→  │  │junit+mock  │
│mvvm      │   │viewbinding│   │  executor  │  │espresso    │
│data-layer│   │navigation │   │findViewById│  │gradle      │
│          │   │glide      │   │ →binding   │  │ci-cd       │
│          │   │a11y       │   │            │  │            │
│          │   │animations │   │            │  │            │
│          │   │i18n       │   │            │  │            │
└──────────┘   └───────────┘   └────────────┘  └────────────┘
```

---

## Available Skills

All skills live in `.github/skills/<category>/<skill-name>/SKILL.md`.

### Architecture

| Skill | What it covers |
|-------|----------------|
| [Java Android Architecture](.github/skills/architecture/java-android-architecture/SKILL.md) | Clean Architecture in Java, package-by-feature vs layer, modularization |
| [Java MVVM](.github/skills/architecture/java-mvvm/SKILL.md) | `ViewModel` + `LiveData`, data binding, unidirectional flow without Kotlin |
| [Java Data Layer](.github/skills/architecture/java-data-layer/SKILL.md) | Repository pattern with Room (Java), Retrofit, offline caching |

### UI

| Skill | What it covers |
|-------|----------------|
| [Android Views XML](.github/skills/ui/android-views-xml/SKILL.md) | XML layouts, `ConstraintLayout` best practices, style/theme system |
| [ViewBinding Patterns](.github/skills/ui/viewbinding-patterns/SKILL.md) | ViewBinding over `findViewById`, binding in Fragment/Activity/Adapter |
| [Navigation (Jetpack)](.github/skills/ui/android-navigation-jetpack/SKILL.md) | Navigation Component with XML graph, Safe Args Java, deep links |
| [Glide Images](.github/skills/ui/glide-images/SKILL.md) | Glide 4 pipelines, placeholders, caching, thumbnail, transformations |
| [Java Accessibility](.github/skills/ui/java-accessibility/SKILL.md) | `contentDescription`, TalkBack, touch targets, dynamic text |
| [Android Animations (Java)](.github/skills/ui/android-animations-java/SKILL.md) | `ObjectAnimator`, `ValueAnimator`, MotionLayout, Transitions |
| [Java Localization](.github/skills/ui/java-localization/SKILL.md) | `strings.xml` variants, plurals, ICU, RTL, per-app locale |

### Migration

| Skill | What it covers |
|-------|----------------|
| [AsyncTask to Executor](.github/skills/migration/asynctask-to-executor/SKILL.md) | Replacing deprecated `AsyncTask` with `ExecutorService` / WorkManager |
| [findViewById to ViewBinding](.github/skills/migration/findviewbyid-to-viewbinding/SKILL.md) | Mechanical and safety-oriented migration path |

### Performance

| Skill | What it covers |
|-------|----------------|
| [Java Performance Audit](.github/skills/performance/java-performance-audit/SKILL.md) | Android Studio Profiler, `StrictMode`, Layout Inspector, overdraw |
| [Java Build Optimization](.github/skills/performance/java-build-optimization/SKILL.md) | Gradle configuration cache, R8 optimization, resource shrinking |

### Concurrency & Networking

| Skill | What it covers |
|-------|----------------|
| [Java Executors](.github/skills/concurrency_and_networking/java-executors/SKILL.md) | `ExecutorService` patterns, thread pools, cancellation, `ListenableFuture` |
| [RxJava 3](.github/skills/concurrency_and_networking/rxjava3/SKILL.md) | `Observable` / `Single` / `Flowable`, schedulers, disposables, error handling |
| [Retrofit (Java)](.github/skills/concurrency_and_networking/retrofit-java/SKILL.md) | Retrofit + OkHttp in Java, interceptors, call adapters, error bodies |

### Testing & Automation

| Skill | What it covers |
|-------|----------------|
| [JUnit + Mockito](.github/skills/testing_and_automation/junit-mockito/SKILL.md) | Unit testing ViewModel/repository, Mockito, parameterized tests |
| [Espresso (Java)](.github/skills/testing_and_automation/espresso-java/SKILL.md) | Espresso `IdlingResource`, Hilt test rules, Robolectric for JVM tests |

### Build & Tooling

| Skill | What it covers |
|-------|----------------|
| [Gradle (Groovy)](.github/skills/build_and_tooling/gradle-groovy/SKILL.md) | `build.gradle` Groovy DSL patterns, productFlavors, signingConfigs |
| [Java Android CI](.github/skills/build_and_tooling/java-android-ci/SKILL.md) | GitHub Actions with JDK 17, Gradle cache, signing, Play Publisher |

---

## Quick Start

1. **Copy** the `.github/skills/` folder and `Agent.md` to your project root.
2. **Verify** the structure:
   ```text
   my-android-app/
   ├── Agent.md
   ├── .github/
   │   └── skills/
   │       ├── architecture/
   │       │   ├── java-android-architecture/
   │       │   │   └── SKILL.md
   │       │   └── ...
   │       └── ...
   ├── app/
   │   ├── build.gradle
   │   └── src/main/java/...
   └── ...
   ```
3. **Reload** your editor (Copilot and similar extensions may need a window reload to index new skills).

Then just ask your agent naturally:

> "How should I structure the User Profile feature in Java?"
> "Create a Retrofit repository for fetching News with Room offline caching."

The agent picks the right skill automatically.

---

## Other Environments

- **Claude**: copy or symlink `.github/skills` to `.claude/skills/`.
- **OpenCode**: supports `.opencode/skill/` and `.claude/skills/`. Global skills go in `~/.config/opencode/skill/`.

---

## Adding Custom Skills

Create a folder in `.github/skills/` with a `SKILL.md` containing the required frontmatter:

```markdown
---
name: my-custom-skill
description: What this skill does
---
# Instructions
...
```

---

## License

Dual-licensed:

- **Code** (snippets, configs, scripts) -- [MIT](LICENSE-CODE)
- **Documentation** (SKILL.md, README, Agent.md) -- [CC BY 4.0](LICENSE-DOCS)

See [LICENSE](LICENSE) for details.
