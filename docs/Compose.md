##  **Compose**


## **About**


[Jetpack Compose](https://developer.android.com/jetpack/compose) is Android’s modern toolkit for building native UI. This library integrates with Compose to allow you to load images in your Compose apps with Glide in a performant manner.


## **Status**

Glide’s Compose integration is in beta. Please submit bugs if you encounter any and/or feature requests to [Glide’s Github Issues](https://github.com/bumptech/glide/issues/new). The API is close to final. We’ll remove the ability to use subcompositions as placeholders. We may replace `GlideSubcomposition` with a custom `Modifier`. If we do so, we will deprecate `GlideSubcomposition` first so that you have time to migrate.


## **How do I include the Compose integration library?**

The Compose integration doesn’t include any components, so you don’t need any changes to your `AppGlideModule` to use it.

Instead, just add a Gradle dependency on the Compose integration library:


```
implementation "com.github.bumptech.glide:compose:1.0.0-beta01"
```



##  **Usage**

 See Glide’s [Gallery sample app](https://github.com/bumptech/glide/tree/master/samples/gallery) for a small application that uses Compose and Glide’s Compose integration. See the [Dokka](https://bumptech.github.io/glide/javadocs/4140/integration/compose/com.bumptech.glide.integration.compose/index.html) page for detailed API documentation.


###  **GlideImage**

The primary integration point between Compose and Glide is `GlideImage`. `GlideImage` is meant to similarly to [Compose’s Image](https://developer.android.com/reference/kotlin/androidx/compose/foundation/package-summary#Image(androidx.compose.ui.graphics.painter.Painter,kotlin.String,androidx.compose.ui.Modifier,androidx.compose.ui.Alignment,androidx.compose.ui.layout.ContentScale,kotlin.Float,androidx.compose.ui.graphics.ColorFilter)) function except that it uses Glide to asynchronously load images.

Simple use cases of GlideImage can include just a Model and a content description:


```
GlideImage(model = myUrl, contentDescription = getString(R.id.picture_of_cat))
```


You can supply a custom <code>[Modifier](https://developer.android.com/jetpack/compose/modifiers)</code> to customize how <code>GlideImage</code> is rendered:


```
GlideImage(
  model = myUrl,
  contentDescription = getString(R.id.picture_of_cat),
  modifier = Modifier.padding(padding).clickable(onClick = onClick).fillParentMaxSize(),
)
```

You can also provide the `alignment`, `contentScale`, `colorFilter`, and `alpha` parameters that have identical defaults and function identically to the same parameters in [Compose’s Image](https://developer.android.com/reference/kotlin/androidx/compose/foundation/package-summary#Image(androidx.compose.ui.graphics.painter.Painter,kotlin.String,androidx.compose.ui.Modifier,androidx.compose.ui.Alignment,androidx.compose.ui.layout.ContentScale,kotlin.Float,androidx.compose.ui.graphics.ColorFilter)).

To configure the Glide load, you can provide a `RequestBuilderTransformation` function. The function will be passed a `RequestBuilder` that already has `load()` called on it with your given model. You can then customize the request with any normal [Glide option](https://nickyshe.github.io/Glide-V4/#/Options ) except for Transitions (see below for details).


```
GlideImage(
  model = myUrl,
  contentDescription = getString(R.id.picture_of_cat),
  modifier = Modifier.padding(padding).clickable(onClick = onClick).fillParentMaxSize(),
) {
   it
    .thumbnail(
      requestManager
      .asDrawable()
      .load(item.uri)
      .signature(signature)
      .override(THUMBNAIL_DIMENSION)
    )
    .signature(signature)
}
```



####  **Placeholders**

Glide supports three types of placeholders for Compose:



1. Drawables
2. Android resource IDs
3. Painters

    To specify a placeholder, use the `loading` and `failure` parameters in `GlideImage`. Placeholders are required to be one of Glide’s `Placeholder` classes:



```
import com.bumptech.glide.integration.compose.GlideImage
import com.bumptech.glide.integration.compose.placeholder

// Drawable
GlideImage(model = myUrl, loading = placeholder(myDrawable))

import com.bumptech.glide.integration.compose.GlideImage
import com.bumptech.glide.integration.compose.placeholder

// Resource ID
GlideImage(model = myUrl, failure = placeholder(R.drawable.my_drawable))

import com.bumptech.glide.integration.compose.GlideImage
import com.bumptech.glide.integration.compose.placeholder

// Painter
GlideImage(model = myUrl, loading = placeholder(ColorPainter(Color.Red)))
```

If you set a placeholder on `GlideImage` directly and on `RequestBuilder` via `requestBuilderTransform`, the placeholder set on `GlideImage` will take precedence.


####  **Sizing**

As with Glide’s View integration, Glide’s Compose integration will attempt to determine the size of your Composable and use that to load an appropriately sized image. That can only be done efficiently if you provide a `Modifier` that restricts the size of the Composable. If Glide determines that either the width or the height of the Composable is unbounded, it will use `Target.SIZE_ORIGINAL`, which can lead to excessive memory usage.


Whenever possible, make sure you either set a `Modifier` with a bounded size or provide an <code>[override()](https://bumptech.github.io/glide/javadocs/430/com/bumptech/glide/request/RequestOptions.html#override-int-int-)</code> size to your Glide request. In addition to saving memory, loads from the disk cache will also be faster if your size is smaller.


####  **Transitions**

 `GlideImage` as of version `alpha5` supports a Compose specific `Transition` API. Transitions must be specified in `GlideImage` using the Compose API. Transitions specified in `RequestBuilder` will be ignored.


To cross fade from the placeholder you’ve set (if any) to the image when the load completes, use the built in `CrossFade` transition:


```
import com.bumptech.glide.integration.compose.CrossFade

GlideImage(uri, contentDescription, transition = CrossFade)
```

You can also write your own custom transition. If you do, please let me know by [filing an issue](https://github.com/bumptech/glide/issues/new). There’s more we can do to make this easier.

For now your best bet is to take a look at the existing `CrossFade` and follow a similar pattern: https://github.com/bumptech/glide/blob/2ae4effcf7bad2131a54423d203c2d124257fc04/integration/compose/src/main/java/com/bumptech/glide/integration/compose/Transition.kt#L99


####  **Recomposition and state changes**

In some rare cases, you might want to change your composition based on the state of a Glide load. You virtually never want to do this in any kind of scrolling list because doing so will incur several recompositions per image load, causing a significant amount of jank. As a rule if you can avoid using this API, you should.

That said, if you do need to recompose on state changes, you can use `GlideSubcomposition`:


```
import com.bumptech.glide.integration.compose.GlideSubcomposition

GlideSubcomposition(item.uri, modifier, requestBuilderTransform = {
  it.thumbnail(preloadRequestBuilder)
}) {
  // state comes from GlideSubcompositionScope
  when (state) {
    RequestState.Failure -> TODO()
    RequestState.Loading -> TODO()
    // painter also comes from GlideSubcompositionScope
    is RequestState.Success -> Image(painter, contentDescription = null)
  }
}
```



 If you have a use case that requires this API, please let me know by [filing an issue](https://github.com/bumptech/glide/issues/new). Another likely option here is to expose a `Modifier` based API where you could compose a custom `Modifier` with Glide’s and monitor Glide’s state changes via that custom `Modifier`. This is not yet implemented.


#### **GlideLazyListPreloader**


GlideLazyListPreloader uses [Compose’s LazyListState](https://developer.android.com/reference/kotlin/androidx/compose/foundation/lazy/LazyListState) to determine the direction the user is scrolling and preload images in the direction of scroll. Preloading, especially when used with relatively small image sizes in combination with [Glide’s thumbnail API](https://bumptech.github.io/glide/doc/options.html#thumbnail-requests), can dramatically improve the UX of horiztonally or vertically scrolling UIs.


 Using the preloader looks like this:


```
@Composable
fun DeviceMedia(mediaStoreData: List<MediaStoreData>) {
  val state = rememberLazyListState()
  LazyRow(state = state) {
    items(mediaStoreData) { mediaStoreItem ->
      // Uses GlideImage to display a MediaStoreData object
      MediaStoreView(mediaStoreItem, requestManager, Modifier.fillParentMaxSize())
    }
  }

   GlideLazyListPreloader(
    state = state,
    data = mediaStoreData,
    size = THUMBNAIL_SIZE,
    numberOfItemsToPreload = 15,
    fixedVisibleItemCount = 2,
  ) { item, requestBuilder ->
    requestBuilder.load(item.uri).signature(item.signature())
  }
}

@Composable
fun MediaStoreView(item: MediaStoreData, requestManager: RequestManager, modifier: Modifier) {
  val signature = item.signature()

  GlideImage(
    model = item.uri,
    contentDescription = item.displayName,
    modifier = modifier,
  ) {
    it
      // This thumbnail request exactly matches the request in GlideLazyListPreloader
      // so that the preloaded image can be used here and display more quickly than 
      // the primary request.
      .thumbnail(
        requestManager
          .asDrawable()
          .load(item.uri)
          .signature(signature)
          .override(THUMBNAIL_DIMENSION)
      )
      .signature(signature)
  }
}
```

 `state` is a <code>[LazyListState](https://developer.android.com/reference/kotlin/androidx/compose/foundation/lazy/LazyListState)</code> that you can obtain using standard Compose APIs. <code>data</code> is the List of model objects that you’re displaying. <code>size</code> is the size of the image that you want to preload. This must exactly match at least one size provided to a Request or thumbnail Request used by <code>GlideImage</code> to display the item. <code>numberOfItemsToPreload</code> is total number of items you want to try to keep in memory ahead of the user’s position as they scroll. You may need some performance testing to find the right balance. Too high of a number and you may exceed the memory cache size. Too low and you won’t be able to keep up with scrolling. <code>fixedVisibleItemCount</code> is a guess of how many items you think will typically be visible on the screen at once.

Finally you can provide a `PreloadRequestBuilderTransform`, which will give you one object from the `data` list at a time to create a Glide request for. `size` will be applied for you automatically via Glide’s override API. So you only need to specify any additional components of the load that are required to make the request exactly match the load (or at least one thumbnail load) in the corresponding `GlideImage`.

 As with much of Glide’s Compose API, there are likely to be some API changes to this class. In particular we’d like to simplify the process of specifying the request so that the request options are not duplicated, once in `GlideLazyListPreloader` and again in `GlideImage`.
