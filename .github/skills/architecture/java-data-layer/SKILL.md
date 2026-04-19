---
name: java-data-layer
description: Expert guidance on building an offline-first data layer in Java using Room, Retrofit, and the Repository pattern. Use this when designing repositories, DAOs, DTO-to-entity mapping, or cache strategies.
---

# Java Data Layer: Repository + Room + Retrofit

## Instructions

The data layer is the single source of truth for domain entities. The UI must never talk to Retrofit or Room directly.

### 1. Domain Contracts

Declare repositories as interfaces in the domain layer. Return domain entities only.

```java
public interface ArticleRepository {
    LiveData<List<Article>> observeArticles();
    ListenableFuture<Void> refresh();
    ListenableFuture<Article> getById(long id);
}
```

### 2. Room DAO (Java annotations)

```java
@Entity(tableName = "articles")
public class ArticleEntity {
    @PrimaryKey public long id;
    public String title;
    public String body;
    public long updatedAt;
}

@Dao
public interface ArticleDao {

    @Query("SELECT * FROM articles ORDER BY updatedAt DESC")
    LiveData<List<ArticleEntity>> observeAll();

    @Query("SELECT * FROM articles WHERE id = :id")
    ListenableFuture<ArticleEntity> findById(long id);

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    ListenableFuture<Void> upsertAll(List<ArticleEntity> items);
}

@Database(entities = {ArticleEntity.class}, version = 1, exportSchema = true)
public abstract class AppDatabase extends RoomDatabase {
    public abstract ArticleDao articleDao();
}
```

Use `androidx.room:room-guava` so DAOs can return `ListenableFuture`. Keep Room entities out of the domain package.

### 3. Retrofit Service

```java
public interface ArticleApi {
    @GET("articles")
    Call<List<ArticleDto>> listArticles(@Query("page") int page);
}

public record ArticleDto(long id, String title, String body, String updatedAt) { }
```

### 4. Mappers

Mappers keep DTOs and persistence entities from leaking out of the data layer.

```java
final class ArticleMappers {
    private ArticleMappers() {}

    static Article toDomain(ArticleEntity e) {
        return new Article(e.id, e.title, e.body, Instant.ofEpochMilli(e.updatedAt));
    }

    static ArticleEntity toEntity(ArticleDto d) {
        ArticleEntity e = new ArticleEntity();
        e.id = d.id();
        e.title = d.title();
        e.body = d.body();
        e.updatedAt = Instant.parse(d.updatedAt()).toEpochMilli();
        return e;
    }
}
```

### 5. Repository Implementation (offline-first)

```java
@Singleton
public class ArticleRepositoryImpl implements ArticleRepository {

    private final ArticleApi api;
    private final ArticleDao dao;
    private final Executor io;

    @Inject
    ArticleRepositoryImpl(ArticleApi api, ArticleDao dao, @IoExecutor Executor io) {
        this.api = api;
        this.dao = dao;
        this.io = io;
    }

    @Override
    public LiveData<List<Article>> observeArticles() {
        return Transformations.map(dao.observeAll(),
                list -> list.stream().map(ArticleMappers::toDomain).toList());
    }

    @Override
    public ListenableFuture<Void> refresh() {
        SettableFuture<Void> result = SettableFuture.create();
        io.execute(() -> {
            try {
                Response<List<ArticleDto>> r = api.listArticles(1).execute();
                if (!r.isSuccessful() || r.body() == null) {
                    throw new IOException("HTTP " + r.code());
                }
                List<ArticleEntity> entities = r.body().stream()
                        .map(ArticleMappers::toEntity).toList();
                dao.upsertAll(entities).get();
                result.set(null);
            } catch (Exception e) {
                result.setException(e);
            }
        });
        return result;
    }

    @Override
    public ListenableFuture<Article> getById(long id) {
        return Futures.transform(dao.findById(id),
                ArticleMappers::toDomain, MoreExecutors.directExecutor());
    }
}
```

### 6. Cache Invalidation

- Serve reads from Room (`LiveData`), triggering refresh as a side effect.
- Use `updatedAt` timestamps to decide when to re-fetch (`NetworkBoundResource` pattern).
- For paging, prefer `Paging 3` with a `RemoteMediator` written in Java.

## Checklist

- [ ] Repository interfaces live in domain; implementations live in data.
- [ ] No Retrofit or Room type escapes the data package.
- [ ] DAOs return `LiveData`, `ListenableFuture`, or `Flowable` — never raw `List` for anything IO-bound.
- [ ] Mappers are package-private and covered by unit tests.
- [ ] `Response` error codes are translated into typed domain errors (`NetworkException`, `NotFoundException`, ...).
- [ ] Schemas are exported (`exportSchema = true`) and committed under `schemas/`.
