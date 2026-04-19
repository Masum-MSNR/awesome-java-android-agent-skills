---
name: java-performance-audit
description: Expert guidance on auditing a Java Android app's runtime performance with Android Studio Profiler, StrictMode, Layout Inspector, and GPU overdraw. Use this when investigating jank, slow startup, ANRs, or memory leaks.
---

# Java Android Runtime Performance Audit

## Instructions

Performance work is measurement-first. Do not optimize without a trace.

### 1. Establish a Baseline

Measure these before changing any code:
- **Startup time** — `adb shell am start -W ...` → `TotalTime`, `WaitTime`.
- **Frame rate** — Profiler → CPU → System Trace, or Macrobenchmark library.
- **Memory** — Profiler → Memory → record allocations.
- **Network** — Profiler → Network timeline.

Save traces in `docs/perf/` with a date; every future change compares against the baseline.

### 2. StrictMode — catch main-thread IO immediately

Enable StrictMode in debug builds only:

```java
public class App extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        if (BuildConfig.DEBUG) {
            StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
                    .detectDiskReads()
                    .detectDiskWrites()
                    .detectNetwork()
                    .penaltyLog()
                    .penaltyFlashScreen()
                    .build());
            StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()
                    .detectLeakedClosableObjects()
                    .detectLeakedSqlLiteObjects()
                    .detectActivityLeaks()
                    .penaltyLog()
                    .build());
        }
    }
}
```

Treat every StrictMode log line as a bug to file.

### 3. Layout Inspector & Hierarchy

- Aim for **depth < 8** from the Activity's root. Anything deeper pays for a measure pass per level.
- Flatten nested `LinearLayout`s into `ConstraintLayout` (see `android-views-xml` skill).
- `RelativeLayout` does two measure passes — prefer `ConstraintLayout`.
- Remove `android:background` set to the same color as the parent; it causes overdraw.

Enable **Debug GPU Overdraw** (developer options) — any red regions on a scrollable list are the main cost to fix.

### 4. RecyclerView

- Use `DiffUtil` / `ListAdapter`; never call `notifyDataSetChanged()` for content changes.
- Set `setHasStableIds(true)` + `getItemId(position)` when IDs are stable.
- Use view pools across RecyclerViews showing the same cell type.
- Set fixed sizes on `ImageView`s so Glide can downsample (see `glide-images` skill).
- Binding should do **zero** allocations on the hot path — move formatters to fields, pre-compute `contentDescription`.

### 5. Startup

- Move work out of `Application.onCreate` — use Jetpack **App Startup** (`androidx.startup`) to lazy-initialize libraries.
- Use `Baseline Profiles` (`androidx.benchmark:benchmark-macro-junit4`) to generate a profile for R8 to pre-JIT critical paths.
- Defer WorkManager enqueues behind `DelayableInitializer`.

```java
public class CrashReporterInitializer implements Initializer<Unit> {
    @NonNull
    @Override public Unit create(@NonNull Context context) {
        CrashReporter.init(context);
        return Unit.INSTANCE;
    }
    @NonNull
    @Override public List<Class<? extends Initializer<?>>> dependencies() {
        return List.of();
    }
}
```

### 6. Memory

- Profiler → Memory → "Dump Java heap" → filter by `Activity`. If an Activity instance persists after `finish()`, you have a leak.
- Use **LeakCanary** (`com.squareup.leakcanary:leakcanary-android`) in `debugImplementation`.
- Common leaks:
  - Inner classes holding implicit outer reference (Fragment listeners, Handlers).
  - Singletons that cache `Activity` or `View`.
  - Forgotten `BroadcastReceiver` / `Observer` registrations.

### 7. ANRs

An ANR is any unresponsive main-thread window > 5s. Diagnose with:
- `adb bugreport` → `traces.txt`.
- Firebase Crashlytics ANR collection.
- Google Play vitals.

Root causes map 1:1 to what StrictMode catches: disk IO, network, large layouts, broadcast processing on main thread.

### 8. Background Work

- Use `JobScheduler`/`WorkManager` instead of `BOOT_COMPLETED` receivers.
- Coalesce network calls; prefer `OkHttp` connection pool reuse (one `OkHttpClient` per app).
- Batch Room writes in a transaction: `db.runInTransaction(() -> { ... })`.

### 9. Profiling Cadence

- Run Systrace on release builds (AOT-compiled) — debug builds with R8 off are not representative.
- Test on a mid-tier device (Pixel 6a, Samsung A-series), not a flagship.
- Record a 30-second trace of the primary user journey each release.

## Checklist

- [ ] StrictMode is enabled in debug and produces zero violations during the smoke journey.
- [ ] `Profile GPU Rendering` shows no frames above the 16ms line on the main list screen.
- [ ] View hierarchy depth is under 8 for every primary screen.
- [ ] `notifyDataSetChanged()` does not appear in production code.
- [ ] LeakCanary is integrated in debug and the primary journey produces no leaks.
- [ ] Cold startup `TotalTime` is tracked release-over-release and trending flat or down.
- [ ] App Startup library is used; heavy libraries initialize lazily.
