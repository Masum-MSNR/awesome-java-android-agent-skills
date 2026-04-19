---
name: java-mvvm
description: Expert guidance on implementing MVVM in Java using Jetpack ViewModel and LiveData with unidirectional data flow. Use this when building screens, wiring Fragments/Activities to state holders, or reviewing MVVM code in Java.
---

# MVVM in Java with ViewModel + LiveData

## Instructions

Use Jetpack `ViewModel` as the state holder and `LiveData` as the observable transport between `ViewModel` and `Fragment`/`Activity`. The view is dumb: it renders an immutable UI state and forwards events.

### 1. Immutable UI State

Model UI state as a single immutable `record` (Java 17). One state per screen — not one `LiveData` per field.

```java
public record ArticleListUiState(
        boolean loading,
        List<Article> articles,
        String errorMessage
) {
    public static ArticleListUiState initial() {
        return new ArticleListUiState(false, List.of(), null);
    }

    public ArticleListUiState loading() {
        return new ArticleListUiState(true, articles, null);
    }

    public ArticleListUiState success(List<Article> items) {
        return new ArticleListUiState(false, List.copyOf(items), null);
    }

    public ArticleListUiState failure(String message) {
        return new ArticleListUiState(false, articles, message);
    }
}
```

### 2. The ViewModel

```java
@HiltViewModel
public class ArticleListViewModel extends ViewModel {

    private final GetArticles getArticles;
    private final MutableLiveData<ArticleListUiState> state =
            new MutableLiveData<>(ArticleListUiState.initial());
    private final ExecutorService io = Executors.newSingleThreadExecutor();

    @Inject
    public ArticleListViewModel(GetArticles getArticles) {
        this.getArticles = getArticles;
        refresh();
    }

    public LiveData<ArticleListUiState> state() {
        return state;
    }

    public void refresh() {
        state.setValue(requireState().loading());
        io.execute(() -> {
            try {
                List<Article> items = getArticles.execute();
                state.postValue(requireState().success(items));
            } catch (Exception e) {
                state.postValue(requireState().failure(e.getMessage()));
            }
        });
    }

    private ArticleListUiState requireState() {
        return Objects.requireNonNull(state.getValue());
    }

    @Override
    protected void onCleared() {
        io.shutdownNow();
    }
}
```

Key points:
- `setValue` on the main thread, `postValue` from background threads.
- Always shut down owned executors in `onCleared()`.
- Expose `LiveData<T>`, not `MutableLiveData<T>`.

### 3. The Fragment

```java
@AndroidEntryPoint
public class ArticleListFragment extends Fragment {

    private FragmentArticleListBinding binding;
    private ArticleListViewModel viewModel;
    private ArticleAdapter adapter;

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup parent, Bundle s) {
        binding = FragmentArticleListBinding.inflate(inflater, parent, false);
        return binding.getRoot();
    }

    @Override
    public void onViewCreated(View view, Bundle s) {
        viewModel = new ViewModelProvider(this).get(ArticleListViewModel.class);
        adapter = new ArticleAdapter();
        binding.list.setAdapter(adapter);
        binding.swipeRefresh.setOnRefreshListener(viewModel::refresh);

        viewModel.state().observe(getViewLifecycleOwner(), this::render);
    }

    private void render(ArticleListUiState s) {
        binding.swipeRefresh.setRefreshing(s.loading());
        adapter.submitList(s.articles());
        binding.error.setVisibility(s.errorMessage() == null ? View.GONE : View.VISIBLE);
        binding.error.setText(s.errorMessage());
    }

    @Override
    public void onDestroyView() {
        super.onDestroyView();
        binding = null;
    }
}
```

### 4. Single-Shot Events

Use a `SingleLiveEvent` or `Channel`-like queue for navigation and snackbars so they are not re-delivered after rotation:

```java
public final class Event<T> {
    private final T content;
    private boolean handled = false;

    public Event(T content) { this.content = content; }

    public T getIfNotHandled() {
        if (handled) return null;
        handled = true;
        return content;
    }
}
```

Expose as `LiveData<Event<NavTarget>>` and call `getIfNotHandled()` in the observer.

### 5. Two-Way Input with DataBinding (optional)

When form editing is heavy, enable DataBinding and bind `MutableLiveData<String>` directly to `EditText` via `android:text="@={viewModel.query}"`. Otherwise stay on ViewBinding + `TextWatcher` and call `viewModel.onQueryChanged(...)`.

## Checklist

- [ ] Exactly one `LiveData<UiState>` per screen. No bag-of-`LiveData`.
- [ ] State class is immutable (`record` or final fields with defensive copies).
- [ ] `Fragment` observes with `getViewLifecycleOwner()`, never `this`.
- [ ] `binding` field is nulled in `onDestroyView()`.
- [ ] Background work runs on an `ExecutorService` or WorkManager — never on the main thread.
- [ ] Single-shot events use an `Event` wrapper or `SingleLiveEvent`.
