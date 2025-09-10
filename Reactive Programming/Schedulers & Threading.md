# Definition
Absolutely critical in Android, because you must keep heavy work off the main [[Thread]].
- Think of it as a **thread pool / execution context**.
- Controls _where_ (`subscribeOn`) and _where results are observed_ (`observeOn`).

# Operators
1. `subscribeOn(scheduler)`
	- Decides **which thread the source (upstream)** runs on.
	- Only the **first `subscribeOn`** in the chain matters.
2. `observeOn(scheduler)`
	- Decides **which thread downstream** runs on (where you observe results).
	- Can be called multiple times (affects everything after it).

# Common Schedulers

|Scheduler|Runs on…|
|---|---|
|`Schedulers.io()`|Background IO pool (network, DB, file I/O).|
|`Schedulers.computation()`|CPU-intensive work (calculations, transformations).|
|`Schedulers.newThread()`|Always creates a new thread (less common, avoid overuse).|
|`Schedulers.single()`|Single dedicated thread.|
|`AndroidSchedulers.mainThread()`|Android UI thread (from RxAndroid).|

**Example: Network call + UI update:**
``` kotlin
Observable.fromCallable {
    // Simulate network/API call
    Thread.sleep(1000)
    "Data from server"
}
.subscribeOn(Schedulers.io())              // run API call on IO thread
.observeOn(AndroidSchedulers.mainThread()) // observe on UI thread
.subscribe { result -> 
    println("Update UI with: $result")
}
```

**Example: Mixed computation:**
``` kotlin
Observable.just(1, 2, 3)
    .subscribeOn(Schedulers.computation())   // heavy calculation
    .map { it * it }
    .observeOn(Schedulers.io())              // save to DB
    .doOnNext { println("Saving $it") }
    .observeOn(AndroidSchedulers.mainThread()) // update UI
    .subscribe { println("Display $it") }
```

# Practical patterns
## Debounce (Search, User Input)
- Avoid firing too many events (e.g., typing in search bar).
``` kotlin
searchView.textChanges()
    .debounce(300, TimeUnit.MILLISECONDS)     // wait for user to pause typing
    .distinctUntilChanged()                   // ignore same query
    .switchMap { query -> api.search(query) } // cancel old requests
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe { results -> showResults(results) }
```

## Combine Multiple API Calls
- Example: fetch `User` and `Posts` together.
``` kotlin
Single.zip(
    api.getUser(userId),
    api.getPosts(userId),
    { user, posts -> Pair(user, posts) }
)
.subscribeOn(Schedulers.io())
.observeOn(AndroidSchedulers.mainThread())
.subscribe { (user, posts) ->
    showUserWithPosts(user, posts)
}
```

## Caching Strategy (Memory → Disk → Network)
``` kotlin
Observable.concat(
    memoryCache.getUser().toObservable(),
    diskCache.getUser().toObservable(),
    api.getUser().toObservable()
)
.firstElement() // take the first non-empty result
.observeOn(AndroidSchedulers.mainThread())
.subscribe { user -> showUser(user) }
```

## Retry with Exponential Backoff
- Good for unstable networks.
``` kotlin
api.getData()
    .retryWhen { errors ->
        errors.zipWith(
            Observable.range(1, 3),
            { error, retryCount -> retryCount }
        ).flatMap { retryCount ->
            Observable.timer(Math.pow(2.0, retryCount.toDouble()).toLong(), TimeUnit.SECONDS)
        }
    }
.subscribe { data -> showData(data) }
```

## Polling API (repeat every X seconds)
``` kotlin
Observable.interval(0, 5, TimeUnit.SECONDS)
    .flatMap { api.getUpdates().toObservable() }
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe { update -> showUpdate(update) }
```

## Form Validation (Combine Latest)
``` kotlin
Observable.combineLatest(
    emailField.textChanges(),
    passwordField.textChanges()
) { email, password ->
    email.isNotEmpty() && password.length > 6
}
.observeOn(AndroidSchedulers.mainThread())
.subscribe { isValid -> loginButton.isEnabled = isValid }
```