# 1. Transforming Operators
## 1.1. `map`
- **One-to-one transformation**.
- Takes each emitted item → transforms → emits new item.
``` kotlin
Observable.just(1, 2, 3)
    .map { it * 10 }
    .subscribe { println(it) } 
    
// Output: 10, 20, 30
```

## 1.2. `flatMap`
- **One-to-many transformation**.
- Each item → mapped into an Observable → all merged together.
- Order **not guaranteed**.
``` kotlin
Observable.just("A", "B")
    .flatMap { letter ->
        Observable.fromIterable(listOf(1, 2))
            .map { num -> "$letter$num" }
    }
    .subscribe { println(it) }
    
// Output (order may vary): A1, A2, B1, B2
```

## 1.3. `concatMap`
- Like `flatMap`, but **preserves order**.
- Waits until inner Observable completes before moving to the next.
``` kotlin
Observable.just("A", "B")
    .concatMap { letter ->
        Observable.fromIterable(listOf(1, 2))
            .map { num -> "$letter$num" }
    }
    .subscribe { println(it) }
    
// Output (ordered): A1, A2, B1, B2
```

## 1.4. `switchMap`
- Always switches to the **latest Observable**.
- Useful for search/autocomplete (ignore outdated results).

``` kotlin
val queryStream = PublishSubject.create<String>()

queryStream
    .switchMap { query ->
        Observable.just("Result for $query")
            .delay(300, TimeUnit.MILLISECONDS) // simulate network delay
    }
    .subscribe { println(it) }

queryStream.onNext("Hello")
queryStream.onNext("Hel")
queryStream.onNext("Help")

// Only "Result for Help" will print (latest query)
```

## 1.5. `groupBy`
- Split stream into **groups by key**.
- Each group is its own Observable.
``` kotlin
Observable.just(1, 2, 3, 4, 5, 6)
    .groupBy { it % 2 == 0 } // even or odd
    .subscribe { group ->
        group.subscribe { println("Group ${group.key}: $it") }
    }
    
// Output: 
// Group false: 1,3,5
// Group true: 2,4,6
```

# 2. Filtering Operators
`filter`
- Emit only items that match a condition.

`take(n)` / `takeLast(n)`
- `take(n)` → only first `n` items.
- `takeLast(n)` → only last `n` items.

`skip(n)` / `skipLast(n)`
- `skip(n)` → ignore first `n` items.
- `skipLast(n)` → ignore last `n` items.

`distinct()` / `distinctUntilChanged()`
- `distinct()` → remove duplicates entirely.
- `distinctUntilChanged()` → remove only consecutive duplicates.

`first()` / `last()` / `elementAt()`
- Select specific element.

`debounce(timeout)`
- Emit item only if no new item arrives within the timeout.
- Useful for **search inputs**, **button clicks**, etc.
``` kotlin
val subject = PublishSubject.create<String>()

subject
    .debounce(300, TimeUnit.MILLISECONDS)
    .subscribe { println("Search for: $it") }

subject.onNext("H")
subject.onNext("He")
subject.onNext("Hel")
Thread.sleep(400)
subject.onNext("Hell")
subject.onNext("Hello")

// Output: "Hel", "Hello" (skips rapid inputs)
```

`sample(period)`
- Emit the **latest item** in a time window.
``` kotlin
Observable.interval(100, TimeUnit.MILLISECONDS)
    .sample(300, TimeUnit.MILLISECONDS)
    .subscribe { println(it) }
// Output: ~0, 3, 6, 9...
```

`throttleFirst(period)`
- Emit only the **first item** in a time window.
- Common for button click prevention (anti-double-click).
``` kotlin
val clicks = PublishSubject.create<Unit>()

clicks
    .throttleFirst(1, TimeUnit.SECONDS)
    .subscribe { println("Button clicked") }

clicks.onNext(Unit)
clicks.onNext(Unit) // ignored if within 1s
```

# 3. Combining Operators
## 3.1. `merge`
- Combine emissions from multiple Observables into one.
- Order depends on emission timing.
``` kotlin
Observable.merge(
    Observable.just("A1", "A2"),
    Observable.just("B1", "B2")
).subscribe { println(it) }
// Output: A1, A2, B1, B2 (order may vary)
```

## 3.2. `concat`
- Combine Observables **sequentially**.
- One completes before next starts.
``` kotlin
Observable.concat(
    Observable.just("First"),
    Observable.just("Second")
).subscribe { println(it) }
// Output: First, Second
```

## 3.3. `zip`
- Combine emissions **pair by pair**.
- Stops when the shortest Observable completes.
``` kotlin
Observable.zip(
    Observable.just("A", "B", "C"),
    Observable.just(1, 2, 3)
) { letter, number -> "$letter$number" }
.subscribe { println(it) }
// Output: A1, B2, C3
```

## 3.4. `combineLatest`
- Emit when **any source emits**, combining the latest value from each.
``` kotlin
val letters = PublishSubject.create<String>()
val numbers = PublishSubject.create<Int>()

Observable.combineLatest(letters, numbers) { l, n ->
    "$l$n"
}.subscribe { println(it) }

letters.onNext("A")
numbers.onNext(1)   // Output: A1
numbers.onNext(2)   // Output: A2
letters.onNext("B") // Output: B2
```

## 3.5. `startWith`
- Prepend initial items before the stream.
``` kotlin
Observable.just("B", "C")
    .startWith("A")
    .subscribe { println(it) }
// Output: A, B, C
```

### 3.5.1. `switchOnNext`
- Flatten an Observable-of-Observables, but always switch to the **latest inner stream**.
- Useful for dynamically changing data sources.
``` kotlin
val outer = PublishSubject.create<Observable<String>>()

Observable.switchOnNext(outer)
    .subscribe { println(it) }

outer.onNext(Observable.just("A", "B"))
outer.onNext(Observable.just("C", "D"))
// Output: C, D  (switched to latest)
```

| Operator        | Use case                                           |
| --------------- | -------------------------------------------------- |
| `merge`         | Parallel events (e.g., multiple buttons, sensors). |
| `concat`        | Sequential tasks (e.g., login → load profile).     |
| `zip`           | Combine related data pairs (e.g., API responses).  |
| `combineLatest` | Form values together (e.g., form validation).      |
| `startWith`     | Add default or initial state.                      |
| `switchOnNext`  | Always listen to latest data source.               |

# 4. Error Handling Operators
## 4.1. `onErrorReturn`
- Replace an error with a fallback **single value**, then complete.
``` kotlin
Observable.error<String>(RuntimeException("Boom!"))
    .onErrorReturn { "Fallback value" }
    .subscribe { println(it) }
// Output: Fallback value
```

## 4.2. `onErrorResumeNext`
- Replace an error with **another Observable**.
``` kotlin
Observable.error<String>(Exception("Network error"))
    .onErrorResumeNext(Observable.just("Cached data"))
    .subscribe { println(it) }
// Output: Cached data
```

## 4.3. `onExceptionResumeNext`
- Similar to above, but **only for Exceptions** (not `Throwable`).
``` kotlin
Observable.error<String>(Exception("Oops"))
    .onExceptionResumeNext(Observable.just("Recovered"))
    .subscribe { println(it) }
// Output: Recovered
```

## 4.4. `retry()`
- Re-subscribe after error.
- `retry(n)` → retry up to `n` times.
- `retry()` → infinite retries.
``` kotlin
var count = 0

Observable.fromCallable {
    if (count++ < 2) throw RuntimeException("Fail")
    "Success"
}
.retry(3)
.subscribe { println(it) }
// Output: Success
```

## 4.5. `retryWhen`
- More control: retry **based on another Observable** (e.g., with delay).
``` kotlin
Observable.error<String>(RuntimeException("Fail"))
    .retryWhen { errors ->
        errors.delay(2, TimeUnit.SECONDS) // wait before retry
    }
    .subscribe(
        { println("Got: $it") },
        { println("Error: $it") }
    )
```

## 4.6. `doOnError`
- Side effect (e.g., logging) when error happens.
``` kotlin
Observable.error<String>(RuntimeException("Oops"))
    .doOnError { println("Log: $it") }
    .onErrorReturn { "Fallback" }
    .subscribe { println(it) }
// Output: 
// Log: java.lang.RuntimeException: Oops
// Fallback
```

## 4.7. Strategy Tips
- **API Calls**:
    - `retryWhen` with exponential backoff.
    - `onErrorResumeNext` for cached/fallback data.
- **UI Layer**:
    - `onErrorReturn` for showing default placeholder.
    - Log errors with `doOnError`.
- **Don’t swallow all errors blindly** → can hide bugs.

| Operator                | Behavior                                          |
| ----------------------- | ------------------------------------------------- |
| `onErrorReturn`         | Emit fallback item                                |
| `onErrorResumeNext`     | Switch to fallback Observable                     |
| `onExceptionResumeNext` | Switch only for `Exception` (not all `Throwable`) |
| `retry(n)`              | Retry n times                                     |
| `retryWhen`             | Retry with custom strategy                        |
| `doOnError`             | Side effect logging                               |

# 5. Utility Operators
`doOnNext`
- Perform a **side effect** when an item is emitted.
- Commonly used for logging, analytics, debugging.
``` kotlin
Observable.just(1, 2, 3)
    .doOnNext { println("About to emit: $it") }
    .subscribe { println("Got: $it") }
// Output:
// About to emit: 1
// Got: 1
// ...
```

`doOnSubscribe` / `doOnDispose`
- Side effects when subscribing or disposing.

`doFinally`
- Runs once when the stream terminates (success, error, or dispose).

`delay`
- Delay emissions by a time amount.

`timeout`
- Throw error if no item is emitted within time.
``` kotlin
Observable.timer(2, TimeUnit.SECONDS)
    .timeout(1, TimeUnit.SECONDS)
    .subscribe(
        { println("Got: $it") },
        { println("Error: $it") }
    )
// Output: Error: TimeoutException
```

`timeInterval`
- Measure the time between emissions.
``` kotlin
Observable.interval(500, TimeUnit.MILLISECONDS)
    .timeInterval()
    .subscribe { println("Item ${it.value()} after ${it.time()}ms") }
```

`materialize` / `dematerialize`
- Convert notifications (`onNext`, `onError`, `onComplete`) into actual objects.
- Useful for debugging/testing.
``` kotlin
Observable.just(1, 2)
    .materialize()
    .subscribe { println(it) }
// Output: OnNextNotification(1), OnNextNotification(2), OnCompleteNotification
```

`using`
- Create a resource that is automatically disposed when the stream ends.
``` kotlin
Observable.using(
    { AutoCloseableResource() },
    { resource -> Observable.just(resource.use()) },
    { resource -> resource.close() }
).subscribe { println(it) }
```

# 6. Custom Operators
Rx has a lot of built-in Operators, but sometimes your business logic doesn’t quite fit. That’s when you write your own.

## 6.1. Extension Function Operators
In Kotlin, you can add operators as **extension functions** to `Observable`, `Flowable`, etc.
``` kotlin
fun Observable<Int>.square(): Observable<Int> {
    return this.map { it * it }
}

// Usage
Observable.just(1, 2, 3, 4)
    .square()
    .subscribe { println(it) } // 1, 4, 9, 16
```

## 6.2. Custom Filtering
``` kotlin
fun Observable<String>.onlyLongWords(minLength: Int): Observable<String> {
    return this.filter { it.length >= minLength }
}

// Usage
Observable.just("Rx", "Kotlin", "Reactive", "Fun")
    .onlyLongWords(5)
    .subscribe { println(it) } // "Kotlin", "Reactive"
```

## 6.3. Chaining Multiple Steps
``` kotlin
fun Observable<Int>.tripleAndFilterEven(): Observable<Int> {
    return this
        .map { it * 3 }
        .filter { it % 2 == 0 }
}

// Usage
Observable.range(1, 5)
    .tripleAndFilterEven()
    .subscribe { println(it) } // 6, 12
```

## 6.4. Using `compose()` for Reusable Transformations
- `compose()` lets you package an operator pipeline into a reusable function.
``` kotlin
fun <T> applySchedulers(): ObservableTransformer<T, T> {
    return ObservableTransformer { upstream ->
        upstream.subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
    }
}

// Usage
api.getUser()
    .compose(applySchedulers())
    .subscribe { user -> println("Got user: $user") }
```

## 6.5. Creating from Scratch with `Observable.create`
For advanced cases where you need **full control**.
``` kotlin
fun customRange(start: Int, count: Int): Observable<Int> {
    return Observable.create { emitter ->
        for (i in start until start + count) {
            if (!emitter.isDisposed) {
                emitter.onNext(i)
            }
        }
        emitter.onComplete()
    }
}

// Usage
customRange(5, 3).subscribe { println(it) } // 5, 6, 7
```