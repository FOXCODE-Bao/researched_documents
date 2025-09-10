# Parallelization
- Some tasks are independent (e.g., processing images, crunching numbers).
- Instead of processing sequentially, you can **split the workload across multiple threads**.
## Using `flatMap()` for Concurrency
``` kotlin
Observable.range(1, 5)
    .flatMap { number ->
        Observable.fromCallable {
            println("Processing $number on ${Thread.currentThread().name}")
            number * number
        }
        .subscribeOn(Schedulers.computation()) // run each in parallel
    }
    .subscribe { result -> println("Result: $result") }
```

## Using `parallel()` with Flowable
``` kotlin
Flowable.range(1, 10)
    .parallel(4) // split into 4 "rails"
    .runOn(Schedulers.computation())
    .map { number ->
        println("Processing $number on ${Thread.currentThread().name}")
        number * number
    }
    .sequential() // merge back to one stream
    .subscribe { result -> println("Result: $result") }
```

## Limiting Concurrency
- Sometimes you don’t want _all_ items to run in parallel (to avoid thread explosion).
- Use `.flatMap()` with **`maxConcurrency`**:
``` kotlin
Observable.range(1, 20)
    .flatMap(
        { num ->
            Observable.fromCallable {
                println("Processing $num on ${Thread.currentThread().name}")
                num * num
            }.subscribeOn(Schedulers.computation())
        },
        4 // max 4 concurrent tasks
    )
    .subscribe { println("Result: $it") }
```

- `flatMap + subscribeOn` → flexible, easy, but can create too many threads if unchecked.
- `parallel()` → structured parallel processing for large datasets (Flowable only).
- `maxConcurrency` → fine-tune how many tasks run at once.

# Subjects as Event Buses
## Definition
### What is an Event Bus?
- A **publish/subscribe mechanism**:
    - Components can **post events**.
    - Other components can **listen for events**.
- In Rx, a **Subject** can play this role.

### Why Subjects?
- Subjects are both **Observable** (emits) and **Observer** (receives).
- This makes them natural candidates for communication between different parts of an app.

## Example
### Step 1: Create a Singleton Event Bus
``` kotlin
object RxBus {
    private val bus = PublishSubject.create<Any>()

    fun send(event: Any) {
        bus.onNext(event)
    }

    fun toObservable(): Observable<Any> = bus
}
```

### Step 2: Post an Event
``` kotlin
RxBus.send("Hello from Activity A")
```

### Step 3: Listen for Events
``` kotlin
RxBus.toObservable()
    .ofType(String::class.java) // filter by type
    .subscribe { message ->
        println("Received: $message")
    }
```

## Example: Using `BehaviorSubject` for State Sharing
``` kotlin
object UserSessionBus {
    private val subject = BehaviorSubject.create<User>()

    fun updateUser(user: User) {
        subject.onNext(user)
    }

    fun observeUser(): Observable<User> = subject.hide()
}
```

# Resource Management
- Rx streams can be **infinite** (`interval`, user input, subjects).
- Subscriptions hold references → **leaks** if not disposed.
- On Android: you must tie streams to the **lifecycle** (Activity/Fragment/ViewModel).

### Disposable Basics
- Every `.subscribe()` returns a **Disposable**.
- Call `.dispose()` when you no longer need the stream.
``` kotlin
val disposable = Observable.interval(1, TimeUnit.SECONDS)
    .subscribe { println("Tick: $it") }

// Later...
disposable.dispose()
```

## `CompositeDisposable`
``` kotlin
val compositeDisposable = CompositeDisposable()

compositeDisposable.add(
    api.getUser().subscribe { println(it) }
)

compositeDisposable.add(
    api.getPosts().subscribe { println(it) }
)

// Clear on destroy
compositeDisposable.clear() // disposes all
```

## Lifecycle Awareness
RxJava itself is not lifecycle-aware, so you need patterns/libraries:
- **Manual disposal**: dispose in `onStop()` / `onDestroy()`.
- **autoDispose** (Uber library): disposes automatically based on lifecycle.
- **Architecture Components**: use ViewModel + expose LiveData/Flow when lifecycle matters.
``` kotlin
class MainActivity : AppCompatActivity() {
    private val disposables = CompositeDisposable()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        disposables.add(
            api.getUser()
                .subscribe { user -> showUser(user) }
        )
    }

    override fun onDestroy() {
        super.onDestroy()
        disposables.clear() // prevent leaks
    }
}
```

## Resource Cleanup with `using()`
``` kotlin
Observable.using(
    { FileInputStream("data.txt") }, // resource
    { stream -> Observable.fromCallable { stream.read() } }, // observable
    { stream -> stream.close() } // cleanup
)
.subscribe { println(it) }
```

# Advanced Subjects & Processors
- `Subject` works with **Observable** (no backpressure).
- `Processor` works with **Flowable** (handles backpressure).
- Think: _Subject + Backpressure = Processor_.

| Type                            | Behavior                               |
| ------------------------------- | -------------------------------------- |
| **PublishSubject / Processor**  | Fire-and-forget, no history            |
| **BehaviorSubject / Processor** | Keeps last value, great for state      |
| **ReplaySubject / Processor**   | Replays all events (or buffer-limited) |
| **AsyncSubject / Processor**    | Only last item, on completion          |
| **UnicastProcessor**            | Single-subscriber pipelines            |

## `PublishProcessor`
- Emits to **current subscribers only**.
- Good for event buses where you don’t need history.
``` kotlin
val processor = PublishProcessor.create<Int>()

processor.subscribe { println("Subscriber1: $it") }

processor.onNext(1)
processor.onNext(2)

processor.subscribe { println("Subscriber2: $it") }
processor.onNext(3)
```

## `BehaviorProcessor`
- Always emits the **latest item** to new subscribers.
- Good for “state sharing”.
``` kotlin
val processor = BehaviorProcessor.createDefault("Initial")

processor.subscribe { println("S1: $it") }

processor.onNext("A")
processor.onNext("B")

processor.subscribe { println("S2: $it") } // gets "B" immediately
```

## `ReplayProcessor`
- Replays **all previous items** to new subscribers.
- Can be bounded with a buffer size.
``` kotlin
val processor = ReplayProcessor.create<Int>()

processor.onNext(1)
processor.onNext(2)

processor.subscribe { println("S1: $it") } // gets 1, 2

processor.onNext(3)

processor.subscribe { println("S2: $it") } // gets 1, 2, 3
```

## `AsyncProcessor`
- Emits **only the last value** _when completed_.
- Rarely used, but useful for “final result” signals.
``` kotlin
val processor = AsyncProcessor.create<Int>()

processor.subscribe { println("S1: $it") }

processor.onNext(1)
processor.onNext(2)
processor.onNext(3)
processor.onComplete()
```

### `UnicastProcessor`
- Only **one subscriber** allowed.
- Buffers events until a subscriber appears.
``` kotlin
val processor = UnicastProcessor.create<Int>()

processor.onNext(1)
processor.onNext(2)

processor.subscribe { println("S1: $it") } // receives 1, 2
processor.onNext(3)
```

## Best Practices
- Use **Processor** if you need backpressure handling.
- Use **Subject** if you only work with `Observable` and no risk of overload.
- Avoid overusing Subjects/Processors as **global mutable state** → leads to hidden coupling.
- For UI state in modern Android → prefer **StateFlow/SharedFlow**.

# Interop with Coroutines & Flow
- Many old projects use **RxJava/RxKotlin**.
- New Jetpack libraries (Room, [[WorkManager]], DataStore, etc.) use **[[Flows]] / [[Coroutines]]**.
- You often need to **connect the two worlds**.

## From Rx to Flow
``` kotlin
// Observable → Flow
val flow: Flow<Int> = observable.toFlow()

// Flowable → Flow
val flow: Flow<Int> = flowable.asFlow()

// Single → Flow
val flow: Flow<Int> = single.asFlow()

// Maybe → Flow
val flow: Flow<Int> = maybe.asFlow()

// Completable → Flow
val flow: Flow<Unit> = completable.asFlow()
```

## From Flow to Rx
``` kotlin
// Flow → Observable
val observable: Observable<Int> = flow.asObservable()

// Flow → Flowable (with backpressure strategy)
val flowable: Flowable<Int> = flow.asFlowable(BackpressureStrategy.BUFFER)

// Flow → Single
val single: Single<Int> = flow.single().toSingle()

// Flow → Completable
val completable: Completable = flow.ignoreElements().toCompletable()
```

## From Rx to Suspend Functions
``` kotlin
// Single → suspend
suspend fun getUser(): User = api.getUser().await()

// Maybe → suspend (nullable)
suspend fun getUserOrNull(): User? = api.getUserMaybe().awaitNullable()

// Completable → suspend (no return)
suspend fun saveUser(user: User) = api.saveUser(user).await()
```

## From Suspend Functions to Rx
``` kotlin
fun getUserSingle(): Single<User> = single { emitter ->
    val user = getUserSuspend() // suspending function
    emitter.onSuccess(user)
}
```