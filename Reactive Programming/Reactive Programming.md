# Definition
Reactive programming is all about **data streams** that can change over time, and your code "reacts" to those changes automatically.
- **RxJava:** the core library, written in Java.
- **RxKotlin:** a thin wrapper around RxJava with nicer Kotlin extensions (e.g., `subscribeBy`, `toObservable()`).
- **RxAndroid:** extra Android-specific bits, mainly:
    - `AndroidSchedulers.mainThread()` → makes it easy to observe results on the UI thread.
**Core Principles:**
- **Asynchronous**: non-blocking, background-friendly.
- **Event streams**: sequences of values/events over time.
- **Operators**: functions to transform, filter, or combine streams.
- **Declarative**: describe _what_ should happen, not _how_ to do it.
# Comparison
**Rx vs [[Coroutines]]/[[Flows]]:**
- **Flow** is Kotlin’s native reactive streams.
- RxJava is older, richer in operators, but more complex.
- Concept mapping:
    - `Observable` ↔ `Flow`
    - `subscribe` ↔ `collect`
    - `Schedulers` ↔ `Dispatchers`

**Rx vs LiveData:**
- LiveData is **UI lifecycle-aware** (auto unsubscribes).
- RxJava is **general-purpose** (you manage disposal).
- LiveData has very few operators, Rx has 100+.

**Quick Analogy:**
- **LiveData** = bicycle (simple, safe, UI only).
- **RxJava** = motorcycle (powerful, flexible, needs careful handling).
- **Flow** = modern e-bike (simpler, Kotlin-native, still powerful).

| Feature                 | **LiveData**                                               | **RxJava / Flow**                                                        |
| ----------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------ |
| **Purpose**             | Lifecycle-aware data holder for Android UI                 | General-purpose reactive streams                                         |
| **Threading**           | Mostly main thread (manual for background work)            | Built-in thread control with `subscribeOn`/`observeOn`                   |
| **Operators**           | Very limited (`map`, `switchMap`)                          | Huge arsenal (100+ operators: debounce, combineLatest, zip, retry, etc.) |
| **Cold/Hot**            | Always hot (keeps latest value, delivers to new observers) | Both cold (`Observable`/`Flow`) and hot (`PublishSubject`)               |
| **Lifecycle awareness** | Auto-stops observing when lifecycle is destroyed           | No lifecycle awareness (you manage disposal)                             |
| **Scope**               | UI-centric                                                 | UI + networking + DB + complex event streams                             |

# Core concepts
## `Observable`
Can emit **0, 1, or many** items.
``` kotlin
Observable.just(1, 2, 3)
    .subscribe(
        { println("Item: $it") },     // onNext
        { error -> println("Error: $error") }, // onError
        { println("Completed") }      // onComplete
    )
    
/* Output
Item: 1
Item: 2
Item: 3
Completed
*/
```
**Types of Observables:**
- `Observable<T>` → unlimited items, no backpressure.
- `Single<T>` → exactly 1 item or error.
- `Maybe<T>` → 0 or 1 item, or error.
- `Completable` → no items, only success/error.

## `Flowable`
### Definition
- Similar to `Observable`, **but with backpressure handling**.
- Use when you expect a **high-frequency stream** (e.g., thousands of events/second).
- Helps prevent `OutOfMemoryError` when the consumer is slower than the producer.

### Backpressure handling
**`BUFFER`:**
- Store _all_ events until consumer is ready.
- Risk: memory overload.
``` kotlin
Flowable.create<Int>({ emitter ->
    for (i in 1 .. 1_000_000) emitter.onNext(i)
    emitter.onComplete()
}, BackpressureStrategy.BUFFER)
.subscribe { println(it) }
```

**`DROP`:**
- Drop events when consumer is busy.
- Good for events where only _some_ updates matter.
``` kotlin
Flowable.create<Int>({ emitter ->
    for (i in 1 .. 1_000_000) emitter.onNext(i)
    emitter.onComplete()
}, BackpressureStrategy.DROP)
.subscribe { println(it) }
```

**`LATEST`:**
- Keep only the most recent value.
- Consumer always processes the **latest item**.
- Great for UI updates (e.g., scroll position).
``` kotlin
Flowable.create<Int>({ emitter ->
    for (i in 1 .. 1_000_000) emitter.onNext(i)
    emitter.onComplete()
}, BackpressureStrategy.LATEST)
.subscribe { println("Latest: $it") }
```

**`ERROR`:**
- Throw `MissingBackpressureException` if consumer is too slow.
- Forces you to handle backpressure manually.
``` kotlin
Flowable.create<Int>({ emitter ->
    for (i in 1 .. 1_000_000) emitter.onNext(i)
    emitter.onComplete()
}, BackpressureStrategy.ERROR)
.subscribe(
    { println(it) },
    { error -> println("Error: $error") }
)
```

**Choosing a Strategy:**
- **BUFFER** → small, bounded streams.
- **DROP** → high-frequency events where missing values is OK (e.g., mouse moves).
- **LATEST** → only care about most recent value (e.g., UI state).
- **ERROR** → strict handling, useful in debugging.

## One-shot types
- Sometimes you don’t need a full `Observable` stream (many items).
- Instead, you want **one result** (e.g., network call), or maybe no result at all.
- These types simplify intent and make APIs clearer.

### `Single<T>`
- Emits exactly **one item** or **error**.
- Similar to a `suspend fun` that returns a value.
``` kotlin
Single.just("Hello RxKotlin")
    .subscribe(
        { println("Success: $it") },
        { error -> println("Error: $error") }
    )
```
✅ Use for **network requests, DB queries, calculations**.

### `Maybe<T>`
- Emits **zero or one item**, or **error**.
- If nothing is emitted → completes silently.
``` kotlin
Maybe.just("Optional value")
    .subscribe(
        { println("Success: $it") },     // onSuccess
        { error -> println("Error: $error") }, // onError
        { println("Completed with no value") } // onComplete
    )
```
✅ Use for **optional results** (e.g., “get user if exists”).

### `Completable`
- Emits **no items**.
- Only signals **completion** or **error**.
``` kotlin
Completable.fromAction {
    println("Task done!")
}
.subscribe(
    { println("Completed") },
    { error -> println("Error: $error") }
)
```
✅ Use for **tasks without return values** (e.g., insert into DB, cache clear, file save).

## Hot vs Cold Observables
### Cold Observables
- Start emitting values **only when subscribed**.
- Each subscriber gets its **own independent sequence**.
``` kotlin
val cold = Observable.just(1, 2, 3)

cold.subscribe { println("Subscriber 1: $it") }
cold.subscribe { println("Subscriber 2: $it") }

/* Output
Subscriber 1: 1,2,3
Subscriber 2: 1,2,3
*/
```
✅ Each subscriber gets the **whole sequence**.

### Hot Observables
- Emit values **regardless of subscribers**.
- New subscribers join “mid-stream” → may **miss previous events**.
``` kotlin
val hot = PublishSubject.create<Int>()

hot.subscribe { println("Subscriber 1: $it") }

hot.onNext(1)
hot.onNext(2)

hot.subscribe { println("Subscriber 2: $it") }

hot.onNext(3)

/* Output
Subscriber 1: 1
Subscriber 1: 2
Subscriber 1: 3
Subscriber 2: 3
*/
```
✅ Subscriber 2 only sees events **after it subscribed**.

|Feature|Cold Observable|Hot Observable|
|---|---|---|
|When emission starts|When subscribed|Independent of subscription|
|Subscribers see|Full sequence (own copy)|Only future emissions|
|Use cases|API calls, DB queries|UI events, sensors, clicks

## Subjects
### Definition
- A **bridge** between an **Observable** and an **Observer**.
- Can both:
    - **emit events** (`onNext`, `onComplete`, `onError`)
    - **subscribe to events**.
- Typically used to create **Hot Observables**.

### `PublishSubject`
- Emits **only items after subscription**.
- Subscribers joining late miss earlier values.
``` kotlin
val subject = PublishSubject.create<Int>()

subject.subscribe { println("S1: $it") }
subject.onNext(1)
subject.onNext(2)

subject.subscribe { println("S2: $it") }
subject.onNext(3)

/* Output
S1: 1
S1: 2
S1: 3
S2: 3
*/
```

### `BehaviorSubject`
- Emits the **most recent item** to new subscribers, then continues.
- Always has a **“current value”**.
``` kotlin
val subject = BehaviorSubject.createDefault(0)

subject.subscribe { println("S1: $it") }
subject.onNext(1)
subject.onNext(2)

subject.subscribe { println("S2: $it") } // gets last value immediately
subject.onNext(3)

/* Output
S1: 0
S1: 1
S1: 2
S2: 2
S1: 3
S2: 3
*/
```

### `ReplaySubject`
Replays **all past items** to new subscribers.
``` kotlin
val subject = ReplaySubject.create<Int>()

subject.onNext(1)
subject.onNext(2)

subject.subscribe { println("S1: $it") } // receives old events too
subject.onNext(3)

/* Output
S1: 1
S1: 2
S1: 3
*/
```

### `AsyncSubject`
Emits **only the last value** (if any), and only when `onComplete()` is called.
``` kotlin
val subject = AsyncSubject.create<Int>()

subject.onNext(1)
subject.onNext(2)
subject.subscribe { println("S1: $it") }
subject.onNext(3)
subject.onComplete()

/* Output
S1: 3
*/
```

| Subject             | New subscribers get...           | Example use case                   |
| ------------------- | -------------------------------- | ---------------------------------- |
| **PublishSubject**  | Only future emissions            | Button clicks                      |
| **BehaviorSubject** | Last value + future emissions    | UI state (last known + updates)    |
| **ReplaySubject**   | All past + future emissions      | Chat history                       |
| **AsyncSubject**    | Only last value after completion | Final result of a long computation |

**Best Practices:**
- Use Subjects carefully → they can easily lead to spaghetti code if abused.
- Prefer using Observable operators when possible.
- Subjects are best for:
	- Bridging non-Rx APIs into Rx (e.g., callback → stream).
	- Event buses (though this is controversial).
	- Caching state (`BehaviorSubject`).

# Subscribing & Disposing
## `Subscribe` & `SubscribeBy`
- `Observable`, `Flowable`, `Single`, etc. **don’t do anything** until you call `subscribe()`.
- Subscribing = start listening to the stream.

### `.subscribe()`
``` kotlin
Observable.just(1, 2, 3)
    .subscribe(
        { item -> println("onNext: $item") },        // onNext
        { error -> println("onError: $error") },     // onError
        { println("onComplete") }                    // onComplete
    )
```

### `.subscribeBy()` (`RxKotlin` extension)
``` kotlin
Observable.just(1, 2, 3)
    .subscribeBy(
        onNext = { println("Next: $it") },
        onError = { println("Error: $it") },
        onComplete = { println("Done") }
    )
```

### Subscribing to Other Types
**Single:**
``` kotlin
Single.just("User")
    .subscribeBy(
        onSuccess = { println("Success: $it") },
        onError = { println("Error: $it") }
    )
```

**Maybe:**
``` kotlin
Maybe.empty<String>()
    .subscribeBy(
        onSuccess = { println("Value: $it") },
        onComplete = { println("No value") },
        onError = { println("Error: $it") }
    )
```

**Completable:**
``` kotlin
Completable.complete()
    .subscribeBy(
        onComplete = { println("Task finished") },
        onError = { println("Error: $it") }
    )
```

> [!WARNING]
> - `subscribe()` returns a **Disposable** → you must manage it.
> - Without disposal, subscriptions may **leak memory** (especially on Android).

## `Disposable` & `CompositeDisposable`
- Prevent **memory leaks** (especially in Android Activities/Fragments).
- Streams like `interval`, `PublishSubject`, or infinite Observables never complete → will leak if not disposed.
### Disposable
- Result of calling `.subscribe()`.
- Represents the **connection** between an Observable and its subscriber.
- Can be **disposed** to stop receiving emissions.
``` kotlin
val disposable = Observable.interval(1, TimeUnit.SECONDS)
    .subscribe { println("Tick: $it") }

// Later, when no longer needed:
disposable.dispose()
```