---
name: java-executors
description: Expert guidance on using java.util.concurrent (ExecutorService, ThreadPoolExecutor, Futures, ListenableFuture) on Android from Java. Use this when designing background work, custom thread pools, or cancellation-aware pipelines.
---

# Java Executors for Android

## Instructions

`java.util.concurrent` is the foundation of every non-UI thread on Android. Build on it directly — avoid `Thread.start()`, `AsyncTask`, and `Handler(new HandlerThread(...))` unless you are integrating with legacy APIs.

### 1. Pool Types

| Call                                    | Use when                                                         |
| --------------------------------------- | ---------------------------------------------------------------- |
| `Executors.newSingleThreadExecutor()`   | Sequential work, e.g. per-feature repository.                    |
| `Executors.newFixedThreadPool(n)`       | Parallel CPU-bound work. `n = Runtime.availableProcessors()`.    |
| `Executors.newCachedThreadPool()`       | Short bursts of IO-bound work. Dangerous unbounded; avoid.       |
| Custom `ThreadPoolExecutor`             | Anything shipped to production. See below.                       |
| `Executors.newScheduledThreadPool(1)`   | Periodic or delayed tasks (prefer WorkManager for > 15 min).     |

### 2. Production-Grade Pool

```java
public final class AppExecutors {

    public static ExecutorService io() {
        ThreadFactory factory = runnable -> {
            Thread t = new Thread(runnable, "io-" + COUNTER.incrementAndGet());
            t.setPriority(Thread.NORM_PRIORITY - 1);
            t.setUncaughtExceptionHandler((thr, err) ->
                    Log.e("AppExecutors", "Uncaught in " + thr.getName(), err));
            return t;
        };
        return new ThreadPoolExecutor(
                4, 8,
                60L, TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(128),
                factory,
                new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }

    private static final AtomicInteger COUNTER = new AtomicInteger();
    private AppExecutors() {}
}
```

Always name your threads — unnamed threads in profiler traces are debugging nightmares.

### 3. Submit + Future

```java
ExecutorService io = AppExecutors.io();

Future<List<Article>> future = io.submit(() -> repo.loadAll());

try {
    List<Article> result = future.get(5, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    future.cancel(true);
    throw new RuntimeException("Load timed out", e);
}
```

`get()` on the main thread is an ANR waiting to happen. Post results via `LiveData.postValue` instead.

### 4. ListenableFuture (Guava)

`ListenableFuture` composes better than `Future`. Room `-guava` DAOs and WorkManager both speak it.

```java
ListenableFuture<Article> future = Futures.submit(
        () -> repo.loadById(id), io);

Futures.addCallback(future, new FutureCallback<>() {
    @Override public void onSuccess(Article result) { state.postValue(result); }
    @Override public void onFailure(@NonNull Throwable t) { state.postValue(null); }
}, ContextCompat.getMainExecutor(ctx));
```

Chain transformations without nesting:

```java
ListenableFuture<Article> fetch = api.getArticle(id);
ListenableFuture<Void> save = Futures.transformAsync(
        fetch, a -> dao.upsert(toEntity(a)), io);
```

### 5. Cancellation

- `future.cancel(true)` — interrupts the worker thread.
- In the task body, check `Thread.currentThread().isInterrupted()` on long loops.
- `shutdownNow()` returns unstarted tasks and interrupts running ones.

```java
@Override
protected void onCleared() {
    io.shutdownNow();
}
```

### 6. Structured Ownership

One executor should have **one owner** with a clear lifecycle:

| Scope           | Owner          | Shutdown                          |
| --------------- | -------------- | --------------------------------- |
| ViewModel       | `ViewModel`    | `onCleared()`                     |
| Feature module  | Hilt `@Module` singleton | `@ApplicationContext` app exit |
| Background sync | WorkManager    | Managed by WorkManager            |

Do not share a `ViewModel`-scoped executor across features.

### 7. `CompletableFuture` (API 24+)

On `minSdk 24+`, `CompletableFuture` is available and composes like `ListenableFuture`:

```java
CompletableFuture.supplyAsync(repo::loadAll, io)
    .thenApply(items -> items.stream()
        .filter(Article::isPublished)
        .collect(Collectors.toUnmodifiableList()))
    .whenComplete((result, err) -> {
        if (err != null) state.postValue(UiState.error(err.getMessage()));
        else             state.postValue(UiState.success(result));
    });
```

For `minSdk 21–23`, stick with Guava `ListenableFuture`.

### 8. Common Pitfalls

- Creating an executor per task — **leaks threads forever**. Reuse.
- `Executors.newCachedThreadPool()` under load → thousands of threads → OOM.
- Swallowing `InterruptedException`. Always restore the flag: `Thread.currentThread().interrupt()`.
- Scheduling UI callbacks via `post` on views already destroyed — gate with `isAttachedToWindow()`.

## Checklist

- [ ] No `new Thread(...)` in production code.
- [ ] Every `ExecutorService` has a named `ThreadFactory`.
- [ ] Every `ExecutorService` is shut down in its owning lifecycle callback.
- [ ] `Future.get()` is never called on the main thread.
- [ ] `ListenableFuture` or `CompletableFuture` is preferred for composition.
- [ ] `InterruptedException` is re-raised or the interrupt flag is restored.
- [ ] Thread names are visible and meaningful in the Android Studio Profiler.
