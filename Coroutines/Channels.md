# 1. Definition
- **channel** is like a **queue** between [[Coroutines]].
- One coroutine can `send` values into it, another can `receive` them.
- It’s like a thread-safe `BlockingQueue`, but **suspending** (non-blocking).\
``` kotlin
val channel = Channel<Int>()

launch {
    repeat(5) {
        channel.send(it) // suspends until received if buffer is full
        println("Sent $it")
    }
    channel.close()
}

launch {
    for (value in channel) {
        println("Received $value")
    }
}
```

- Use **[[Flows]]** for data _processing pipelines_.
- Use **Channels** for _coroutine-to-coroutine communication_ (producers/actors).
# 2. Types
| Type                     | Behavior                                                                      |
| ------------------------ | ----------------------------------------------------------------------------- |
| **Rendezvous (default)** | Capacity = 0. `send` suspends until `receive` is ready. One-to-one handshake. |
| **Buffered**             | Capacity = N. `send` suspends only when buffer is full.                       |
| **Conflated**            | Only the **latest** value kept. New sends overwrite old if not consumed yet.  |
| **Unlimited**            | Essentially an unbounded queue (dangerous for memory leaks).                  |
``` Kotlin
val buffered = Channel<Int>(capacity = 2)
```

# 3. Closing Channels

``` kotlin
val ch = Channel<Int>()

launch {
    repeat(3) { ch.send(it) }
    ch.close()
}

for (x in ch) println(x)
```

>[!WARNING]
>If a coroutine consuming a channel is cancelled, the channel **is not automatically closed** — the producer may still send unless you manage cancellation explicitly.

# 4. Multiple channel receiving
``` kotlin
select<Unit> {
    channel1.onReceive { value -> println("Got $value from channel1") }
    channel2.onReceive { value -> println("Got $value from channel2") }
}
```

# 5. `consumeEach` vs `for (x in channel)`

- `for (x in channel)` is idiomatic and closes automatically when channel is closed.
- `consumeEach { }` is similar, but ensures **cancellation** of the producer if the consumer is cancelled.

# 6. Actors
An **actor** is a coroutine that owns some state and receives messages via a channel.
``` kotlin
fun CoroutineScope.counterActor() = actor<Int> {
    var count = 0
    for (msg in channel) {
        count += msg
    }
    println("Final count: $count")
}

val actor = counterActor()
actor.send(1)
actor.send(2)
actor.close()
```
💡 This is the **Actor model**: no locks, no race conditions, state confined to one coroutine.

