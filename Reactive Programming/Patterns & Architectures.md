This is where Rx truly shines: wiring up **UI → ViewModel → Repository → API/DB** flows with streams.
# MVVM with Rx
- **MVVM (Model–View–ViewModel)** is standard in Android.
- Rx makes:
    - **Events** reactive (button clicks, text changes).
    - **Data streams** live (DB, network).
    - **Thread switching** simple (Schedulers).

## Typical MVVM Layers

- **View (Activity/Fragment/Compose UI)**
    - Subscribes to `Observable` / `Flowable` from ViewModel.
    - Emits user input as Observables (clicks, text changes).
- **ViewModel**
    - Consumes streams from the View.
    - Exposes reactive state streams to the UI.
    - Talks to Repository with Rx.
- **Repository**
    - Coordinates network (Retrofit Rx adapter), DB (Room Rx support).
    - Returns Rx types: `Single`, `Maybe`, `Flowable`.

- Rx turns **UI events** + **data sources** into **streams**.
- ViewModel = middleman: transforms inputs into outputs with Rx operators.
- Repository = source of truth (network, DB), returning Rx types.
- Clean separation, reactive by design.

## Example
**Data → UI:**
``` kotlin
// Repository
class UserRepository(private val api: UserApi) {
    fun getUser(userId: String): Single<User> =
        api.getUser(userId) // Retrofit with RxJava adapter
}

// ViewModel
class UserViewModel(private val repo: UserRepository) {
    fun loadUser(userId: String): Single<User> =
        repo.getUser(userId)
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
}

// View (Activity/Fragment)
userViewModel.loadUser("42")
    .subscribeBy(
        onSuccess = { user -> showUser(user) },
        onError = { e -> showError(e) }
    )
    .addTo(compositeDisposable) // Dispose properly
```

**UI → Data:**
``` kotlin
// Example: RxBinding for text changes
RxTextView.textChanges(editText)
    .debounce(300, TimeUnit.MILLISECONDS)
    .distinctUntilChanged()
    .switchMapSingle { query ->
        repo.searchUsers(query.toString())
            .subscribeOn(Schedulers.io())
    }
    .observeOn(AndroidSchedulers.mainThread())
    .subscribeBy { users -> showUsers(users) }
    .addTo(compositeDisposable)
```

## State Management in ViewModel
``` kotlin
sealed class UserState {
    object Loading : UserState()
    data class Success(val user: User) : UserState()
    data class Error(val throwable: Throwable) : UserState()
}

class UserViewModel(private val repo: UserRepository) {
    private val state = BehaviorSubject.create<UserState>()
    fun observeState(): Observable<UserState> = state.hide()

    fun loadUser(userId: String) {
        repo.getUser(userId)
            .doOnSubscribe { state.onNext(UserState.Loading) }
            .subscribe(
                { user -> state.onNext(UserState.Success(user)) },
                { error -> state.onNext(UserState.Error(error)) }
            )
    }
}
```

## Best Practices
- Always **dispose** in `onDestroy` (or `onCleared` in ViewModel).
- Keep UI events (clicks, text) as `PublishSubject` or `Observable`.
- Expose only **immutable Observables** (`.hide()`) to prevent external `.onNext()`.
- Combine with **Schedulers** to move work off the main thread.

# Redux/MVI with Rx
- **MVI = Model–View–Intent**
- Inspired by **Redux** (JavaScript world).
- Goal: predictable, testable, **single source of truth** for UI state.
- Data flow is always **one-way**: View → Intent → Reducer → State → View

## Core Concepts
- **Intent**
    - User actions (button clicks, text inputs).
    - Represented as events → `Observable<Intent>`.
- **Action** (optional layer)
    - Internal representation of Intent, used by ViewModel/Interactor.
- **Reducer**
    - Pure function: `(State, Action) -> State`.
    - Decides how State changes.
- **State**
    - Immutable snapshot of the UI at a point in time.
    - Exposed as `Observable<State>`.
- **View**
    - Observes State, renders UI.
    - Emits Intents (back to ViewModel).

**Summary:**
- View emits `Intents`.
- ViewModel transforms Intents into `State` with Rx pipelines.
- Reducer ensures predictable, testable state transitions.
- View observes State and renders accordingly.
## Example
``` kotlin
// Intents
sealed class UserIntent {
    object LoadUser : UserIntent()
    data class Search(val query: String) : UserIntent()
}

// State
sealed class UserState {
    object Idle : UserState()
    object Loading : UserState()
    data class Data(val user: User) : UserState()
    data class Error(val throwable: Throwable) : UserState()
}
```

``` kotlin
class UserViewModel(private val repo: UserRepository) {

    private val intents = PublishSubject.create<UserIntent>()
    private val states = BehaviorSubject.createDefault<UserState>(UserState.Idle)

    fun processIntents(intents: Observable<UserIntent>) {
        intents.subscribe(this.intents) // forward intents
    }

    fun states(): Observable<UserState> = states.hide()

    init {
        this.intents
            .observeOn(Schedulers.io())
            .flatMap { intent ->
                when (intent) {
                    is UserIntent.LoadUser ->
                        repo.getUser("42")
                            .toObservable<UserState>()
                            .map<UserState> { UserState.Data(it) }
                            .onErrorReturn { UserState.Error(it) }
                            .startWith(UserState.Loading)

                    is UserIntent.Search ->
                        repo.searchUsers(intent.query)
                            .toObservable<UserState>()
                            .map<UserState> { UserState.Data(it) }
                            .onErrorReturn { UserState.Error(it) }
                            .startWith(UserState.Loading)
                }
            }
            .subscribe(states::onNext)
    }
}
```

``` kotlin
// Emit Intents
val loadIntent = Observable.just(UserIntent.LoadUser)
val searchIntent = RxTextView.textChanges(searchBox)
    .map { UserIntent.Search(it.toString()) }

viewModel.processIntents(Observable.merge(loadIntent, searchIntent))

// Observe State
viewModel.states()
    .observeOn(AndroidSchedulers.mainThread())
    .subscribeBy { state ->
        when (state) {
            is UserState.Idle -> showIdle()
            is UserState.Loading -> showLoading()
            is UserState.Data -> showUser(state.user)
            is UserState.Error -> showError(state.throwable)
        }
    }
```

## Why MVI with Rx?

✅ **Single Source of Truth** → one state stream.  
✅ **Predictable UI** → every state is explicit, no hidden transitions.  
✅ **Testable** → Reducer is a pure function.  
✅ **Operator power** → debounce, switchMap, merge for complex flows.  
✅ **Error handling built-in** → state can represent error explicitly.

## Best Practices
- State should be **immutable**.
- Use `BehaviorSubject` for state (latest value always available).
- Don’t emit partial UI changes — always a **full state object**.
- Keep Reducers **pure** (no side-effects).
- Push side-effects (network, DB) into repositories.

# Clean Architecture
- Separates code into **independent layers**:
    1. **Presentation** → UI logic (ViewModels, state, Rx pipelines).
    2. **Domain** → Business logic (UseCases).
    3. **Data** → Repositories, network, DB.
- Rx works perfectly for **async boundaries** between layers.

## Layered Overview
``` txt
View (UI)  ⇄  ViewModel (Presentation)  
ViewModel  ⇄  UseCase (Domain)  
UseCase    ⇄  Repository (Data)  
Repository ⇄  API/DB (External)
```

- View emits events → ViewModel → UseCase → Repository → API/DB.
- Data flows back as Rx streams → transformed into immutable UI state.
- Result: scalable, testable, reactive Android app.
## Example
``` kotlin
interface UserRepository {
    fun getUser(userId: String): Single<User>
}

class UserRepositoryImpl(
    private val api: UserApi,
    private val dao: UserDao
) : UserRepository {
    override fun getUser(userId: String): Single<User> =
        api.getUser(userId)
            .onErrorResumeNext { dao.getUser(userId).toSingle() }
}
```

``` kotlin
class GetUserUseCase(private val repo: UserRepository) {
    operator fun invoke(userId: String): Single<User> =
        repo.getUser(userId)
            .subscribeOn(Schedulers.io())
}
```
Use Cases:
- Contain **business rules only**.
- Return `Single`, `Maybe`, `Flowable`, etc.
- Decoupled from Android framework.

``` kotlin
class UserViewModel(
    private val getUserUseCase: GetUserUseCase
) {
    private val state = BehaviorSubject.create<UserState>()
    fun observeState(): Observable<UserState> = state.hide()

    fun loadUser(userId: String) {
        getUserUseCase(userId)
            .observeOn(AndroidSchedulers.mainThread())
            .doOnSubscribe { state.onNext(UserState.Loading) }
            .subscribe(
                { user -> state.onNext(UserState.Success(user)) },
                { error -> state.onNext(UserState.Error(error)) }
            )
    }
}
```

``` kotlin
userViewModel.observeState()
    .observeOn(AndroidSchedulers.mainThread())
    .subscribeBy { state ->
        when (state) {
            is UserState.Loading -> showLoading()
            is UserState.Success -> showUser(state.user)
            is UserState.Error -> showError(state.throwable)
        }
    }
```

### Best Practices
- Keep **Rx types in Domain layer** (don’t expose `LiveData`).
- Use **Schedulers in UseCases**, not in Repositories (centralized threading).
- Expose **immutable Observables** (`.hide()`) from ViewModels.
- Use **CompositeDisposable** in ViewModel’s `onCleared()`.
- Avoid mixing `Flow` and `Rx` inside one UseCase — pick one per layer and use adapters at boundaries.