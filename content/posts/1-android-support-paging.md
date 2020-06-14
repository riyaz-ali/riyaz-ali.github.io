---
title: Android Support Paging Library
date: 2018-06-01
author: riyaz-ali
background: https://source.unsplash.com/w33-zg-dNL4
description: |
    In this how-to, I've tried to explain how you can use
    Android Support Paging Library to improve your app's 
    performance while loading dynamic data.
tags:
- Android
- AndroidSupportLibrary
- HowTo
keywords:
- Android
- Android Support Library
- Android Pagination
---

Recently, I was working on a project for the [Udacity Android Nanodegree program](https://in.udacity.com/course/android-developer-nanodegree-by-google--nd801) to develop an Android application for [**TMDb**](https://www.themoviedb.org/) to list movies using the [TMDb API](https://developers.themoviedb.org/3/getting-started/introduction).

As you might've guessed, TMDb has a _huge collection of movies_, and so a need for _paging the data_ was quite obvious from the very begining of the project.

So I started looking into ways in which I could've implemented paging with my `RecyclerView`. Turns out there are a few ways to achieve this!

1. _The old way_ (as documented in [<u>this CodePath guide</u>](https://github.com/codepath/android_guides/wiki/Endless-Scrolling-with-AdapterViews-and-RecyclerView)) is to use [`RecyclerView.OnScrollListener`](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.OnScrollListener) and _listen for_ scroll events and load data accordingly! But with the release of the [Android Paging Library](https://developer.android.com/topic/libraries/architecture/paging), the guide is now considered _deprecated_!

2. _The new way_ (yeah! you guessed it) is to use [Android Paging Library](https://developer.android.com/topic/libraries/architecture/paging) and the rest of this post is all about how to integrate it!

### Architecture

So you ask, how this Paging Library works?

Well, there are four major components in the Paging Library:

1. `DataSource`: The `DataSource` is where you write code to interact with your source (REST API, sqlite database, etc.) All the requests for data from the `DataSource` are made on a background thread and so you can safely write _synchronous_ code to interact with your source, if you want to. `DataSource` is responsible to load a `List<T>` of your model object given a page/key and implement business logic for the same.

2. `PagedListAdapter<T, VH>`: `PagedListAdapter` is a `RecyclerView.Adapter` which uses a `PagedList` (see below) to load data into the UI. `PagedListAdapter` uses a `DiffUtils.ItemCallback<T>` implementation to calculate _difference_ between your model objects in the list and animates add/remove operations on UI.

3. `DiffUtils.ItemCallback<T>`: The `DiffUtils.ItemCallback<T>` for your model class is utilized by the `PagedListAdapter` to calculate _diff_ between two list and animate changes in the UI. You must provide an implementation of this callback. The callback is executed on a background thread to prevent stalling the UI.

4. `PagedList<T>`: A `PagedList` is a Java-collection which loads it's data in chunks (pages) from a `DataSource`. The `PagedList` connects your `PagedListAdapter` and your `DataSource` together and is responsible to _lazy load_ data by calling appropriate methods on the `DataSource`.

{{< image src="https://cdn-images-1.medium.com/max/800/1*O1acN3yAOS70zbRMfThXrw.gif" alt="Paging Library Architecture" position="center" >}}

First, we create a _model POJO_ class (referred in the post as `Movie`) and implement [`DiffUtil.ItemCallback<T>`](https://developer.android.com/reference/android/support/v7/util/DiffUtil.ItemCallback)

```java
public class Movie {
  // id of the movie
  private long id;
  // title of the movie
  private String title;
  // path to movie's poster
  private String poster;

  // DiffCallback to assist Adapter
  public static final DiffUtil.ItemCallback<Movie> DIFF_CALLBACK = 
    new DiffUtil.ItemCallback<Movie>() {
      @Override 
      public boolean areItemsTheSame(Movie oldItem, Movie newItem) {
        return oldItem.id == newItem.id;
      }

      @Override
      public boolean areContentsTheSame(Movie oldItem, Movie newItem) {
        return TextUtils.equals(oldItem.poster, newItem.poster)
            && TextUtils.equals(oldItem.title, newItem.title);
      }
    };
}
```

Then, we subclass [PagedListAdapter](https://developer.android.com/reference/android/arch/paging/PagedListAdapter) and create our custom [adapter](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.Adapter) which will be used to display `Movie`

```java
// required imports
import android.arch.paging.PagedListAdapter;

public class MovieAdapter 
  extends PagedListAdapter<Movie, MovieAdapter.ViewHolder> {
  // ViewHolder removed for brevity... nothing special in the ViewHolder

  /**
   * Create new adapter and pass the diff callback to parent
   */
  public MovieAdapter(
      @NonNull DiffUtil.ItemCallback<Movie> diffCallback) {
    super(diffCallback);
  }

  // ... then the usual create(...) and bind(...) view holder methods
}
```

Next we need to create a [`DataSource`](https://developer.android.com/reference/android/arch/paging/DataSource) that will _connect_ with the API and fetch data! There are 3 types of `DataSource` provided by the Paging Library:
* [`ItemKeyedDataSource<Key, Value>`](https://developer.android.com/reference/android/arch/paging/ItemKeyedDataSource): Use it if you need to use data from item `N-1` to load item `N`

* [`PageKeyedDataSource<Key, Value>`](https://developer.android.com/reference/android/arch/paging/PageKeyedDataSource): Use it if pages you load embed keys for loading adjacent pages.

* [`PositionalDataSource<T>`](https://developer.android.com/reference/android/arch/paging/PositionalDataSource): Use it if you can load pages of a requested size at arbitrary positions, and provide a fixed item count.

You can read more about the types of `DataSource` and when to use which one on [`DataSource` documentation](https://developer.android.com/reference/android/arch/paging/DataSource)

For our example, we will use [`PageKeyedDataSource<Key, Value>`](https://developer.android.com/reference/android/arch/paging/PageKeyedDataSource)

```java
// required imports
import android.arch.paging.PageKeyedDataSource;

public final class MoviesDataSource 
  extends PageKeyedDataSource<Integer, Movie> {
    // constant max retry count, see Bonus section for more!
    private static final int MAX_RETRY = 3;
    // retrofit service instance
    private final Tmdb.Api tmdbApi;

    public MoviesDataSource(@NonNull Tmdb.Api tmdbApi) {
      this.tmdbApi = tmdbApi;
    }
    
    @Override public void loadInitial(@NonNull LoadInitialParams<Integer> params,
      @NonNull LoadInitialCallback<Integer, Movie> callback) {

    // load the first page
    tmdbApi.getPopularMovies(1 /* page */).enqueue(new CallbackWithRetry<MovieResponse>(MAX_RETRY) {
      @Override public void onResponse(Call<MovieResponse> call, Response<MovieResponse> response) {
        if (response.isSuccessful()) {
          MovieResponse movieResponse = response.body();

          // send response back
          callback.onResult(movieResponse.getMovies(), null, // null because there is no page before this
              movieResponse.getPage() + 1); // TMDb follows incrementing integer as pages numbers
        } else {
          onFailure(call, new Exception("unknown error"));
        }
      }

      @Override public void onFinalFailure(Call<MovieResponse> call, Throwable t) {
        callback.onResult(Collections.emptyList(), null, null);
      }
    });
  }

  @Override public void loadBefore(@NonNull LoadParams<Integer> params,
      @NonNull LoadCallback<Integer, Movie> callback) {

    // params.key contains the page number of the page to load
    tmdbApi.getPopularMovies(params.key).enqueue(new CallbackWithRetry<MovieResponse>(MAX_RETRY) {
      @Override public void onResponse(Call<MovieResponse> call, Response<MovieResponse> response) {
        if (response.isSuccessful()) {
          MovieResponse movieResponse = response.body();
          // return response, since we are moving backward adjacent page is - 1 from current position
          callback.onResult(movieResponse.getMovies(), movieResponse.getPage() - 1);
        } else {
          onFailure(call, new Exception("unknown error"));
        }
      }

      @Override public void onFinalFailure(Call<MovieResponse> call, Throwable t) {
        callback.onResult(Collections.emptyList(), null);
      }
    });
  }

  @Override public void loadAfter(@NonNull LoadParams<Integer> params,
      @NonNull LoadCallback<Integer, Movie> callback) {
    
    // params.key contains the page number of the page to load
    tmdbApi.getPopularMovies(params.key).enqueue(new CallbackWithRetry<MovieResponse>(MAX_RETRY) {
      @Override public void onResponse(Call<MovieResponse> call, Response<MovieResponse> response) {
        if(response.isSuccessful()){
          MovieResponse movieResponse = response.body();
          // return response, since we are moving forward adjacent page is + 1 from current position
          callback.onResult(movieResponse.getMovies(), movieResponse.getPage() + 1);
        } else {
          onFailure(call, new Exception("unknown error"));
        }
      }

      @Override public void onFinalFailure(Call<MovieResponse> call, Throwable t) {
        callback.onResult(Collections.emptyList(), null);
      }
    });
  }
}
```

The only thing that remains for us is to connect our `MovieAdapter` with our `MovieDataSource` using a [`PagedList`](https://developer.android.com/reference/android/arch/paging/PagedList). 

To use a `PagedList`, we first create a [`PagedList.Config`](https://developer.android.com/reference/android/arch/paging/PagedList.Config). We also need to create two [`Executor`](https://developer.android.com/reference/java/util/concurrent/Executor) which will be used to _execute_ fetch and notification calls from the `PagedList`, i.e., data fetching via `DataSource` will execute on the _fetch executor_ and load- and boundary- callbacks will be executed by the _notify executor_.

```java
import java.util.concurrent.Executor;
import java.util.concurrent.ExecutorService;

// Ui thread executor to execute runnable on UI thread
class UiThreadExecutor implements Executor {
  private Handler handler = new Handler(Looper.getMainLooper());
  
  @Override public void execute(@NonNull Runnable command) {
    handler.post(command);
  }
}

// Background thread executor to execute runnable on background thread
class BackgroundThreadExecutor implements Executor {
  private ExecutorService executorService = 
        Executors.newFixedThreadPool(2);
  
  @Override public void execute(@NonNull Runnable command) {
    executorService.execute(command);
  }
}
```

Then in your `Activity`
```java
@Override protected void onCreate(Bundle savedInstanceState) {
  super.onCreate(savedInstanceState);

  // usual onCreate code...

  // setup recycler view
  RecyclerView recycler = findViewById(R.id.movies);
  recycler.setLayoutManager(new GridLayoutManager(this, 2));
  // create new movie adapter
  MovieAdapter adapter = new MovieAdapter(Movie.DIFF_CALLBACK);
  // add adapter to recyclerview
  recycler.setAdapter(adapter);
  
  // create data source
  MoviesDataSource dataSource = new MoviesDataSource(getRetofitApi());
  // create configuration
  // see https://is.gd/l4hv42
  PagedList.Config config = new PagedList.Config.Builder()
      // this is not used by TMDb as we cannot pass page size to API
      // but is required by PagedList to use as hint
      .setPageSize(10)
      .setPrefetchDistance(5)
      .build();
  // create paged list
  PagedList<Movie> pagedList = 
    new PagedList.Builder<>(dataSource, config)
      // get list notifications on Ui thread
      .setNotifyExecutor(new UiThreadExecutor())
      // execute fetches on a background thread
      .setFetchExecutor(new BackgroundThreadExecutor())
      // build!
      .build();
  // update adapter's paged list (and data source)
  adapter.submitList(pagedList);
}
```

That's it! In a (sort of) few lines of code (if we remove the inherent verbosity in Java), we get a `RecyclerView.Adapter` that is capable of _efficiently_ loading data and presenting it on the UI all while keeping it _low_ on resources and preventing janky UI!


### Bonus!
Here is the source of the `retrofit2.Callback<T>` with _exponential backoff_ retry policy, i.e., it will retry the request each time on failure with exponentially increasing delay to prevent network and API abuse!

```java
// retrofit callback implementation with retry policy
// adapted from https://stackoverflow.com/a/41884400/6611700
// adapted from https://is.gd/KaEwPt (exponential delay)
abstract class CallbackWithRetry<T> implements retrofit2.Callback<T> {
  // exponential backoff delay
  private static final int RETRY_DELAY = 300; /* ms */
  
  // max allowed retries
  private final int maxAttempts;
  // current attempts
  private int attempts = 0;

  CallbackWithRetry(int max){
    this.maxAttempts = max;
  }
  
  @Override public final void onFailure(Call<T> call, Throwable t) {
    attempts++;
    if(attempts < maxAttempts){
      int delay = 
          (int) (RETRY_DELAY * Math.pow(2, Math.max(0, attempts - 1)));
      new Handler().postDelayed(() -> retry(call), delay);
    } else {
      onFinalFailure(call, t);
    }
  }
  
  // failure hook called when retries are consumed
  public abstract void onFinalFailure(Call<T> call, Throwable t);
  
  private void retry(Call<T> call){
    call.clone().enqueue(this);
  }
}
```

Hope you enjoyed the post! If you've any doubts regarding any of the stuff here feel free to [open an issue on GitHub](https://github.com/riyaz-ali/riyaz-ali.github.io/issues/new){:target="_blank"} and I'll try my best to explain it! Till then goodbye!