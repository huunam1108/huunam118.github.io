---
layout: post
title: Using Vector Drawable Safely on pre-lollipop version
image: /img/vector_drawable.png
---

Want to see more about docs and how to use vector drawable, please come and read [here](https://developer.android.com/guide/topics/graphics/vector-drawable-resources)

I want to show what we need to handle all cases for using Vector Drawable via XML resources

#### build.gradle
```DSL
 android {
   defaultConfig {
     vectorDrawables.useSupportLibrary = true
    }
 }
```
#### Application class

```Java
override fun onCreate() {
    super.onCreate()
    // App crashes when selector with using vector-drawable
    // So, keep using this line + AppCompatImageView for android 4.4
    AppCompatDelegate.setCompatVectorFromResourcesEnabled(true)
    // Your other logic code ...
}
```

#### Way to get Drawable

```Java
// using ContextCompat
ContextCompat.getDrawable(@NonNull Context context, @DrawableRes int resId)
// using ResourcesCompat
ResourcesCompat.getDrawable(@NonNull Resources res, @DrawableRes int id, @Nullable Theme theme)
// using AppCompatResources
AppCompatResources.getDrawable(@NonNull Context context, @DrawableRes int resId)
// using VectorDrawableCompat
VectorDrawableCompat.create(@NonNull Resources res, @DrawableRes int resId, @Nullable Theme theme)
```

Here is the comparision table when using one of these above ways. This is [StackOverflow link](https://stackoverflow.com/a/48237058/3682565)

For normal image resource:

|Kind                  | Set `setCompatVectorFromResourcesEnabled` | Not Set |
|----------------------|-------------------------------------------|---------|
|ContextCompat         | Good                                      |  Good   |
|----------------------|-------------------------------------------|---------|
|ResourcesCompat       | Good                                      |  Good   |
|----------------------|-------------------------------------------|---------|
|AppCompatResources    | Good                                      |  Good   |
|----------------------|-------------------------------------------|---------|
|VectorDrawableCompat  | Crashed                                   | Crashed |
|----------------------|-------------------------------------------|---------|

For vector drawable image resource:

|Kind                  | Set `setCompatVectorFromResourcesEnabled` | Not Set |
|----------------------|-------------------------------------------|---------|
|ContextCompat         | Good                      |  Crashed if API < 21    |
|----------------------|-------------------------------------------|---------|
|ResourcesCompat       | Good                      |  Crashed if API < 21    |
|----------------------|-------------------------------------------|---------|
|AppCompatResources    | Good                                      |  Good   |
|----------------------|-------------------------------------------|---------|
|VectorDrawableCompat  | Good                                      |  Good   |
|----------------------|-------------------------------------------|---------|

#### For custom view

Here is the sample for custom view with drawable using vector
```Java
const val INVALID_RES_ID = -1
private fun initAttrs(attrs: AttributeSet?) {
    if (isInEditMode || attrs == null) return
    var drawableStart: Drawable? = null
    var ta: TypedArray? = null
    try {
        ta = context.obtainStyledAttributes(attrs, R.styleable.YourCustomView)
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            // Directly call getDrawable is safely
            drawableStart = typedArray.getDrawable(...)
        } else {
            val drawableStartId = typedArray.getResourceId(..., INVALID_RES_ID)
            if (drawableStartId != INVALID_RES_ID)
                drawableStart = AppCompatResources.getDrawable(context, drawableStartId)
        }
        // other logic
    } finally {
        ta?.recycle()
    }
}
```
For using `selector` or `DrawableStateList` in custom view, you must using `AppCompat...` kind like `AppCompatImageView`, `AppCompatImageButton`...
if not your app will be crashed on pre-lollipop android version

Example:
Your selector
```XML
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:drawable="@drawable/your_vector_drawable" android:state_pressed="true" />
    <item android:drawable="@drawable/your_vector_drawable" android:state_enabled="true" />
    <item android:drawable="@drawable/your_vector_drawable" android:state_enabled="false" />
    <item android:drawable="@drawable/your_vector_drawable" />
</selector>
```
Your layout
```XML
<android.support.v7.widget.AppCompatImageView
    android:id="@+id/img_id"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:contentDescription="@null"
    />
```
Your code
```Java
val resId = typedArray.getResourceId(stylable, -1)
img_id.setImageResource(resId)
```