---
name: android-navigation-jetpack
description: Expert guidance on using the Jetpack Navigation Component from Java with an XML nav graph, Safe Args, and deep links. Use this when adding screens, passing arguments, or wiring a bottom nav/drawer in a Java project.
---

# Jetpack Navigation Component (Java)

## Instructions

Use the Navigation Component as the single navigation engine. One activity hosts a `NavHostFragment` containing a nav graph; `Fragment`s are destinations.

### 1. Gradle

```groovy
plugins {
    id 'com.android.application'
    id 'androidx.navigation.safeargs'   // Java variant generates *Directions / *Args classes
}

dependencies {
    def nav = "2.8.0"
    implementation "androidx.navigation:navigation-fragment:$nav"
    implementation "androidx.navigation:navigation-ui:$nav"
}
```

The Java Safe Args plugin is `androidx.navigation.safeargs`, **not** `...safeargs.kotlin`.

### 2. Nav Graph

```xml
<!-- res/navigation/main_graph.xml -->
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/main_graph"
    app:startDestination="@id/articleListFragment">

    <fragment
        android:id="@+id/articleListFragment"
        android:name="com.example.app.articles.ArticleListFragment"
        android:label="@string/articles">
        <action
            android:id="@+id/action_list_to_detail"
            app:destination="@id/articleDetailFragment" />
    </fragment>

    <fragment
        android:id="@+id/articleDetailFragment"
        android:name="com.example.app.articles.ArticleDetailFragment"
        android:label="@string/article_detail">
        <argument
            android:name="articleId"
            app:argType="long" />
        <deepLink app:uri="https://example.com/articles/{articleId}" />
    </fragment>
</navigation>
```

### 3. Host in Activity

```xml
<androidx.fragment.app.FragmentContainerView
    android:id="@+id/navHost"
    android:name="androidx.navigation.fragment.NavHostFragment"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:defaultNavHost="true"
    app:navGraph="@navigation/main_graph" />
```

Wire the toolbar:

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle s) {
        super.onCreate(s);
        setContentView(R.layout.activity_main);

        NavHostFragment host = (NavHostFragment)
                getSupportFragmentManager().findFragmentById(R.id.navHost);
        NavController nav = Objects.requireNonNull(host).getNavController();

        setSupportActionBar(findViewById(R.id.toolbar));
        NavigationUI.setupActionBarWithNavController(this, nav);
    }

    @Override
    public boolean onSupportNavigateUp() {
        return Navigation.findNavController(this, R.id.navHost).navigateUp()
                || super.onSupportNavigateUp();
    }
}
```

### 4. Safe Args (Java)

Safe Args generates `ArticleListFragmentDirections` and `ArticleDetailFragmentArgs`.

Navigate:

```java
NavDirections direction =
        ArticleListFragmentDirections.actionListToDetail().setArticleId(article.id());
Navigation.findNavController(view).navigate(direction);
```

Read arguments:

```java
public class ArticleDetailFragment extends Fragment {
    private long articleId;

    @Override
    public void onCreate(Bundle s) {
        super.onCreate(s);
        articleId = ArticleDetailFragmentArgs.fromBundle(requireArguments()).getArticleId();
    }
}
```

If an action has no args, use the generated no-arg factory method directly. Never hand-craft `Bundle` keys.

### 5. Deep Links

Add `<intent-filter>` to the host Activity:

```xml
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="https" android:host="example.com" android:pathPrefix="/articles" />
</intent-filter>
```

The nav graph's `<deepLink>` entries match incoming URIs automatically.

### 6. Bottom Nav / Drawer

```java
BottomNavigationView nav = binding.bottomNav;
NavigationUI.setupWithNavController(nav, navController);
```

Use multiple nav graphs, one per tab, when the tabs have independent back stacks (`NavigationUI.setupWithNavController` supports this for bottom nav).

## Checklist

- [ ] Exactly one `NavHostFragment` per Activity.
- [ ] All destinations are declared in XML; no programmatic `FragmentTransaction.replace` for navigation.
- [ ] Arguments are passed via Safe Args, never via manual `Bundle.putX`.
- [ ] Deep links are declared in the nav graph and tested with `adb shell am start`.
- [ ] `popUpTo` / `popUpToInclusive` are used to clear stacks on logout / finishing flows.
- [ ] Back navigation is handled by `onSupportNavigateUp()` delegating to `NavController`.
