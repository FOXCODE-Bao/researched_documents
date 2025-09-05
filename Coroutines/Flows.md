# Context Controlling
## `flowOn(dispatcher)`
``` kotlin
val f = flow {
    println("Emitting on: ${Thread.currentThread().name}")
    emit(heavyComputation()) // simulate CPU work
}
.flowOn(Dispatchers.Default) // upstream context
```

## Producer vs Consumer Context
``` kotlin
flow {
    emit("Hello from ${Thread.currentThread().name}")
}
.flowOn(Dispatchers.IO) // upstream = IO
.collect {
    println("Collected '$it' on ${Thread.currentThread().name}")
}
```

**Output:**
```
Hello from DefaultDispatcher-IO
Collected 'Hello from DefaultDispatcher-IO' on main
```
Producer (`emit`) on **IO**, Consumer (`collect`) on **main**.
👉 ___`flowOn` only affects upstream of where it’s placed.___

## `launchIn(scope)`
**Instead of calling `collect`**, you can start a flow in a scope as a background job.
``` kotlin
val job = flow {
    emit(1); emit(2)
}.onEach {
    println("Got $it")
}.launchIn(CoroutineScope(Dispatchers.Default))
```
- Runs collection _asynchronously_ (non-blocking).
- Returns a `Job` you can cancel.

# Callback-based to Flow-based

| Feature                    | `callbackFlow`                      | `channelFlow`                     |
| -------------------------- | ----------------------------------- | --------------------------------- |
| Designed for               | Callback/listener APIs              | Multiple coroutines producing     |
| Cleanup                    | Requires `awaitClose {}`            | Not required                      |
| Typical usage              | Wrapping Android listeners, sockets | Merging async tasks into one Flow |
| Emission method            | `trySend`                           | `send` or `trySend`               |
| Concurrency inside builder | ❌ Usually single callback           | ✅ Multiple `launch` coroutines    |
## `callbackFlow`
### Definition
- A special builder for `Flow`.
- Designed to **wrap callback-based or listener APIs**.
- Provides a `SendChannel` (`trySend`, `send`) for emitting values.
- Requires you to handle resource cleanup with `awaitClose {}`.

### Example
**API:**
``` kotlin
interface LocationListener {
    fun onLocationChanged(location: String)
}
```

**Convert to Flows:**
``` kotlin
fun locationFlow(): Flow<String> = callbackFlow {
    val listener = object : LocationListener {
        override fun onLocationChanged(location: String) {
            trySend(location) // ✅ Non-suspending, safe
        }
    }

    // register the listener
    LocationManager.register(listener)

    // cleanup when flow collection is cancelled
    awaitClose { 
        LocationManager.unregister(listener) 
    }
}
```

**Collect:**
``` kotlin
locationFlow().collect { println("Location: $it") }
```

## `channelFlow`
### Definition
- Like `callbackFlow`, but more general.
- Allows **multiple coroutines inside builder** to emit values concurrently.
- Use when you want to combine different `async` sources into one flow.

### Example
``` kotlin
fun multiSourceFlow(): Flow<String> = channelFlow {
    // Producer 1
    launch {
        repeat(3) {
            send("From producer 1: $it")
            delay(100)
        }
    }

    // Producer 2
    launch {
        repeat(3) {
            send("From producer 2: $it")
            delay(150)
        }
    }
}

multiSourceFlow().collect { println(it) }

/* Output
** From producer 1: 0
** From producer 2: 0
** From producer 1: 1
** From producer 2: 1
** From producer 1: 2
** From producer 2: 2
*/
```

# Flow Operators
## `map` vs `mapLatest`

``` kotlin
val flow = flow {
    emit(1)
    delay(100)
    emit(2)
}
```

### `map`:
``` kotlin
flow.map { value ->
    delay(200) // pretend to process
    "Processed $value"
}.collect { println(it) }

/* Output
** Processed 1
** Processed 2
*/
```

### `mapLatest`:
``` kotlin
flow.mapLatest { value ->
    delay(200) // pretend to process
    "Processed $value"
}.collect { println(it) }

/* Output
** Processed 2
*/
```

## `flatMapConcat`, `flatMapMerge`, `flatMapLatest`

``` kotlin
fun numbersFlow(n: Int) = flow {
    emit("$n: First")
    delay(100)
    emit("$n: Second")
}
```

### `flatMapConcat` (sequential)
``` kotlin
flowOf(1, 2).flatMapConcat { numbersFlow(it) }
    .collect { println(it) }

/* Output
** 1: First
** 1: Second
** 2: First
** 2: Second
*/
```

### `flatMapMerge` (concurrent)
``` kotlin
flowOf(1, 2).flatMapMerge { numbersFlow(it) }
    .collect { println(it) }
    
/* Output
** 1: First
** 2: First
** 1: Second
** 2: Second
*/
```

### `flatMapLatest` (only keep latest)
``` kotlin
flowOf(1, 2).flatMapLatest { numbersFlow(it) }
    .collect { println(it) }

/* Output
** 1: First
** 2: First
** 2: Second
*/
```

## `debounce` vs `sample`
### `debounce`
Wait for a pause in emissions, then emit the latest item, it delays each emission until no new value arrives within the given timeout.
``` kotlin
flow {
    emit(1); delay(100)
    emit(2); delay(400)
    emit(3); delay(100)
    emit(4)
}.debounce(300)
 .collect { println(it) }
 
/* Timeline
** t=0    emit 1
** t=100  emit 2
** t=500  emit 3
** t=600  emit 4
*/

/* Output
** 2
** 4
*/
```
 With `debounce(300)`:
- `1` dropped (replaced quickly by `2`).
- `2` emitted at t=400 (since no new value for 300ms).
- `3` dropped (replaced quickly by `4`).
- `4` emitted at end.

### `sample()`
Emit the latest item at fixed intervals.
``` kotlin
flow {
    repeat(5) {
        emit(it)
        delay(100) // fast emissions
    }
}.sample(250)
 .collect { println(it) }

/* Timeline
** t=0   emit 0
** t=100 emit 1
** t=200 emit 2
** t=300 emit 3
** t=400 emit 4
*/

/* Output
** 2
** 4
*/
```
With `sample(250)`:
- At ~t=250 → latest value is `2`
- At ~t=500 → latest value is `4`

### Real-world Use Cases
- **`debounce`**:
    - Search text field → wait until user stops typing for 500ms.
    - Auto-saving draft → wait for pause in input.
- **`sample`**:
    - GPS updates every 50ms → but UI only needs 1 update per second.
    - Button mashing → allow one click every 300ms.

## `zip` vs `combine`
### `zip`
- **Waits** for both flows to produce the next element.
- Completes when the **shorter flow** finishes.
``` kotlin
val flow1 = flowOf(1, 2, 3)
val flow2 = flowOf("A", "B", "C", "D")

flow1.zip(flow2) { a, b -> "$a$b" }
    .collect { println(it) }

/* Output
** 1A
** 2B
** 3C
*/
```
Explanation:
- `flow1` has 3 items, `flow2` has 4.
- Zipping stops at the **shortest** = 3 items.
- Like a zipper on two sides → must move in sync.

### `combine`
- **Never waits** for both to emit _at the same time_.
- Emits as soon as there’s a new value from either side (after both have emitted once).
``` kotlin
val flow1 = flow {
    emit(1); delay(200)
    emit(2)
}
val flow2 = flow {
    delay(100)
    emit("X"); delay(200)
    emit("Y")
}

flow1.combine(flow2) { a, b -> "$a$b" }
    .collect { println(it) }

/*
** 1X   // first emission after both sides emitted something
** 2X   // flow1 updates, combine with latest "X"
** 2Y   // flow2 updates, combine with latest "2"
*/
```

## `fold` / `reduce` / `scan`

| Aspect        | `reduce`                                                    | `fold`                                                               |
| ------------- | ----------------------------------------------------------- | -------------------------------------------------------------------- |
| Initial value | First element of the flow                                   | Provided by you                                                      |
| Empty flow    | ❌ Exception                                                 | ✅ Returns initial                                                    |
| Use case      | When flow guaranteed non-empty and accumulator is same type | When you want explicit starting value, or result of a different type |
### `reduce`
- Takes the **first element** as the initial accumulator.
- Requires the flow to have **at least one element**, otherwise it **throws an exception**.
``` kotlin
val numbers = flowOf(1, 2, 3, 4)

val sum = numbers.reduce { acc, value ->
    println("acc=$acc, value=$value")
    acc + value
}

println("Result: $sum")
```

### `fold`
- Similar to `reduce`, but **you provide the initial value**.
- Works even on empty flows, because it doesn’t rely on the first element.
``` kotlin
val numbers = flowOf(1, 2, 3, 4)

val product = numbers.fold(1) { acc, value ->
    println("acc=$acc, value=$value")
    acc * value
}

println("Result: $product")
```

### `scan`
Like `fold`, but it **emits intermediate accumulation results**.  
Good for building running totals or progress states.
``` kotlin
(1..5).asFlow()
    .scan(0) { acc, value -> acc + value }
    .collect { println(it) }

/* Output
** 0
** 1
** 3
** 6
** 10
** 15
*/
```

###  `runningFold` / `runningReduce`
Shorthand for `scan`, more consistent with collection APIs.
``` kotlin
(1..5).asFlow()
    .runningFold(0) { acc, value -> acc + value }
    .collect { println(it) }
```

## `transform`
You can emit **zero, one, or many values** per input.
``` kotlin
(1..3).asFlow()
    .transform { value ->
        emit("Before $value")
        emit("After $value")
    }
    .collect { println(it) }
    
/* Output
** Before 1
** After 1
** Before 2
** After 2
** Before 3
** After 3
*/
```



# Exception Handling
- `catch` can **emit recovery values** → continues the flow.
- `onCompletion` is like a `finally` block → no modification, just side-effects.
- `retry` **re-subscribes** the whole flow upstream.

## `catch`
- Catches exceptions _upstream_ (before the operator where it’s used).
- Lets you handle the error, emit fallback values, or rethrow.
- Doesn’t catch downstream (inside `collect {}`).
``` kotlin
val numbers = flow {
    emit(1)
    emit(2)
    throw RuntimeException("Boom!")
    emit(3) // never reached
}

numbers
    .catch { e -> emit(-1) } // handled here
    .collect { println(it) }

/* Output
** 1
** 2
** -1
*/
```

## `onCompletion`
- Called when the flow _completes normally_ or _with an exception_.
- You get access to the `cause: Throwable?`.
``` kotlin
numbers
    .onCompletion { cause ->
        if (cause == null) println("Completed successfully")
        else println("Flow failed: $cause")
    }
    .catch { emit(-1) }
    .collect { println(it) }

/* Output
** 1
** 2
** Flow failed: java.lang.RuntimeException: Boom!
** -1
*/
```

# Backpressure Handling
- Producers can emit faster than consumers can collect.
- Without backpressure handling, memory grows → app lags or crashes.
- Flow gives **backpressure-safe operators**: `buffer`, `conflate`, `collectLatest`.

|Operator|Behavior|
|---|---|
|**buffer**|Decouple producer & consumer with a queue|
|**conflate**|Skip intermediate values, keep only latest|
|**collectLatest**|Cancel ongoing collector work, restart with latest|

## `buffer`
- Allows emissions to be **buffered** instead of *blocking* the producer.
- Size default = unlimited.
``` kotlin
val time = measureTimeMillis {
    flow {
        repeat(3) {
            delay(100) // producer is fast
            emit(it)
        }
    }
    .buffer() // decouple producer and consumer
    .collect { value ->
        delay(300) // consumer is slow
        println("Collected $value")
    }
}
println("Total time: $time ms")
```

## `conflate`
- Keeps **only the latest value** if consumer is too slow.
- Skips intermediate emissions.
``` kotlin
flow {
    repeat(5) {
        delay(100) // fast producer
        emit(it)
        println("Emitted $it")
    }
}
.conflate()
.collect { value ->
    delay(250) // slow consumer
    println("Collected $value")
}
```

## `collectLatest`
- **Cancels** the **previous collector block** if a new value arrives.
``` kotlin
flow {
    emit("A"); delay(100)
    emit("B"); delay(100)
    emit("C")
}
.collectLatest { value ->
    println("Start $value")
    delay(200) // simulate work
    println("Done $value")
}
```

