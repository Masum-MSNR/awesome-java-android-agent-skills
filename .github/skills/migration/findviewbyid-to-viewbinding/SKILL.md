---
name: findviewbyid-to-viewbinding
description: Mechanical, safety-oriented migration from findViewById to Jetpack ViewBinding in a Java Android codebase. Use this when modernizing an older project that still uses findViewById throughout Activities, Fragments, and adapters.
---

# Migrating `findViewById` to ViewBinding

## Instructions

`findViewById` returns `View` (pre-API 26 it was unchecked) and relies on string-like IDs resolved at runtime. Every rename, deletion, or typo becomes an `NPE` or `ClassCastException`. ViewBinding replaces this with a generated, null-safe class per layout.

### 1. Enable ViewBinding

`app/build.gradle`:

```groovy
android {
    buildFeatures {
        viewBinding true
    }
}
```

Do not enable DataBinding yet — it has a different migration surface and a heavier annotation processor.

### 2. Layout Preparation

ViewBinding uses the layout filename to generate the class (`activity_main.xml` → `ActivityMainBinding`). Before migrating code:

- Rename layouts so they match their screen (`main.xml` → `activity_main.xml`, `row.xml` → `item_article.xml`). IDE's "Refactor → Rename" keeps resource references in sync.
- Remove unused `<include>` tags; every remaining `<include>` will need a unique id.
- Convert nested `<merge>` usages only after the parent layout has been migrated; a `<merge>` root is bound through the parent.

### 3. Activity Migration

Before:

```java
public class MainActivity extends AppCompatActivity {
    private Toolbar toolbar;
    private RecyclerView list;

    @Override
    protected void onCreate(Bundle s) {
        super.onCreate(s);
        setContentView(R.layout.activity_main);
        toolbar = findViewById(R.id.toolbar);
        list    = findViewById(R.id.list);
        toolbar.setTitle(R.string.app_name);
    }
}
```

After:

```java
public class MainActivity extends AppCompatActivity {
    private ActivityMainBinding binding;

    @Override
    protected void onCreate(Bundle s) {
        super.onCreate(s);
        binding = ActivityMainBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());
        binding.toolbar.setTitle(R.string.app_name);
    }
}
```

Activity bindings live for the Activity's full lifetime — no nullification needed.

### 4. Fragment Migration (most error-prone)

Fragments keep the binding **only while their view is alive**. The Fragment instance itself can outlive multiple `onCreateView` / `onDestroyView` cycles.

```java
public class ArticleListFragment extends Fragment {

    private FragmentArticleListBinding binding;

    @Override
    public View onCreateView(LayoutInflater i, ViewGroup p, Bundle s) {
        binding = FragmentArticleListBinding.inflate(i, p, false);
        return binding.getRoot();
    }

    @Override
    public void onViewCreated(View v, Bundle s) {
        binding.list.setAdapter(adapter);
    }

    @Override
    public void onDestroyView() {
        super.onDestroyView();
        binding = null;   // mandatory
    }
}
```

If you forget `binding = null` in `onDestroyView`, the Fragment holds a reference to its destroyed view hierarchy.

### 5. RecyclerView Adapter Migration

Before:

```java
@Override
public void onBindViewHolder(VH h, int pos) {
    Article a = items.get(pos);
    ((TextView) h.itemView.findViewById(R.id.title)).setText(a.title());
    ((TextView) h.itemView.findViewById(R.id.body)).setText(a.body());
}
```

After:

```java
static class VH extends RecyclerView.ViewHolder {
    final ItemArticleBinding b;
    VH(ItemArticleBinding b) { super(b.getRoot()); this.b = b; }
}

@Override
public VH onCreateViewHolder(ViewGroup parent, int type) {
    return new VH(ItemArticleBinding.inflate(
            LayoutInflater.from(parent.getContext()), parent, false));
}

@Override
public void onBindViewHolder(VH h, int pos) {
    Article a = items.get(pos);
    h.b.title.setText(a.title());
    h.b.body.setText(a.body());
}
```

Binding in `onBindViewHolder` repeatedly is the most common pre-migration hot path. Binding once in the ViewHolder constructor fixes it.

### 6. Custom Views

For a compound view, inflate in the constructor:

```java
public class RatingBar extends ConstraintLayout {
    private final ViewRatingBarBinding b;

    public RatingBar(Context ctx, AttributeSet a) {
        super(ctx, a);
        b = ViewRatingBarBinding.inflate(LayoutInflater.from(ctx), this);
    }
}
```

Note: inflating into `this` uses a `<merge>` root in the layout to avoid an extra level.

### 7. Migration Strategy (large codebase)

1. Enable ViewBinding.
2. Turn on the lint check `UseCompoundDrawables` / add a custom script to list files still using `findViewById`.
3. Migrate **leaf** files first (adapters), then Fragments, then Activities.
4. After each file, run the app's smoke tests; ViewBinding catches renames at compile time.
5. Remove `synthetic` imports (should be none in a Java project; present only if Kotlin was mixed in).
6. Add a lint rule or Spotless custom step that rejects new `findViewById` calls in `src/main`.

## Checklist

- [ ] `viewBinding true` is set in every module that has UI.
- [ ] Every Activity assigns `binding` in `onCreate` before `setContentView`.
- [ ] Every Fragment nulls its binding in `onDestroyView`.
- [ ] RecyclerView ViewHolders hold the binding, not loose view references.
- [ ] No `findViewById` calls remain in `src/main` (verified by `grep -rn 'findViewById' src/main/java`).
- [ ] App launches and all screens render identically after the migration.
