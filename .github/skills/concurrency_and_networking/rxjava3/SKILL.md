---
name: rxjava3
description: Expert guidance on RxJava 3 for Android — Observable, Single, Flowable, schedulers, disposables, and error handling. Use this when building reactive pipelines or maintaining existing RxJava code in Java.
---

# RxJava 3 on Android

## Instructions

RxJava 3 (`io.reactivex.rxjava3`) is a supported reactive option for Java-based Android projects. Do not mix RxJava 2 (`io.reactivex.*`) and 3 in the same module.

### 1. Setup

```groovy
dependencies {
    implementation 'io.reactivex.rxjava3:rxjava:3.1.8'
    implementation 'io.reactivex.rxjava3:rxandroid:3.0.2'
    implementation 'com.squareup.retrofit2:adapter-rxjava3:2.11.0'  // Retrofit call adapter
}
```

### 2. Stream Types — pick the right one

| Type              | Emits                                   | Backpressure | Typical use                        |
| ----------------- | --------------------------------------- | ------------ | ---------------------------------- |
| `Single<T>`       | 1 value or error                        | no           | One-shot network call, DB read.    |
| `Maybe<T>`        | 0 or 1 value or error                   | no           | "Get by id" that may be empty.     |
| `Completable`     | Completion or error                     | no           | Insert/update/delete.              |
| `Observable<T>`   | Many values                             | no           | UI events (small, fast).           |
| `Flowable<T>`     | Many values                             | yes          | Database change streams, sensors.  |

Choose the **least powerful** type that fits. Don't return `Observable<User>` for a single network call.

### 3. Schedulers

| Scheduler                    | For                                        |
| ---------------------------- | ------------------------------------------ |
| `Schedulers.io()`            | Network / disk IO.                         |
| `Schedulers.computation()`   | CPU-bound work (parsing, transformations). |
| `AndroidSchedulers.mainThread()` | UI updates.                            |
| `Schedulers.single()`        | Serialized work.                           |
| `Schedulers.from(executor)`  | Bridging to your own pool.                 |

Standard suffix on every pipeline:

```java
repo.loadArticles()
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(items -> render(items), this::showError);
```

### 4. Disposables and Lifecycle

Every subscription returns a `Disposable`. Own its lifetime.

```java
public class ArticleListFragment extends Fragment {

    private final CompositeDisposable disposables = new CompositeDisposable();

    private void load() {
        disposables.add(
            repo.loadArticles()
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(this::render, this::showError));
    }

    @Override
    public void onDestroyView() {
        disposables.clear();   // clear, not dispose — reusable
        super.onDestroyView();
    }
}
```

For ViewModel:

```java
@Override
protected void onCleared() {
    disposables.dispose();
}
```

### 5. Error Handling

Never swallow errors. Every `subscribe` **must** pass an error consumer.

```java
.subscribe(
    value -> ...,
    err -> {
        Log.e(TAG, "load failed", err);
        state.postValue(UiState.error(humanMessage(err)));
    }
);
```

Set a global safety net for truly unexpected errors:

```java
RxJavaPlugins.setErrorHandler(t -> {
    if (t instanceof UndeliverableException) return;  // common: subscription already disposed
    Log.e("RxGlobal", "Uncaught", t);
});
```

### 6. Combining Streams

```java
Single<User>      user    = api.user();
Single<Settings>  settings = db.settings();

Single.zip(user, settings, Profile::new)
      .subscribeOn(Schedulers.io())
      .observeOn(AndroidSchedulers.mainThread())
      .subscribe(this::render, this::showError);
```

`zip` waits for both; `combineLatest` emits whenever either source emits; `merge` interleaves.

### 7. Retrofit Integration

```java
public interface ArticleApi {
    @GET("articles")
    Single<List<ArticleDto>> listArticles();
}

Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(MoshiConverterFactory.create())
    .addCallAdapterFactory(RxJava3CallAdapterFactory.create())
    .build();
```

Compose retries:

```java
api.listArticles()
    .retryWhen(errors -> errors
        .zipWith(Flowable.range(1, 3), (e, i) -> i)
        .flatMap(i -> Flowable.timer((long) Math.pow(2, i), TimeUnit.SECONDS)))
```

### 8. Testing

```java
@Before
public void setUp() {
    RxJavaPlugins.setIoSchedulerHandler(s -> Schedulers.trampoline());
    RxAndroidPlugins.setMainThreadSchedulerHandler(s -> Schedulers.trampoline());
}

@Test
public void loads() {
    TestObserver<List<Article>> obs = repo.loadArticles().test();
    obs.assertValue(items -> items.size() == 2);
}
```

## Checklist

- [ ] Exactly one version of RxJava (3.x) is on the classpath.
- [ ] Every pipeline has `subscribeOn(Schedulers.io())` and `observeOn(AndroidSchedulers.mainThread())` where relevant.
- [ ] Every `subscribe(...)` provides an error handler.
- [ ] Disposables are owned by `CompositeDisposable` tied to a lifecycle.
- [ ] The least powerful stream type is used (`Single` over `Observable` for one-shot).
- [ ] `RxJavaPlugins.setErrorHandler` is installed.
- [ ] No `blockingGet()` on the main thread.
