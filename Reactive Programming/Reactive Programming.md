
| Table of contents            |
| ---------------------------- |
| [[Core concepts]]            |
| [[Operators]]                |
| [[Schedulers & Threading]]   |
| [[Advanced Usage]]           |
| [[Patterns & Architectures]] |
| [[Rx Community]]             |

# 1. Definition
Reactive programming is all about **data streams** that can change over time, and your code "reacts" to those changes automatically.
- **RxJava:** the core library, written in Java.
- **RxKotlin:** a thin wrapper around RxJava with nicer Kotlin extensions (e.g., `subscribeBy`, `toObservable()`).
- **RxAndroid:** extra Android-specific bits, mainly:
    - `AndroidSchedulers.mainThread()` → makes it easy to observe results on the UI thread.

# 2. Comparison
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

