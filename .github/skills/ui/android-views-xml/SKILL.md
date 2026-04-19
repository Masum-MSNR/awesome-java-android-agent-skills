---
name: android-views-xml
description: Expert guidance on authoring modern Android XML layouts with ConstraintLayout, the Material 3 style/theme system, and proper resource organisation. Use this when creating or reviewing layout, dimens, colors, or themes XML.
---

# Android XML Layouts & Theming

## Instructions

The XML view system remains the canonical Android UI toolkit for Java projects. Keep layouts flat, use `ConstraintLayout` for complex screens, and push repeated attributes into `styles` and `themes`.

### 1. ConstraintLayout First

Prefer a single `ConstraintLayout` root over nested `LinearLayout`s. Chains, barriers, guidelines, and groups replace most nesting.

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.google.android.material.appbar.MaterialToolbar
        android:id="@+id/toolbar"
        style="?attr/toolbarStyle"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/list"
        android:layout_width="0dp"
        android:layout_height="0dp"
        app:layout_constraintTop_toBottomOf="@id/toolbar"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

Rules:
- Width/height `0dp` + constraints on both sides is how you say "fill between these anchors".
- Use `app:layout_constraintHorizontal_chainStyle="packed"` for chains.
- Use `<Barrier>` instead of `RelativeLayout`-style `alignBaseline` tricks.

### 2. Include, Merge, ViewStub

- `<include>` shares repeated sub-layouts.
- `<merge>` eliminates a redundant root when the parent already provides one.
- `<ViewStub>` defers inflation of heavy optional sections (e.g. empty state, error card).

```xml
<ViewStub
    android:id="@+id/emptyStub"
    android:layout="@layout/view_empty_articles"
    android:inflatedId="@+id/empty"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:layout_constraintTop_toTopOf="parent"
    app:layout_constraintBottom_toBottomOf="parent"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintEnd_toEndOf="parent" />
```

### 3. Styles, Themes, and Attributes

Split concerns:
- `themes.xml` — app/activity-wide tokens (`colorPrimary`, `colorSurface`, typography).
- `styles.xml` — component-level reusable styles that reference theme attributes.

```xml
<!-- themes.xml -->
<style name="Theme.MyApp" parent="Theme.Material3.DayNight.NoActionBar">
    <item name="colorPrimary">@color/brand_primary</item>
    <item name="colorOnPrimary">@color/white</item>
    <item name="toolbarStyle">@style/Widget.MyApp.Toolbar</item>
    <item name="android:textAppearanceBody1">@style/TextAppearance.MyApp.Body</item>
</style>

<!-- styles.xml -->
<style name="Widget.MyApp.Toolbar" parent="Widget.Material3.Toolbar">
    <item name="titleTextAppearance">@style/TextAppearance.MyApp.Title</item>
    <item name="android:background">?attr/colorSurface</item>
</style>

<style name="TextAppearance.MyApp.Title" parent="TextAppearance.Material3.TitleLarge">
    <item name="android:textColor">?attr/colorOnSurface</item>
</style>
```

Always reference theme attributes (`?attr/colorPrimary`) instead of hard-coded `@color/...` inside reusable styles — it lets the same widget adapt to dark mode, themed flavors, and per-screen overlays.

### 4. Resource Qualifiers

- Dark mode: `values-night/`
- RTL: `values-ldrtl/`, `layout-ldrtl/`
- Screen width: `values-sw600dp/`, `layout-sw600dp/`
- Fold/large screen: `values-w840dp/`

Do not duplicate entire layouts for minor tweaks — use `ConstraintSet` or `<layout>` with guidelines keyed on `@dimen/` values per qualifier.

### 5. Data-Driven Visibility

Avoid writing "loading"/"error"/"content" as three separate layouts. Toggle via `Group`:

```xml
<androidx.constraintlayout.widget.Group
    android:id="@+id/contentGroup"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:constraint_referenced_ids="list,toolbar,fab" />
```

## Checklist

- [ ] No layout file has `tools:ignore="HardcodedText"` — every user-visible string is a `@string/` resource.
- [ ] No `px` or raw dimensions in layouts; use `@dimen/` or `dp`/`sp` units.
- [ ] No `wrap_content` + `0dp` mix without constraints on both sides.
- [ ] Style attributes reference `?attr/` theme tokens, not hard-coded colors.
- [ ] Heavy optional sections use `<ViewStub>`.
- [ ] `themes.xml` has a `-night` variant and the app looks correct in dark mode.
