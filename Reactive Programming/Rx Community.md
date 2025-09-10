# `RxBinding`
- A set of **RxJava bindings for Android UI widgets**.
- Created by Jake Wharton.
- Converts typical UI callbacks (`setOnClickListener`, `TextWatcher`, etc.) into **Observables**.
- Works perfectly with **MVVM** and **MVI** architectures.

**Button clicks:**
``` kotlin
RxView.clicks(myButton)
    .throttleFirst(1, TimeUnit.SECONDS) // prevent double clicks
    .subscribe { 
        println("Button clicked!") 
    }
```

**Text changes:**
``` kotlin
RxTextView.textChanges(searchEditText)
    .debounce(300, TimeUnit.MILLISECONDS) // wait for typing to stop
    .map { it.toString() }
    .distinctUntilChanged()
    .switchMapSingle { query ->
        repo.searchUsers(query)
            .subscribeOn(Schedulers.io())
    }
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe { users -> showUsers(users) }
```

**Checkbox / Switch:**
``` kotlin
RxCompoundButton.checkedChanges(mySwitch)
    .subscribe { isChecked ->
        println("Switch is $isChecked")
    }
```

**`RecyclerView` scroll:**
``` kotlin
RxRecyclerView.scrollEvents(recyclerView)
    .subscribe { event ->
        println("Scroll state: ${event.scrollState()}")
    }
```

# `RxRelay`
- A library by **Jake Wharton** (same author of `RxBinding`).
- Provides **`Relay`** types → similar to `Subject`, but with a big difference:  
    ✅ They **never complete**  
    ✅ They **never error**
- Ideal for **UI events** (clicks, navigation, form inputs) where you don’t want the stream to die.

## Types
Relays mirror `Subject`s:
- **`PublishRelay<T>`**
    - Like `PublishSubject`.
    - Emits to subscribers only items after subscription.
    - Common for UI events (clicks, navigation).
- **`BehaviorRelay<T>`**
    - Like `BehaviorSubject`.
    - Stores the latest value → new subscribers immediately get the last item.
    - Useful for UI state.
- **`ReplayRelay<T>`**
    - Like `ReplaySubject`.
    - Buffers values for new subscribers (configurable size).

## Example
``` kotlin
val clickRelay = PublishRelay.create<Unit>()

// Emit event
myButton.setOnClickListener { clickRelay.accept(Unit) }

// Subscribe
clickRelay.subscribe { println("Button clicked!") }
```

``` kotlin
val stateRelay = BehaviorRelay.createDefault("Idle")

stateRelay.subscribe { println("State: $it") }

stateRelay.accept("Loading")   // prints "State: Loading"
stateRelay.accept("Success")   // prints "State: Success"
```

``` kotlin
val relay = ReplayRelay.create<Int>()

relay.accept(1)
relay.accept(2)

relay.subscribe { println("Received: $it") }
// prints both 1 and 2 immediately
```

## Why Use Relays Instead of Subjects?
- In UI, `onComplete()` or `onError()` are dangerous → they stop the stream forever.
- With `Relay`, streams are **immortal**.
- Prevents accidental termination from breaking your UI event bus.
- Cleaner semantics: **UI events should never error/complete**.

# `RxPreferences`
- A wrapper around **Android `SharedPreferences`**.
- Turns stored values into **reactive streams**.
- Great for **settings, feature flags, user preferences**.
- Built on top of RxJava → every preference can be **observed** for changes.

## Setup
``` kotlin
val sharedPreferences = PreferenceManager.getDefaultSharedPreferences(context)
val rxPreferences = RxSharedPreferences.create(sharedPreferences)
```

## Getting a Preference
``` kotlin
val username: Preference<String> = rxPreferences.getString("username", "Guest")
val notifications: Preference<Boolean> = rxPreferences.getBoolean("notifications", true)
```
The returned `Preference<T>` gives you:
- `get()` → current value
- `set(value)` → update value
- `asObservable()` → reactive stream of changes

## Example
``` kotlin
username.set("Alice")
println(username.get()) // prints "Alice"
```

``` kotlin
username.asObservable()
    .subscribe { value ->
        println("Username changed to: $value")
    }
```

## Operators on Preferences
``` kotlin
username.asObservable()
    .filter { it.isNotEmpty() }
    .map { it.uppercase() }
    .subscribe { println("Transformed: $it") }
```

# `RxPermissions`
- A small library that wraps Android’s **runtime permission requests** with RxJava.
- No need to manually override `onRequestPermissionsResult`.
- Requests become **Observables** → easy to compose with your other Rx streams.
- Created by **tbruyelle**.

## Setup
``` kotlin
val rxPermissions = RxPermissions(this) // in Activity/Fragment
```

## Basic
``` kotlin
rxPermissions.request(Manifest.permission.CAMERA)
    .subscribe { granted ->
        if (granted) {
            println("Camera granted ✅")
        } else {
            println("Camera denied ❌")
        }
    }
```

## Multiple Permissions
``` kotlin
rxPermissions.request(
    Manifest.permission.CAMERA,
    Manifest.permission.READ_CONTACTS
).subscribe { granted ->
    if (granted) {
        println("All permissions granted ✅")
    } else {
        println("Some permissions denied ❌")
    }
}
```

## Detail result
``` kotlin
rxPermissions.requestEach(
    Manifest.permission.CAMERA,
    Manifest.permission.READ_CONTACTS
).subscribe { permission ->
    when {
        permission.granted -> println("${permission.name} granted ✅")
        permission.shouldShowRequestPermissionRationale ->
            println("${permission.name} denied, ask again later 🔄")
        else -> println("${permission.name} denied permanently ❌")
    }
}
```