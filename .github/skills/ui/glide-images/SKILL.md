---
name: glide-images
description: Expert guidance on using Glide 4 from Java for image loading, caching, placeholders, thumbnails, and transformations. Use this when loading remote or local images on Android.
---

# Glide 4 Image Loading (Java)

## Instructions

Glide is the default image loader. It handles lifecycle, memory cache, disk cache, and decoding off the main thread. Prefer Glide over Picasso for new code; prefer it over Coil when the project must stay Java.

### 1. Setup

```groovy
dependencies {
    implementation 'com.github.bumptech.glide:glide:4.16.0'
    annotationProcessor 'com.github.bumptech.glide:compiler:4.16.0'
}
```

With the annotation processor you get a generated `GlideApp` class. Register a module for app-wide defaults:

```java
@GlideModule
public class AppGlideModule extends com.bumptech.glide.module.AppGlideModule {

    @Override
    public void applyOptions(@NonNull Context context, @NonNull GlideBuilder builder) {
        builder.setDefaultRequestOptions(new RequestOptions()
                .format(DecodeFormat.PREFER_RGB_565)
                .diskCacheStrategy(DiskCacheStrategy.AUTOMATIC));
        builder.setMemoryCache(new LruResourceCache(1024L * 1024L * 40L)); // 40MB
    }

    @Override
    public boolean isManifestParsingEnabled() { return false; }
}
```

### 2. Basic Load

Always pass a lifecycle-aware scope: pass `this` for Activity, `this` (Fragment), or the `View`.

```java
Glide.with(imageView)
     .load(article.imageUrl())
     .placeholder(R.drawable.placeholder_article)
     .error(R.drawable.error_article)
     .into(imageView);
```

Never call `Glide.with(getApplicationContext())` — it disables lifecycle cancellation and leaks.

### 3. Transformations

```java
Glide.with(imageView)
     .load(user.avatarUrl())
     .transform(new CenterCrop(), new RoundedCorners(16))
     .into(imageView);
```

Common transformations:
- `CenterCrop`, `CenterInside`, `FitCenter`
- `RoundedCorners(px)`
- `CircleCrop()` for avatars
- Custom `BitmapTransformation` for effects (blur, grayscale) — override `transform()` and return a new `Bitmap`.

### 4. Thumbnails (fast perceived load)

```java
RequestBuilder<Drawable> fullSize = Glide.with(imageView).load(article.imageUrl());
RequestBuilder<Drawable> thumb    = Glide.with(imageView).load(article.thumbUrl());

fullSize.thumbnail(thumb).into(imageView);
```

Or a fractional thumbnail of the same URL:

```java
Glide.with(imageView)
     .load(url)
     .thumbnail(0.25f)
     .into(imageView);
```

### 5. RecyclerView

Cancel outstanding loads on view recycle:

```java
@Override
public void onBindViewHolder(VH h, int pos) {
    Article a = getItem(pos);
    Glide.with(h.b.image)
         .load(a.imageUrl())
         .centerCrop()
         .placeholder(R.color.image_placeholder)
         .into(h.b.image);
}

@Override
public void onViewRecycled(@NonNull VH h) {
    super.onViewRecycled(h);
    Glide.with(h.b.image).clear(h.b.image);
}
```

Set fixed sizes (`android:layout_width`, `android:layout_height`) so Glide downsamples to the correct resolution. Avoid `wrap_content` on image views.

### 6. Preloading

For paging / detail screens, preload upcoming images:

```java
RecyclerViewPreloader<Article> preloader = new RecyclerViewPreloader<>(
        Glide.with(this),
        new ListPreloader.PreloadModelProvider<>() {
            @Override public List<Article> getPreloadItems(int position) {
                return List.of(adapter.getItem(position));
            }
            @Override public RequestBuilder<?> getPreloadRequestBuilder(Article a) {
                return Glide.with(recyclerView).load(a.imageUrl());
            }
        },
        new ViewPreloadSizeProvider<>(),
        10);
recyclerView.addOnScrollListener(preloader);
```

### 7. Disk / Memory Cache

- `DiskCacheStrategy.AUTOMATIC` (default) is correct for most apps.
- Use `DiskCacheStrategy.NONE` for one-off blobs (e.g. captchas).
- Use `DiskCacheStrategy.ALL` for assets served with stable URLs.
- `.skipMemoryCache(true)` for debug overlays.

Clear caches on sign-out:

```java
new Thread(() -> Glide.get(context).clearDiskCache()).start();
Glide.get(context).clearMemory();
```

## Checklist

- [ ] `Glide.with(...)` is called with a lifecycle-bound owner, not `applicationContext`.
- [ ] Every `ImageView` has a fixed size or explicit constraint so Glide can downsample.
- [ ] Placeholders and error drawables are set for network loads.
- [ ] `Glide.with(view).clear(view)` is called in `onViewRecycled` for long lists.
- [ ] Avatars use `CircleCrop()`, not a rounded background + mask hack.
- [ ] `AppGlideModule` sets reasonable cache sizes and default `DecodeFormat`.
