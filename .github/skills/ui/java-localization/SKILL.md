---
name: java-localization
description: Expert guidance on localizing Android apps from Java — strings.xml variants, plurals, ICU MessageFormat, RTL, and per-app locale. Use this when adding a language, handling pluralization, or enabling runtime language switching.
---

# Android Localization (Java)

## Instructions

Every user-visible string must be a resource. Code that concatenates localized fragments is almost always wrong.

### 1. `strings.xml` Variants

```
res/
├── values/strings.xml        // default (usually en)
├── values-es/strings.xml     // Spanish
├── values-fr/strings.xml     // French
├── values-de/strings.xml     // German
└── values-ar/strings.xml     // Arabic (RTL)
```

Keep keys descriptive, not content-shaped (`cta_sign_in`, not `button_text`). Put translator context in comments:

```xml
<!-- Shown as a button in the login screen -->
<string name="cta_sign_in">Sign in</string>
```

### 2. String Formatting

Use positional format specifiers so translators can reorder arguments:

```xml
<string name="greeting">Hello, %1$s. You have %2$d new messages.</string>
```

```java
binding.greeting.setText(getString(R.string.greeting, user.name(), unreadCount));
```

Never build strings with `+` or `String.format` without positional indices.

### 3. Plurals

```xml
<plurals name="items_count">
    <item quantity="one">%d item</item>
    <item quantity="other">%d items</item>
</plurals>
```

```java
String label = getResources().getQuantityString(
        R.plurals.items_count, count, count);
```

Do not invent manual `if (count == 1)` branches — plural rules differ per language (Polish, Russian, Arabic have multiple forms).

### 4. ICU MessageFormat

For complex patterns (gender, nested plurals), use ICU MessageFormat via `android.icu`:

```xml
<string name="notification_summary">{count, plural,
    one {You have # new message from {sender}}
    other {You have # new messages from {sender}}
}</string>
```

```java
String pattern = getString(R.string.notification_summary);
MessageFormat fmt = new MessageFormat(pattern, Locale.getDefault());
Map<String, Object> args = Map.of("count", count, "sender", user.name());
String text = fmt.format(args);
```

### 5. RTL Support

- Set `android:supportsRtl="true"` in `AndroidManifest.xml`.
- Use **start/end** attributes instead of left/right: `paddingStart`, `marginEnd`, `drawableStart`.
- Mirror directional drawables in `drawable-ldrtl/` or set `android:autoMirrored="true"` on vector drawables.
- Test with Developer Options → "Force RTL layout direction".

```xml
<ImageView
    android:src="@drawable/ic_chevron"
    android:autoMirrored="true" />
```

### 6. Date, Time, Currency, Numbers

Never hard-code formats. Use:

```java
NumberFormat currency = NumberFormat.getCurrencyInstance(Locale.getDefault());
currency.setCurrency(Currency.getInstance("EUR"));
String price = currency.format(product.price());

DateTimeFormatter df = DateTimeFormatter
        .ofLocalizedDate(FormatStyle.MEDIUM)
        .withLocale(Locale.getDefault());
String date = df.format(article.publishedAt());
```

### 7. Per-App Locale (API 33+)

```java
AppCompatDelegate.setApplicationLocales(
        LocaleListCompat.forLanguageTags("es-ES"));
```

Declare supported locales in `build.gradle`:

```groovy
android {
    defaultConfig {
        resourceConfigurations += ['en', 'es', 'fr', 'de', 'ar']
    }
}
```

And create `res/xml/locales_config.xml` referenced from the manifest:

```xml
<application
    android:localeConfig="@xml/locales_config">
</application>
```

### 8. Pseudo-localization (testing)

Enable in Developer Options → "Pseudolocales". `values-en-rXA/` inflates every string and `values-ar-rXB/` reverses direction. Catch truncation and RTL bugs before your translator does.

## Checklist

- [ ] No hard-coded strings in layouts or Java. `lint` rule `HardcodedText` is enabled.
- [ ] Plurals use `<plurals>`, never `if (count == 1)`.
- [ ] Formatting uses positional `%1$s` specifiers.
- [ ] Layouts use `Start` / `End` attributes; no `Left` / `Right`.
- [ ] `android:supportsRtl="true"` and the app is verified in Force RTL mode.
- [ ] Dates, numbers, and currencies use `java.text` / `java.time` with `Locale.getDefault()`.
- [ ] `resourceConfigurations` lists only the locales you actually ship (reduces APK size).
