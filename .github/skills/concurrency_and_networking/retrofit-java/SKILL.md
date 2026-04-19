---
name: retrofit-java
description: Expert guidance on Retrofit 2 + OkHttp 4 from Java — services, converters, interceptors, call adapters, timeouts, and error body parsing. Use this when building or reviewing the networking layer in a Java Android app.
---

# Retrofit + OkHttp (Java)

## Instructions

Retrofit defines HTTP services as Java interfaces; OkHttp is the client under it. One `OkHttpClient` per app, one `Retrofit` per base URL, services are stateless singletons provided by Hilt.

### 1. Dependencies

```groovy
dependencies {
    implementation 'com.squareup.retrofit2:retrofit:2.11.0'
    implementation 'com.squareup.retrofit2:converter-moshi:2.11.0'
    implementation 'com.squareup.retrofit2:adapter-rxjava3:2.11.0'   // optional
    implementation 'com.squareup.okhttp3:okhttp:4.12.0'
    implementation 'com.squareup.okhttp3:logging-interceptor:4.12.0'

    implementation 'com.squareup.moshi:moshi:1.15.1'
    implementation 'com.squareup.moshi:moshi-adapters:1.15.1'
    annotationProcessor 'com.squareup.moshi:moshi-kotlin-codegen:1.15.1' // only if codegen
}
```

Moshi is the preferred JSON converter for new code. Gson is acceptable for existing code; do not mix within a single module.

### 2. Service Interface

```java
public interface ArticleApi {

    @GET("articles")
    Call<List<ArticleDto>> listArticles(@Query("page") int page,
                                        @Query("limit") int limit);

    @GET("articles/{id}")
    Call<ArticleDto> getArticle(@Path("id") long id);

    @POST("articles")
    Call<ArticleDto> createArticle(@Body CreateArticleRequest body);

    @Multipart
    @POST("articles/{id}/cover")
    Call<Void> uploadCover(@Path("id") long id,
                           @Part MultipartBody.Part file);
}
```

Models are `record`s (Java 17):

```java
public record ArticleDto(long id, String title, String body,
                         Instant updatedAt) { }
```

With Moshi + the `InstantAdapter` below, `record`s work out of the box via `reflect` adapter.

### 3. Building Retrofit

```java
@Module
@InstallIn(SingletonComponent.class)
public abstract class NetworkModule {

    @Provides @Singleton
    static Moshi provideMoshi() {
        return new Moshi.Builder()
                .add(Instant.class, new Rfc3339InstantAdapter())
                .add(new KotlinJsonAdapterFactory())     // only if Kotlin classes mixed in
                .build();
    }

    @Provides @Singleton
    static OkHttpClient provideOkHttp(AuthInterceptor auth) {
        HttpLoggingInterceptor log = new HttpLoggingInterceptor();
        log.setLevel(BuildConfig.DEBUG
                ? HttpLoggingInterceptor.Level.BODY
                : HttpLoggingInterceptor.Level.NONE);

        return new OkHttpClient.Builder()
                .connectTimeout(10, TimeUnit.SECONDS)
                .readTimeout(30, TimeUnit.SECONDS)
                .callTimeout(60, TimeUnit.SECONDS)
                .retryOnConnectionFailure(true)
                .addInterceptor(auth)
                .addInterceptor(log)
                .build();
    }

    @Provides @Singleton
    static Retrofit provideRetrofit(OkHttpClient client, Moshi moshi) {
        return new Retrofit.Builder()
                .baseUrl(BuildConfig.API_BASE_URL)
                .client(client)
                .addConverterFactory(MoshiConverterFactory.create(moshi))
                .build();
    }

    @Provides @Singleton
    static ArticleApi provideArticleApi(Retrofit r) {
        return r.create(ArticleApi.class);
    }
}
```

### 4. Auth Interceptor

```java
@Singleton
public class AuthInterceptor implements Interceptor {

    private final TokenStore tokens;

    @Inject
    AuthInterceptor(TokenStore tokens) { this.tokens = tokens; }

    @NonNull
    @Override
    public Response intercept(@NonNull Chain chain) throws IOException {
        Request req = chain.request();
        String token = tokens.current();
        if (token != null) {
            req = req.newBuilder()
                    .header("Authorization", "Bearer " + token)
                    .build();
        }
        return chain.proceed(req);
    }
}
```

Never log tokens. The logging interceptor's `HEADERS` level redacts values added via `redactHeader("Authorization")`:

```java
log.redactHeader("Authorization");
log.redactHeader("Cookie");
```

### 5. Error Bodies

```java
public record ApiError(String code, String message) { }

public static ApiError parseError(Response<?> response, Moshi moshi) {
    ResponseBody body = response.errorBody();
    if (body == null) return new ApiError("unknown", "empty");
    try {
        return moshi.adapter(ApiError.class).fromJson(body.source());
    } catch (IOException e) {
        return new ApiError("parse_error", e.getMessage());
    }
}
```

Map to typed domain exceptions at the repository boundary:

```java
public Article getArticle(long id) throws NetworkException, NotFoundException {
    try {
        Response<ArticleDto> r = api.getArticle(id).execute();
        if (r.code() == 404) throw new NotFoundException("article " + id);
        if (!r.isSuccessful()) throw new NetworkException(parseError(r, moshi));
        return ArticleMappers.toDomain(r.body());
    } catch (IOException e) {
        throw new NetworkException(e);
    }
}
```

### 6. Call Adapters

- Default: `Call<T>` — call `.enqueue(...)` or `.execute()`.
- RxJava 3: `Single<T>`, `Completable`, `Flowable<T>` via `RxJava3CallAdapterFactory`.
- Guava: `ListenableFuture<T>` via `GuavaCallAdapterFactory`.

Pick one per module and keep it consistent.

### 7. Caching and Offline

OkHttp has a built-in HTTP cache. Install it and pair with `Cache-Control` headers:

```java
File cacheDir = new File(context.getCacheDir(), "http");
Cache cache = new Cache(cacheDir, 10L * 1024 * 1024);  // 10MB
builder.cache(cache);
```

For authentication-gated APIs, manage cache busting via `ETag` / `If-None-Match` rather than disabling cache entirely.

### 8. TLS & Certificate Pinning

```java
CertificatePinner pinner = new CertificatePinner.Builder()
    .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
    .build();
builder.certificatePinner(pinner);
```

Rotate pins before expiry. Commit the pin and its expected rotation date to source control.

## Checklist

- [ ] Exactly one `OkHttpClient` and one `Retrofit` per base URL, both singletons.
- [ ] `HttpLoggingInterceptor` is `NONE` in release builds.
- [ ] Sensitive headers are redacted via `redactHeader`.
- [ ] All HTTP errors are mapped to typed domain exceptions; UI never sees `Response` codes.
- [ ] Timeouts are set (`connect`, `read`, `call`).
- [ ] `Cache` is configured for idempotent GETs where appropriate.
- [ ] Service interfaces return the call adapter chosen for the module consistently.
