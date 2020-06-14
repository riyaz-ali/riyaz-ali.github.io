---
title: Google Image Proxy
date: 2018-06-06
author: riyaz-ali
background: https://source.unsplash.com/yeB9jDmHm6M/1600x900
description: |
    This is an how-to that explains how you can use
    Google's public proxy to optimise and load images
    in an Android app.
tags:
- Android
- HowTo
keywords:
- Android
- Image Loading
- Google Proxy
---

Continuing with [app that I created](https://github.com/riyaz-ali/tringo) as a project for the [Udacity Android Nanodegree program](https://in.udacity.com/course/android-developer-nanodegree-by-google--nd801), I faced a new challenge! The challenge was to load _poster images_ from
TMDb Image server inside my app!

Is it even a challenge? One could say that I could've simply used a library like [Picasso](http://square.github.io/picasso/) or [Glide](https://bumptech.github.io/glide/) to offload image loading to the _tried and trusted tools_. But my main challenge **was not just to load the image but to load the _perfect image_**. 

And that means the image _should_ fit-well into `ImageView` without much need to resize and/or scale it, and thus, giving a _crisp and sharp_ look and improving
UX.

So, what's the big deal? Why don't I just use the [different bucket sizes](https://api.themoviedb.org/3/configuration) (`w185`, `w300`, etc.) that TMDb offers?

The thing is that the aspect size of the images served by TMDb is a bit weird (IMO!). For example, for the `w185` bucket, the image served has an aspect ratio of `185:278`!

What can be done? Well, the only thing we can do is to download a bigger image (from `original` bucket) and resize it  on device. But that's a total no-go because the `original` image is too large (`2000x3000`) and would consume _signifacntly more_ resources. And also, performing _resize_ on user's device is not a good idea!

What I initally thought was to setup a [Thumbor](http://thumbor.org/) server. But I quickly dropped the idea because then I would have to maintain the server (which I didn't want) for the (atleast) lifetime of the project or else the app would simply not work!

This led me to search for _open, free and hosted_ alternatives. Although, I couldn't find a _trustworthy_ solution, what I found was [this Gist](https://gist.github.com/coolaj86/2b2c14b1745028f49207). In the Gist, the author _reverse engineered_ (sort of) the URL for the Google Image proxy and the required parameters and leveraged it as a _caching and resizing proxy_ to serve his images!

Cool! how to use that in the project app? Simple! create the primary TMDb image url, append it as a parameter to the Google Image Proxy url and load the resulting url into the `ImageView` using your favorite library!

So, here's a [Utility class](https://www.quora.com/What-is-a-utility-class) that I created to handle the stuff!

```java
public final class ImageUtils {
  private ImageUtils(){
    /*  ¯\_(ツ)_/¯ */
  }

 // Google's image caching and resizing proxy
 private static final String GOOGLE_CACHE_BASE =
  "https://images1-focus-opensocial.googleusercontent.com/gadgets/proxy";


  /**
   * Create base tmdb url
   */
  @NonNull private static String tmdbUrl(@NonNull String poster){
    return String.format(
      "%s/original%s", 
      // or however you get the base url...
      BuildConfig.TMDB_IMAGE_BASE, 
      poster);
  }

  /**
   * Calculate aspect
   */
  public static int aspect(int width, int height) {
    return height / width;
  }

  /**
   * Calculate the other dimension (either height or width) 
   * given one dimension and an aspect
   *
   * Use as: calculateOtherDimension(aspect(4, 3), 400) -> 300
   *
   * @param aspect aspect ratio
   * @param dimension measurement of one dimension
   */
  public static int calculateOtherDimension(int aspect, int dimension){
    return aspect * dimension;
  }

  @NonNull public static String createTmdbImageUrl(
    @NonNull String poster, 
    int height, 
    int width){
      return Uri.parse(GOOGLE_CACHE_BASE).buildUpon()
        .appendQueryParameter("container", "focus")
        .appendQueryParameter("resize_h", "" + height)
        .appendQueryParameter("resize_w", "" + width)
        .appendQueryParameter("url", tmdbUrl(poster))
        .toString();
  }

}
```

Now to use it, just call it as

```java
import static ImageUtils.*;

// ...
createTmdbImageUrl(
  movie.getPoster(), 
  300 /* height */, 
  calculateOtherDimension(aspect(2 /*w*/, 3 /*h*/), 300)
);
//...
```

{{< image src="https://images1-focus-opensocial.googleusercontent.com/gadgets/proxy?container=focus&resize_w=200&resize_h=300&url=http%3A%2F%2Fimage.tmdb.org%2Ft%2Fp%2Foriginal%2Fto0spRl1CMDvyUbOnbb4fTk3VAd.jpg" alt="Deadpool" position="center" >}}

And that's it! Using this you can load image of any aspect that you need and deliver a _crispier and sharper experience_ to your users. But, I'd sincerely like to request to you to _only use this_ for small, open source projects so that you don't need to maintain a server!<br />
If you are developing a closed-source business application, than it would be much better to use something like [Thumbor](http://thumbor.org/) or [ImageProxy](https://github.com/willnorris/imageproxy) as they are dedicated solutions and provide more features!