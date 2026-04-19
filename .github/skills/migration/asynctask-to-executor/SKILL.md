---
name: asynctask-to-executor
description: Step-by-step migration from deprecated AsyncTask to modern java.util.concurrent and WorkManager. Use this when modernizing legacy Java Android code that still uses AsyncTask.
---

# Migrating AsyncTask to ExecutorService / WorkManager

## Instructions

`AsyncTask` has been **deprecated since API 30**. It leaks Activities, serializes tasks globally on older devices, and has no structured cancellation. Replace it with either a scoped `ExecutorService` + `Handler` or `WorkManager` depending on the use case.

### 1. Decision Tree

- Short-lived, tied to a screen (load, parse, save on click) → **`ExecutorService` + `ViewModel`**.
- Deferrable, must run even if the app dies (upload, sync) → **`WorkManager`**.
- Reactive pipeline with cancellation-on-recycle → **RxJava 3** (see `rxjava3` skill).

### 2. Pattern: In-Activity AsyncTask

Before:

```java
private class LoadTask extends AsyncTask<Void, Void, List<Article>> {
    @Override
    protected List<Article> doInBackground(Void... voids) {
        return repo.loadAll();
    }
    @Override
    protected void onPostExecute(List<Article> items) {
        if (isDestroyed()) return;
        adapter.submitList(items);
    }
}
```

After (ExecutorService + ViewModel + LiveData):

```java
@HiltViewModel
public class ArticleListViewModel extends ViewModel {

    private final ArticleRepository repo;
    private final ExecutorService io = Executors.newSingleThreadExecutor();
    private final MutableLiveData<List<Article>> items = new MutableLiveData<>();

    @Inject
    ArticleListViewModel(ArticleRepository repo) { this.repo = repo; }

    public LiveData<List<Article>> items() { return items; }

    public void load() {
        io.execute(() -> items.postValue(repo.loadAll()));
    }

    @Override
    protected void onCleared() {
        io.shutdownNow();
    }
}
```

The Fragment observes `items()` via `getViewLifecycleOwner()`. No leak, no `isDestroyed()` guard, cancellation is automatic when the `ViewModel` is cleared.

### 3. Pattern: AsyncTask with Progress

Before:

```java
class DownloadTask extends AsyncTask<String, Integer, File> {
    protected File doInBackground(String... urls) {
        publishProgress(0);
        // ...
        publishProgress(100);
        return file;
    }
    protected void onProgressUpdate(Integer... p) { progressBar.setProgress(p[0]); }
}
```

After:

```java
public record DownloadState(int progress, File file, String error) { }

public void download(String url) {
    MutableLiveData<DownloadState> state = new MutableLiveData<>();
    io.execute(() -> {
        try {
            for (int p : steps) {
                state.postValue(new DownloadState(p, null, null));
            }
            File out = doDownload(url);
            state.postValue(new DownloadState(100, out, null));
        } catch (IOException e) {
            state.postValue(new DownloadState(0, null, e.getMessage()));
        }
    });
}
```

### 4. Pattern: Background Sync (move to WorkManager)

Before:

```java
new SyncTask().execute();   // started from BOOT_COMPLETED receiver
```

After:

```java
public class SyncWorker extends Worker {

    private final ArticleRepository repo;

    @AssistedInject
    public SyncWorker(@Assisted Context ctx, @Assisted WorkerParameters params,
                      ArticleRepository repo) {
        super(ctx, params);
        this.repo = repo;
    }

    @NonNull
    @Override
    public Result doWork() {
        try {
            repo.syncBlocking();
            return Result.success();
        } catch (IOException e) {
            return getRunAttemptCount() < 3 ? Result.retry() : Result.failure();
        }
    }
}

// Enqueue
Constraints constraints = new Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .build();
PeriodicWorkRequest req = new PeriodicWorkRequest.Builder(
        SyncWorker.class, 6, TimeUnit.HOURS)
        .setConstraints(constraints)
        .build();
WorkManager.getInstance(context).enqueueUniquePeriodicWork(
        "article-sync", ExistingPeriodicWorkPolicy.KEEP, req);
```

WorkManager survives process death, retries on backoff, and respects constraints.

### 5. Cancellation Map

| AsyncTask call                | Replacement                                            |
| ----------------------------- | ------------------------------------------------------ |
| `task.cancel(true)`           | `future.cancel(true)` or `executor.shutdownNow()`      |
| `isCancelled()` in `doInBg`   | Check `Thread.currentThread().isInterrupted()`         |
| `onCancelled()`               | `catch (InterruptedException)` or future cancellation callback |
| `SERIAL_EXECUTOR`             | `Executors.newSingleThreadExecutor()`                  |
| `THREAD_POOL_EXECUTOR`        | `Executors.newFixedThreadPool(n)` with named threads   |

### 6. Step-by-Step Migration

1. List all `AsyncTask` subclasses (`grep -r 'extends AsyncTask'`).
2. For each, classify as: UI-scoped, deferrable, or reactive.
3. Replace with the corresponding pattern above.
4. Delete the `AsyncTask` subclass.
5. Turn on the lint check `AsyncTaskUsage` (`android.lint.warningsAsErrors = true`) to prevent regressions.
6. Audit for `Handler(Looper.getMainLooper())` + background thread combinations — many are also hand-rolled AsyncTasks. Consolidate on `LiveData.postValue` or `runOnUiThread` at the boundary only.

## Checklist

- [ ] `grep -rn 'AsyncTask' src/main/java` returns **zero** results.
- [ ] Every background executor is owned by a lifecycle (`ViewModel.onCleared`, `WorkManager`, or `shutdown` on app exit).
- [ ] No `new Handler()` without an explicit `Looper`.
- [ ] `WorkManager` is used for anything that must survive the process.
- [ ] Lint rule `AsyncTaskUsage` is set to `error`.
