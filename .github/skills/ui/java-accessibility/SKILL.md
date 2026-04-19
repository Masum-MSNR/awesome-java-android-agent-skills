---
name: java-accessibility
description: Expert guidance on Android accessibility from Java — contentDescription, TalkBack, touch targets, dynamic text sizing, and Accessibility Scanner. Use this when building UI or auditing an existing screen for a11y.
---

# Android Accessibility (Java)

## Instructions

Accessibility is a correctness requirement, not a polish item. Treat Accessibility Scanner findings like lint errors.

### 1. Content Descriptions

Every meaningful non-text view needs a `contentDescription`. Decorative views set it to `null` and `android:importantForAccessibility="no"`.

```xml
<ImageView
    android:id="@+id/avatar"
    android:layout_width="40dp"
    android:layout_height="40dp"
    android:contentDescription="@string/cd_user_avatar" />

<ImageView
    android:id="@+id/divider_icon"
    android:contentDescription="@null"
    android:importantForAccessibility="no" />
```

For dynamic descriptions (avatars that show a specific user) set the value in code:

```java
binding.avatar.setContentDescription(
        getString(R.string.cd_avatar_format, user.displayName()));
```

### 2. Touch Targets

Minimum **48dp × 48dp** tappable area. Use `minWidth` / `minHeight`, or expand with `TouchDelegate`:

```java
binding.root.post(() -> {
    Rect r = new Rect();
    binding.smallIcon.getHitRect(r);
    int pad = (int) (8 * getResources().getDisplayMetrics().density);
    r.inset(-pad, -pad);
    binding.root.setTouchDelegate(new TouchDelegate(r, binding.smallIcon));
});
```

### 3. Dynamic Text

- Always size text in `sp`, never `dp`/`px`.
- Respect the user's font scale. Test at 200% and 100%.
- Use `android:autoSizeTextType="uniform"` cautiously — it breaks screen reader text parsing for long content.

### 4. Roles and State

Use AndroidX `ViewCompat.setAccessibilityDelegate`:

```java
ViewCompat.setAccessibilityDelegate(binding.card, new AccessibilityDelegateCompat() {
    @Override
    public void onInitializeAccessibilityNodeInfo(View host,
                                                  AccessibilityNodeInfoCompat info) {
        super.onInitializeAccessibilityNodeInfo(host, info);
        info.setClassName(Button.class.getName());
        info.setRoleDescription(getString(R.string.role_article_card));
        info.addAction(new AccessibilityNodeInfoCompat.AccessibilityActionCompat(
                AccessibilityNodeInfoCompat.ACTION_CLICK,
                getString(R.string.action_open_article)));
    }
});
```

This is how you expose custom semantics to TalkBack.

### 5. Focus and Reading Order

- `android:focusable="true"`, `android:focusableInTouchMode="true"` on non-native interactive views.
- Use `android:nextFocusForward`, `android:nextFocusDown` to fix visual-vs-logical ordering.
- Group related views with a single clickable parent and `android:importantForAccessibility="no"` on leaves.

```xml
<LinearLayout
    android:id="@+id/listItem"
    android:clickable="true"
    android:focusable="true"
    android:contentDescription="@string/cd_article_item">

    <ImageView android:importantForAccessibility="no" ... />
    <TextView android:importantForAccessibility="no" ... />
</LinearLayout>
```

### 6. Color and Contrast

- Text contrast at least 4.5:1 for body, 3:1 for large text. Verify with Accessibility Scanner or the Material 3 contrast validator.
- Never rely on color alone to convey state — add an icon or label ("Required").

### 7. Live Regions

Announce async updates:

```java
ViewCompat.setAccessibilityLiveRegion(binding.status,
        ViewCompat.ACCESSIBILITY_LIVE_REGION_POLITE);
binding.status.setText(R.string.loading_articles);
```

Or fire a direct announcement:

```java
binding.root.announceForAccessibility(getString(R.string.a11y_saved));
```

### 8. TalkBack Testing Flow

1. Enable TalkBack (Settings → Accessibility).
2. Swipe right through the screen; every actionable element must announce role + label.
3. Enable "Switch Access" and ensure full keyboard reachability.
4. Run Accessibility Scanner on every new screen.

## Checklist

- [ ] Every interactive view has a meaningful `contentDescription` or labelled text.
- [ ] All tap targets are at least 48dp × 48dp (or use `TouchDelegate`).
- [ ] Text sizes use `sp`; layouts survive 200% font scale.
- [ ] Grouped list items expose a single focusable node with a combined description.
- [ ] Custom views use `AccessibilityDelegateCompat` to advertise role and actions.
- [ ] Contrast is at least 4.5:1 for body text.
- [ ] Accessibility Scanner reports zero errors on every primary screen.
