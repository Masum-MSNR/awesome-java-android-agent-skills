---
name: espresso-java
description: Expert guidance on Android UI testing in Java with Espresso, IdlingResource, Hilt test rules, and Robolectric for fast JVM UI tests. Use this when writing instrumented tests, replacing flaky sleeps, or setting up a hermetic UI test harness.
---

# Espresso + Hilt + Robolectric (Java)

## Instructions

Espresso is the default instrumented UI test framework. Robolectric runs the same assertions on the JVM for fast feedback. Hilt test rules make both deterministic.

### 1. Dependencies

```groovy
dependencies {
    // Instrumented
    androidTestImplementation 'androidx.test.ext:junit:1.2.1'
    androidTestImplementation 'androidx.test:runner:1.6.2'
    androidTestImplementation 'androidx.test:rules:1.6.1'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.6.1'
    androidTestImplementation 'androidx.test.espresso:espresso-contrib:3.6.1'
    androidTestImplementation 'androidx.test.espresso:espresso-idling-resource:3.6.1'
    androidTestImplementation 'androidx.fragment:fragment-testing:1.8.3'
    androidTestImplementation 'com.google.dagger:hilt-android-testing:2.52'
    androidTestAnnotationProcessor 'com.google.dagger:hilt-compiler:2.52'

    // JVM UI
    testImplementation 'org.robolectric:robolectric:4.13'
    testImplementation 'androidx.test.ext:junit:1.2.1'
    testImplementation 'androidx.test.espresso:espresso-core:3.6.1'
}

android {
    defaultConfig {
        testInstrumentationRunner "com.example.app.HiltTestRunner"
    }
}
```

### 2. Hilt Test Runner

```java
public class HiltTestRunner extends AndroidJUnitRunner {
    @Override
    public Application newApplication(ClassLoader cl, String name, Context ctx)
            throws ClassNotFoundException, IllegalAccessException, InstantiationException {
        return super.newApplication(cl, HiltTestApplication.class.getName(), ctx);
    }
}
```

### 3. Fragment + Hilt Test

```java
@HiltAndroidTest
@RunWith(AndroidJUnit4.class)
public class ArticleListFragmentTest {

    @Rule(order = 0) public HiltAndroidRule hilt = new HiltAndroidRule(this);

    @BindValue ArticleRepository repository = new FakeArticleRepository();

    @Before
    public void setUp() {
        hilt.inject();
        ((FakeArticleRepository) repository).setArticles(List.of(
            new Article(1, "First", "...", Instant.EPOCH),
            new Article(2, "Second", "...", Instant.EPOCH)
        ));
    }

    @Test
    public void rendersItems() {
        try (FragmentScenario<ArticleListFragment> scenario =
                 FragmentScenario.launchInHiltContainer(ArticleListFragment.class)) {

            onView(withId(R.id.list))
                .check(matches(hasDescendant(withText("First"))))
                .check(matches(hasDescendant(withText("Second"))));
        }
    }

    @Test
    public void clickOpensDetail() {
        try (FragmentScenario<ArticleListFragment> s =
                 FragmentScenario.launchInHiltContainer(ArticleListFragment.class)) {

            onView(withText("First")).perform(click());
            onView(withId(R.id.detailTitle)).check(matches(withText("First")));
        }
    }
}
```

`@BindValue` replaces the production repository via Hilt. `launchInHiltContainer` is a small helper you write once that hosts the fragment in a Hilt-injected test Activity.

### 4. IdlingResource — eliminate sleeps

Espresso waits for the main thread to be idle **and** all registered IdlingResources to be idle. Register one per background worker.

```java
public class CountingIdlingResource implements IdlingResource {

    private final AtomicInteger counter = new AtomicInteger();
    private volatile ResourceCallback callback;

    public void increment() { counter.incrementAndGet(); }
    public void decrement() {
        if (counter.decrementAndGet() == 0 && callback != null) {
            callback.onTransitionToIdle();
        }
    }

    @Override public String getName() { return "App-IO"; }
    @Override public boolean isIdleNow() { return counter.get() == 0; }
    @Override public void registerIdleTransitionCallback(ResourceCallback c) {
        this.callback = c;
    }
}
```

Wrap your executor's submit:

```java
public class TestableExecutor implements Executor {
    private final Executor delegate;
    private final CountingIdlingResource idling;

    @Override public void execute(Runnable r) {
        idling.increment();
        delegate.execute(() -> {
            try { r.run(); }
            finally { idling.decrement(); }
        });
    }
}
```

Register in the test:

```java
IdlingRegistry.getInstance().register(idling);
```

**Never use `Thread.sleep(...)` in UI tests.**

### 5. Espresso Matchers & Actions

```java
onView(withId(R.id.email)).perform(typeText("test@example.com"), closeSoftKeyboard());
onView(withId(R.id.submit)).perform(click());
onView(withId(R.id.result)).check(matches(isDisplayed()));
onView(withText(R.string.welcome)).check(matches(isDisplayed()));
onView(allOf(withId(R.id.tile), hasDescendant(withText("Premium"))))
    .perform(click());
```

For RecyclerView:

```java
onView(withId(R.id.list))
    .perform(RecyclerViewActions.actionOnItemAtPosition(2, click()));
```

### 6. Robolectric for JVM UI Tests

Robolectric runs Espresso-like tests on the JVM in milliseconds:

```java
@RunWith(RobolectricTestRunner.class)
@Config(sdk = 33)
public class ArticleAdapterTest {

    @Test
    public void bindsTitle() {
        ViewGroup parent = new FrameLayout(ApplicationProvider.getApplicationContext());
        ItemArticleBinding b = ItemArticleBinding.inflate(
                LayoutInflater.from(parent.getContext()), parent, false);

        ArticleAdapter.VH vh = new ArticleAdapter.VH(b);
        vh.bind(new Article(1, "Title", "Body", Instant.EPOCH));

        assertThat(b.title.getText().toString()).isEqualTo("Title");
    }
}
```

Keep Robolectric for **widget-level** tests. Full user journeys should run on a real emulator so animations, input, and lifecycle match production.

### 7. Flaky Test Hygiene

- Run every UI test with `adb shell settings put global window_animation_scale 0` (and transition, animator scales). CI scripts must enforce this.
- Quarantine flakes in a `@Ignore("flaky")` bucket, not in the main suite.
- Record failing runs via `--enable-record` to capture screenshots / video.

## Checklist

- [ ] Every Fragment/Activity under test runs in a Hilt test container with a `@BindValue` fake.
- [ ] Background work is gated by an `IdlingResource`.
- [ ] `Thread.sleep` does not appear in any UI test.
- [ ] Animations are disabled during instrumented test runs.
- [ ] Critical widgets have Robolectric tests for fast feedback.
- [ ] Test runner is the custom `HiltTestRunner` so `HiltTestApplication` is used.
- [ ] Shared fakes (FakeRepository, FakeClock) live under `testFixtures/` and are reused across tests.
