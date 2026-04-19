---
name: android-animations-java
description: Expert guidance on Android animations in Java using ObjectAnimator, ValueAnimator, MotionLayout, and Transitions. Use this when building transitions, micro-interactions, or shared-element flows.
---

# Android Animations in Java

## Instructions

Prefer the property-animation system (`android.animation.*`) over the legacy view animations (`android.view.animation.*`). Prefer declarative tools (MotionLayout, Transitions) for multi-view choreography.

### 1. ObjectAnimator (most cases)

```java
ObjectAnimator fadeIn = ObjectAnimator.ofFloat(binding.card, View.ALPHA, 0f, 1f);
fadeIn.setDuration(200);
fadeIn.setInterpolator(new FastOutSlowInInterpolator());
fadeIn.start();
```

Run several in parallel with `AnimatorSet`:

```java
AnimatorSet set = new AnimatorSet();
set.playTogether(
        ObjectAnimator.ofFloat(binding.card, View.TRANSLATION_Y, 40f, 0f),
        ObjectAnimator.ofFloat(binding.card, View.ALPHA, 0f, 1f)
);
set.setDuration(220);
set.setInterpolator(new FastOutSlowInInterpolator());
set.start();
```

### 2. ValueAnimator (custom properties)

```java
ValueAnimator anim = ValueAnimator.ofArgb(
        ContextCompat.getColor(ctx, R.color.surface),
        ContextCompat.getColor(ctx, R.color.primary));
anim.setDuration(300);
anim.addUpdateListener(a -> binding.root.setBackgroundColor((int) a.getAnimatedValue()));
anim.start();
```

Use `TypeEvaluator` for domain types (e.g. interpolating `Rect`, `PointF`, custom data).

### 3. ViewPropertyAnimator (fluent one-offs)

```java
binding.fab.animate()
    .translationY(-16f)
    .alpha(1f)
    .setDuration(180)
    .setInterpolator(new FastOutSlowInInterpolator())
    .withEndAction(() -> binding.fab.setClickable(true))
    .start();
```

Best for single-view, single-shot animations; cancels previous animations on the same view automatically.

### 4. MotionLayout

MotionLayout lets you declare complex choreography in XML.

```xml
<!-- res/xml/scene_article.xml -->
<MotionScene xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:motion="http://schemas.android.com/apk/res-auto">

    <Transition
        motion:constraintSetStart="@+id/start"
        motion:constraintSetEnd="@+id/end"
        motion:duration="350">
        <OnSwipe
            motion:touchAnchorId="@+id/card"
            motion:dragDirection="dragUp" />
    </Transition>

    <ConstraintSet android:id="@+id/start">
        <Constraint android:id="@+id/card" ... />
    </ConstraintSet>
    <ConstraintSet android:id="@+id/end">
        <Constraint android:id="@+id/card" ... />
    </ConstraintSet>
</MotionScene>
```

```xml
<androidx.constraintlayout.motion.widget.MotionLayout
    android:id="@+id/motionRoot"
    app:layoutDescription="@xml/scene_article"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

Trigger from code:

```java
binding.motionRoot.transitionToEnd();
```

### 5. Transitions (`androidx.transition`)

Great for layout changes that are hard to script by hand.

```java
TransitionManager.beginDelayedTransition(binding.root, new AutoTransition());
binding.detail.setVisibility(View.VISIBLE);
```

Shared-element transitions across Fragments:

```java
FragmentNavigator.Extras extras = new FragmentNavigator.Extras.Builder()
        .addSharedElement(binding.image, "article-image")
        .build();
Navigation.findNavController(view).navigate(R.id.detailFragment, args, null, extras);
```

In the detail fragment, set `setSharedElementEnterTransition(TransitionInflater.from(ctx).inflateTransition(android.R.transition.move))`.

### 6. Performance

- Animate **transform properties** (`alpha`, `translationX`, `translationY`, `scale`, `rotation`) — they skip layout.
- Avoid animating `layout_width` / `layout_height`; use `scaleX`/`scaleY` instead when possible.
- Turn on **Profile GPU Rendering** (developer options) and keep frames under 16ms (8ms for 120Hz).
- Prefer `Hardware Acceleration` (default on API 21+). Call `setLayerType(View.LAYER_TYPE_HARDWARE, null)` only during animations with heavy compositing, then reset to `LAYER_TYPE_NONE` in `withEndAction`.

### 7. Cancellation

Always keep a reference to animations that may outlive their view:

```java
private Animator animator;

void show() {
    animator = ObjectAnimator.ofFloat(binding.fab, View.ALPHA, 0f, 1f);
    animator.start();
}

@Override
public void onDestroyView() {
    if (animator != null) animator.cancel();
    super.onDestroyView();
}
```

## Checklist

- [ ] Animations target transform properties; layout passes are avoided.
- [ ] Durations are 150–350ms for micro-interactions, 250–500ms for screen transitions.
- [ ] Interpolators are `FastOutSlowInInterpolator` (standard), `LinearOutSlowIn` (enter), `FastOutLinear` (exit).
- [ ] Animators created in a Fragment are cancelled in `onDestroyView`.
- [ ] Reduced-motion accessibility setting is respected (check `Settings.Global.ANIMATOR_DURATION_SCALE`).
- [ ] MotionLayout is preferred over hand-rolled multi-view choreography.
