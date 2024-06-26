
## **Debugging**



* [Local Logs](#local-logs)
    * [Request errors](#request-errors)
    * [Unexpected cache misses](#unexpected-cache-misses)
    * [RequestListener and custom logs](#requestlistener-and-custom-logs)
* [Missing images and local logs](#missing-images-and-local-logs)
    * [Failing to start the request.](#failing-to-start-the-request)
    * [Missing Size](#missing-size)
        * [Custom Targets](#custom-targets)
        * [Views](#views)
* [Out of memory errors](#out-of-memory-errors)
    * [Excessively large allocations.](#excessively-large-allocations)
    * [Memory leaks.](#memory-leaks)
* [Other common issues](#other-common-issues)
    * [“You can’t start or clear loads in RequestListener or Target callbacks”](#you-cant-start-or-clear-loads-in-requestlistener-or-target-callbacks)
    * [“cannot resolve symbol ‘GlideApp’”](#cannot-resolve-symbol-glideapp)

### **Local Logs**

If you have access to the device, you can look for a few log lines using `adb logcat` or your IDE. You can enable logging for any tag mentioned here using:



```
adb shell setprop log.tag.<tag_name> <VERBOSE|DEBUG>
```


 VERBOSE logs tend to be more verbose but contain more useful information. Depending on the tag, you can try both VERBOSE and DEBUG to see which provides the best level of information.


####  **Request errors**


The highest level and easiest to understand logs are logged with the `Glide` tag:


```
adb shell setprop log.tag.Glide DEBUG
```


The Glide tag will log both successful and failed requests and differing levels of detail depending on the log level. VERBOSE should be used to log successful requests. DEBUG can be used to log detailed error messages.


You can also control the verbosity of the Glide log tag programmatically using <code>[setLogLevel(int)](https://bumptech.github.io/glide/javadocs/400/com/bumptech/glide/GlideBuilder.html#setLogLevel-int-)</code>. <code>setLogLevel</code> allows you to enable more verbose logs in developer builds but not release builds, for example.


#### **Unexpected cache misses**


For details on how Glide’s caching works, see [the Caching page](https://nickyshe.github.io/Glide-V4/#/Caching).


The <code>[Engine](https://github.com/bumptech/glide/blob/6b137c2b1d4b2ab187ea2aa56834dea039daa090/library/src/main/java/com/bumptech/glide/load/engine/Engine.java#L33)</code> log tag provides details on how a request will be fulfilled and includes the full in memory cache key used to store the corresponding resource. If you’re trying to debug why images you have in memory in one place aren’t being used in another place, the <code>Engine</code> tag lets you compare the cache keys directly to see the differences.


For each started request, the `Engine` tag will log that the request will be completed from cache, active resources, an existing load, or a new load. Cache means that the resource wasn’t in use, but was available in the in memory cache. Active resources means that the resource was actively being used by another `Target`, typically in a `View`. An existing load means that the resource wasn’t available in memory, but another `Target` had previously requested the same resource and the load is already in progress. Finally a new load means that the resource was neither in memory nor already being loaded so our request triggered a new load.


#### **RequestListener and custom logs**

If you’d like to programmatically keep track of errors and successful loads, track the overall cache hit ratio of images in your application, or have more control over local logs, you can use the <code>[RequestListener](https://bumptech.github.io/glide/javadocs/400/com/bumptech/glide/request/RequestListener.html)</code> interface. <code>RequestListener</code> can be added to an individual load using <code>[RequestBuilder#listener()](https://bumptech.github.io/glide/javadocs/400/com/bumptech/glide/RequestBuilder.html#listener-com.bumptech.glide.request.RequestListener-)</code>. Sample usage looks like this:


```
Glide.with(fragment)
   .load(url)
   .listener(new RequestListener() {
       @Override
       boolean onLoadFailed(@Nullable GlideException e, Object model,
           Target<R> target, boolean isFirstResource) {
         // Log the GlideException here (locally or with a remote logging framework):
         Log.e(TAG, "Load failed", e);

         // You can also log the individual causes:
         for (Throwable t : e.getRootCauses()) {
           Log.e(TAG, "Caused by", t);
         }
         // Or, to log all root causes locally, you can use the built in helper method:
         e.logRootCauses(TAG);

         return false; // Allow calling onLoadFailed on the Target.
       }

       @Override
       boolean onResourceReady(R resource, Object model, Target<R> target,
           DataSource dataSource, boolean isFirstResource) {
         // Log successes here or use DataSource to keep track of cache hits and misses.

         return false; // Allow calling onResourceReady on the Target.
       }
    })
    .into(imageView);
```


Note that each GlideException has multiple `Throwable` root causes. Each load in Glide may have an arbitrary number of ways the registered components (`ModelLoader`, `ResourceDecoder`, `Encoder` etc) can be used to load the a given Resource (`Bitmap`, `GifDrawable` etc) from a given Model (URL, File etc). Each `Throwable` root cause describes the reason why a particular combination of Glide’s components failed. Understanding why a particular request failed may require inspecting all root causes.


However, you may also be able to find a single root cause that’s more relevant than the others. For example if you’re loading URLs and you’re trying to find the specific HttpException that would indicate that your load failed due to a network error, you can iterate over all of the root causes and use `instanceof` to inspect the type:


```
for (Throwable t : e.getRootCauses()) {
  if (t instanceof HttpException) {
    Log.e(TAG, "Request failed due to HttpException!", t);
    break;
  }
}
```


You can use a similar process involving iteration and `instanceof` to try to detect other types of exceptions if you’re interested in more than HTTP errors.


To save object allocations, you can re-use the same `RequestListener` for multiple loads.


### **Missing images and local logs**


In some cases you may see that an image never loads and that no logs with either the `Glide` tag or the `Engine` tag are ever logged for your request. There are a few possible causes.


#### **Failing to start the request.**

 Verify that you’re calling <code>[into()](https://bumptech.github.io/glide/javadocs/400/com/bumptech/glide/RequestBuilder.html#into-android.widget.ImageView-)</code> or <code>[submit()](https://bumptech.github.io/glide/javadocs/400/com/bumptech/glide/RequestBuilder.html#submit-int-int-)</code> for your request. If you don’t call either method, you’re never asking Glide to start your load.


#### **Missing Size**

If you verify that you are in fact calling <code>[into()](https://bumptech.github.io/glide/javadocs/400/com/bumptech/glide/RequestBuilder.html#into-android.widget.ImageView-)</code> or <code>[submit()](https://bumptech.github.io/glide/javadocs/400/com/bumptech/glide/RequestBuilder.html#submit-int-int-)</code> and you’re still not seeing logs, the most likely explanation is that Glide is unable to determine the size of the <code>View</code> or <code>Target</code> you’re attempting to load your resource into.


#####  **Custom Targets**

If you’re using a custom `Target`, make sure you’ve either implemented <code>[getSize](https://bumptech.github.io/glide/javadocs/400/com/bumptech/glide/request/target/Target.html#getSize-com.bumptech.glide.request.target.SizeReadyCallback-)</code> and are calling the given callback with a non-zero width and height or are subclassing a <code>Target</code> like <code>[ViewTarget](https://bumptech.github.io/glide/javadocs/400/com/bumptech/glide/request/target/ViewTarget.html)</code> that implements the method for you.


#####  **Views**

If you’re just loading a resource into a `View`, the most likely explanation is that your view is either not going through layout or is being given a 0 width or height. Views may not go through layout if their visibility is set to `View.GONE` or if they are never attached. Views may receive invalid or 0 widths and heights if they and/or their parents have certain combinations of `wrap_content` and `match_parent` for their widths and heights. You can experiment by giving your views fixed non-zero dimensions or passing in an specific size to Glide to use for the request with the <code>[override(int, int) API](https://bumptech.github.io/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html#override-int-int-)</code>.


###  **Out of memory errors**

Almost all OOM errors are due to issues with the hosting application and not with Glide.


There are two common causes of OOMs in applications:



1. Excessively large allocations
2. Memory leaks (memory that is allocated but never released)

#### **Excessively large allocations.**

If opening a single page or loading a single image causes an OOM, your applications is probably loading an unnecessarily large image.


The amount of memory required to display an image in a Bitmap is width * height * bytes per pixel. The number of bytes per pixel depends on the `Bitmap.Config` used to display the image, but typically four bytes per pixel are required for `ARGB_8888` Bitmaps. As a result, even a 1080p image requires 8mb of ram. The larger the image, the more ram required, so a 12 megapixel image requires a fairly massive 48mb.


Glide will downsample images automatically based on the size provided by the `Target`, `ImageView` or `override()` request option provided. If you’re seeing excessively large allocations in Glide, usually that means that the size of your `Target` or `override()` is too large or you’re using `Target.SIZE_ORIGINAL` in conjunction with a large image.


To fix excessively large allocations, avoid `Target.SIZE_ORIGINAL` and ensure that the size of your `ImageViews` or that you provide to Glide via `override()` are reasonable.


#### **Memory leaks.**

 If repeating the same set of steps in your application over and over again gradually increases your applications’ memory usage and eventually leads to an OOM, you probably have a memory leak.


 The [Android documentation](https://developer.android.com/studio/profile/investigate-ram.html) has a lot of good information on tracking and debugging memory usage. To investigate memory leaks, you’re almost certainly going to want to [capture a heap dump](https://developer.android.com/studio/profile/investigate-ram.html#HeapDump) and look for Fragments, Activities or other objects that are retained after they’re no longer used.


 To fix memory leaks, remove references to the destroyed `Fragment` or `Activity` at the appropriate point in the lifecycle to avoid retaining excessive objects. Use the heap dump to help find other ways your application retains memory and remove unnecessary references as you find them. It’s often helpful to start by listing the shortest paths excluding weak references to all Bitmap objects (using [MAT](https://www.eclipse.org/mat/) or another memory analyzer) and then looking for reference chains that seem suspicious. You can also check to make sure that you have no more than once instance of each `Activity` and only the expected number of instances of each `Fragment` by searching for them in your memory analyzer.


###  **Other common issues**


####  **“You can’t start or clear loads in RequestListener or Target callbacks”**


If you attempt to start a new load in `onResourceReady` or `onLoadFailed` in a `Target` or `RequestListener`, Glide will throw an exception. We throw this exception because it’s challenging for us to handle the load being recycled while it’s in the middle of notifying.


Fortunately this is easy to fix. Starting in Glide 4.3.0, you can simply use the <code>[.error()](https://bumptech.github.io/glide/javadocs/430/com/bumptech/glide/RequestBuilder.html#error-com.bumptech.glide.RequestBuilder-)</code> method. <code>[error()](https://bumptech.github.io/glide/javadocs/430/com/bumptech/glide/RequestBuilder.html#error-com.bumptech.glide.RequestBuilder-)</code> takes an arbitrary <code>[RequestBuilder](https://bumptech.github.io/glide/javadocs/430/com/bumptech/glide/RequestBuilder.html)</code> that will start a new request only if the primary request fails:



```
Glide.with(fragment)
  .load(url)
  .error(Glide.with(fragment)
     .load(fallbackUrl))
  .into(imageView);
```



Prior to Glide 4.3.0, you can also use an Android <code>[Handler](https://developer.android.com/reference/android/os/Handler.html)</code> to post a Runnable with your request:


```
private final Handler handler = new Handler();
...

Glide.with(fragment)
  .load(url)
  .listener(new RequestListener<Drawable>() {
      ...

      @Override
      public boolean onLoadFailed(@Nullable GlideException e, Object model, 
          Target<Drawable> target, boolean isFirstResource) {
        handler.post(new Runnable() {
            @Override
            public void run() {
              Glide.with(fragment)
                .load(fallbackUrl)
                .into(imageView);
            }
        });
      }
  )
  .into(imageView);
```



#### **“cannot resolve symbol ‘GlideApp’”**


When using the generated API, you may run into errors that prevent the annotation processor from generating Glide’s API. Sometimes these errors are related to [your setup](https://nickyshe.github.io/Glide-V4/#/Download_Setup), but other times they can be completely unrelated.


Often unrelated failures are hidden by the number of non root cause error messages. There may be so many other errors that you’ll be unable to find the root cause in your build logs. If this happens and you’re using Gradle, try adding the following to increase the number of error messages Gradle will print:


```
allprojects {
  gradle.projectsEvaluated {
    tasks.withType(JavaCompile) {
        options.compilerArgs << "-Xmaxerrs" << "1000"
    }
  }
}
```



See also:



* [https://github.com/bumptech/glide/issues/1945](https://github.com/bumptech/glide/issues/1945)
* [https://stackoverflow.com/questions/3115537/java-compilation-errors-limited-to-100/35707023#35707023](https://stackoverflow.com/questions/3115537/java-compilation-errors-limited-to-100/35707023#35707023)