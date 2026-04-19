---
name: viewbinding-patterns
description: Expert guidance on using Jetpack ViewBinding idiomatically from Java in Activities, Fragments, and RecyclerView adapters. Use this when wiring XML to Java code, replacing findViewById, or reviewing ViewBinding usage.
---

# ViewBinding Patterns in Java

## Instructions

ViewBinding replaces `findViewById` with a generated, null-safe, type-safe binding class per layout. `findViewById` is legacy and should only remain in code explicitly marked for deletion.

### 1. Enable ViewBinding

In `app/build.gradle`:

```groovy
android {
    buildFeatures {
        viewBinding true
    }
}
```

For a file named `fragment_article_list.xml`, Android Studio generates `FragmentArticleListBinding` in your module's package.

### 2. Activity

```java
public class MainActivity extends AppCompatActivity {

    private ActivityMainBinding binding;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        binding = ActivityMainBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());

        binding.toolbar.setTitle(R.string.app_name);
        binding.fab.setOnClickListener(v -> openComposer());
    }
}
```

The binding field lives for the Activity's full lifetime. No nullification needed.

### 3. Fragment

```java
public class ArticleListFragment extends Fragment {

    private FragmentArticleListBinding binding;

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup parent, Bundle s) {
        binding = FragmentArticleListBinding.inflate(inflater, parent, false);
        return binding.getRoot();
    }

    @Override
    public void onViewCreated(View view, Bundle s) {
        binding.list.setLayoutManager(new LinearLayoutManager(requireContext()));
        binding.swipeRefresh.setOnRefreshListener(this::refresh);
    }

    @Override
    public void onDestroyView() {
        super.onDestroyView();
        binding = null; // required: Fragment view lifecycle ends before Fragment
    }

    private void refresh() { /* ... */ }
}
```

**Rule:** always null the binding in `onDestroyView()`. The Fragment instance can outlive its view (back stack, config change).

### 4. Safe Accessor Helper

To avoid the null check at every call site, expose a helper:

```java
public abstract class BindingFragment<B extends ViewBinding> extends Fragment {

    private B binding;

    protected abstract B inflate(LayoutInflater inflater, ViewGroup parent);

    @Override
    public View onCreateView(LayoutInflater i, ViewGroup p, Bundle s) {
        binding = inflate(i, p);
        return binding.getRoot();
    }

    protected B requireBinding() {
        return Objects.requireNonNull(binding, "Fragment view is destroyed");
    }

    @Override
    public void onDestroyView() {
        super.onDestroyView();
        binding = null;
    }
}
```

Subclass:

```java
public class ArticleListFragment extends BindingFragment<FragmentArticleListBinding> {
    @Override
    protected FragmentArticleListBinding inflate(LayoutInflater i, ViewGroup p) {
        return FragmentArticleListBinding.inflate(i, p, false);
    }

    @Override
    public void onViewCreated(View v, Bundle s) {
        requireBinding().list.setAdapter(adapter);
    }
}
```

### 5. RecyclerView Adapter

Bind once per row, in the `ViewHolder` constructor:

```java
public class ArticleAdapter
        extends ListAdapter<Article, ArticleAdapter.VH> {

    public ArticleAdapter() { super(DIFF); }

    static final DiffUtil.ItemCallback<Article> DIFF =
        new DiffUtil.ItemCallback<>() {
            @Override public boolean areItemsTheSame(Article a, Article b) { return a.id() == b.id(); }
            @Override public boolean areContentsTheSame(Article a, Article b) { return a.equals(b); }
        };

    @Override
    public VH onCreateViewHolder(ViewGroup parent, int viewType) {
        ItemArticleBinding b = ItemArticleBinding.inflate(
                LayoutInflater.from(parent.getContext()), parent, false);
        return new VH(b);
    }

    @Override
    public void onBindViewHolder(VH h, int pos) { h.bind(getItem(pos)); }

    static class VH extends RecyclerView.ViewHolder {
        private final ItemArticleBinding b;
        VH(ItemArticleBinding b) { super(b.getRoot()); this.b = b; }

        void bind(Article a) {
            b.title.setText(a.title());
            b.subtitle.setText(a.body());
        }
    }
}
```

### 6. `<include>` Tags and IDs

An `<include>` with an `android:id="@+id/header"` exposes the included layout's binding as `binding.header` whose type is `HeaderBinding`. No `findViewById` chain needed.

## Checklist

- [ ] No new `findViewById` calls in production code.
- [ ] `Fragment.binding` is nulled in `onDestroyView`.
- [ ] Activities do not null their binding (they do not need to).
- [ ] RecyclerView rows bind in `ViewHolder`, not in `onBindViewHolder`, when possible.
- [ ] Shared layouts use `<include>` and consume the nested generated binding.
- [ ] Any `kotlin-android-extensions` / `synthetic` import is removed (never present in a pure-Java project).
