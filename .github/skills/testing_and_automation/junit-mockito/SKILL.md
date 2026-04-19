---
name: junit-mockito
description: Expert guidance on unit testing Java Android code with JUnit, Mockito, and parameterized tests for ViewModels, repositories, and use cases. Use this when writing or reviewing JVM-side unit tests.
---

# JUnit + Mockito for Java Android

## Instructions

Write unit tests as the first consumer of your architecture. If testing is painful, the architecture is wrong — fix the design before papering over it with fakes.

### 1. Dependencies

```groovy
dependencies {
    testImplementation 'junit:junit:4.13.2'
    testImplementation 'org.mockito:mockito-core:5.12.0'
    testImplementation 'org.mockito:mockito-inline:5.2.0'   // final classes, static methods
    testImplementation 'androidx.arch.core:core-testing:2.2.0'  // InstantTaskExecutorRule
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.2' // optional, pure Java modules
    testImplementation 'com.google.truth:truth:1.4.4'       // readable assertions
}
```

For Robolectric-free ViewModel tests, `core-testing`'s `InstantTaskExecutorRule` is mandatory.

### 2. ViewModel Test

```java
public class ArticleListViewModelTest {

    @Rule public InstantTaskExecutorRule instant = new InstantTaskExecutorRule();
    @Rule public MockitoRule mockito = MockitoJUnit.rule();

    @Mock GetArticles getArticles;

    private ArticleListViewModel viewModel;

    @Before
    public void setUp() {
        when(getArticles.execute()).thenReturn(List.of(
            new Article(1, "One", "...", Instant.EPOCH),
            new Article(2, "Two", "...", Instant.EPOCH)
        ));
        viewModel = new ArticleListViewModel(getArticles);
    }

    @Test
    public void emitsSuccessStateAfterLoad() throws InterruptedException {
        ArticleListUiState s = LiveDataTestUtil.awaitValue(viewModel.state());
        assertThat(s.loading()).isFalse();
        assertThat(s.articles()).hasSize(2);
        assertThat(s.errorMessage()).isNull();
    }
}
```

`LiveDataTestUtil` is a small helper that uses a `CountDownLatch` to capture the first non-initial value. Avoid reading `.getValue()` directly — it depends on observer timing.

### 3. Repository Test with Mockito

```java
@RunWith(MockitoJUnitRunner.class)
public class ArticleRepositoryImplTest {

    @Mock ArticleApi api;
    @Mock ArticleDao dao;

    private ArticleRepositoryImpl repo;
    private final ExecutorService io = Executors.newSingleThreadExecutor();

    @Before
    public void setUp() {
        repo = new ArticleRepositoryImpl(api, dao, io);
    }

    @Test
    public void refreshPersistsEntitiesOnSuccess() throws Exception {
        ArticleDto dto = new ArticleDto(1, "Title", "Body", Instant.EPOCH.toString());
        Response<List<ArticleDto>> ok = Response.success(List.of(dto));

        Call<List<ArticleDto>> call = mock(Call.class);
        when(call.execute()).thenReturn(ok);
        when(api.listArticles(1)).thenReturn(call);
        when(dao.upsertAll(any())).thenReturn(Futures.immediateFuture(null));

        repo.refresh().get(1, TimeUnit.SECONDS);

        ArgumentCaptor<List<ArticleEntity>> captor = ArgumentCaptor.forClass(List.class);
        verify(dao).upsertAll(captor.capture());
        assertThat(captor.getValue()).hasSize(1);
        assertThat(captor.getValue().get(0).title).isEqualTo("Title");
    }
}
```

Use `ArgumentCaptor` to verify **what** you persisted, not just that a call happened.

### 4. Parameterized Tests

JUnit 4:

```java
@RunWith(Parameterized.class)
public class PriceFormatterTest {

    @Parameters(name = "{0} → {1}")
    public static Collection<Object[]> data() {
        return List.of(new Object[][] {
            { new BigDecimal("0"),     "$0.00" },
            { new BigDecimal("1"),     "$1.00" },
            { new BigDecimal("1.999"), "$2.00" },
        });
    }

    @Parameter(0) public BigDecimal input;
    @Parameter(1) public String expected;

    @Test public void formats() {
        assertThat(PriceFormatter.format(input, Locale.US)).isEqualTo(expected);
    }
}
```

JUnit 5:

```java
@ParameterizedTest
@CsvSource({
    "0,       $0.00",
    "1,       $1.00",
    "1.999,   $2.00"
})
void formats(String input, String expected) {
    assertThat(PriceFormatter.format(new BigDecimal(input), Locale.US))
        .isEqualTo(expected);
}
```

### 5. Mockito Gotchas

- Use `mockito-inline` to mock `final` classes (`OkHttpClient`, Retrofit services prior to interface-based).
- `when(...).thenReturn(...)` + `doReturn(...).when(...)` — prefer `when/thenReturn` except for spies.
- `verify(mock, times(1))` is the default; omit the count.
- `@Spy` over `@Mock` when you want real behavior with the option to override.

### 6. Hermetic Tests

- Never hit the real network or real Room in unit tests.
- Use `java.time.Clock` as an injected dependency; test with `Clock.fixed(...)`.
- Seed randomness: take `Random random` via constructor and inject `new Random(42L)` in tests.

### 7. Code Coverage

```groovy
plugins { id 'jacoco' }

android {
    buildTypes {
        debug {
            testCoverageEnabled true
        }
    }
}
```

Target: 80% line coverage on the domain and data layers, lower for UI. Coverage is a floor; reject drops in CI.

## Checklist

- [ ] Tests run without an emulator (JVM only) for everything below the UI layer.
- [ ] `InstantTaskExecutorRule` is used whenever `LiveData` is under test.
- [ ] No test hits real IO (network, file system, Room).
- [ ] Every mock interaction is verified or `verifyNoInteractions(...)` is called.
- [ ] Time, random, and UUIDs come from injectable providers.
- [ ] JUnit/Mockito versions are consistent across modules (no RxJava2 vs RxJava3 split).
- [ ] CI fails the build on coverage regression below the agreed threshold.
