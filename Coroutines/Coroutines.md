# How works
## What happens when calling `launch`
When you call `launch`, you are **not creating a new OS thread**. Instead, you’re creating a small **coroutine object** (like a task) that the Kotlin coroutine library manages.
## Suspending instead of blocking
- **Thread sleep** → blocks the OS thread. That thread is unusable until it wakes up.
- **`delay()`** → suspends the coroutine. The OS thread is released back to the pool, free to run other coroutines. Later, when the delay is over, the coroutine’s saved state (stack frame, continuation) is resumed on a thread again.
## General mechanism
At a suspension point:
1. Coroutine calls a suspending function.
2. That suspending function checks: “Can I return a result immediately?”
    - If **yes** → it just returns and doesn’t suspend.
    - If **no** (like waiting for delay, I/O, or async result) → it calls `suspendCoroutine { cont -> ... }` under the hood.
3. Inside `suspendCoroutine`, the library stores `cont` (the continuation) somewhere (e.g., in a timer, a network callback).
4. Coroutine returns to dispatcher, freeing its thread.
5. Later, when the event happens, the stored `cont.resume(value)` is called → the state machine jumps back in where it left off.

**Example:**
Retrofit integrates with coroutines by wrapping its **callback-based API** into a **suspending function** by using `suspendCoroutine {}` (or more modern `suspendCancellableCoroutine {}`)

![[coroutine_flow.png]]

## Under the hood
``` kotlin
suspend fun loginUser(userId: String, password: String): User {  
	val user = userRemoteDataSource.logUserIn(userId, password)  
	val userDb = userLocalDataSource.logUserIn(user)  
	return userDb  
}

// UserRemoteDataSource.kt  
suspend fun logUserIn(userId: String, password: String): User

// UserLocalDataSource.kt  
suspend fun logUserIn(userId: String): UserDb
```
👉 ___You can think of a suspend function as a regular function which can suspend and then resume at specific point.___
👉 ___But under the hood, the Kotlin compiler will convert suspend functions to an optimized version of callbacks using a finite state machine.___

Suspend functions communicate with each other by `Continuation` objects.
``` kotlin
interface Continuation<in T> {  
	public val context: CoroutineContext  
	public fun resumeWith(value: Result<T>)  
}
```

1. The compiler will replace the suspend modifier with the extra parameter completion (of type `Continuation`)
``` kotlin
fun loginUser(userId: String, password: String, completion: Continuation<Any?>) {  
	val user = userRemoteDataSource.logUserIn(userId, password)  
	val userDb = userLocalDataSource.logUserIn(user)  
	completion.resume(userDb)
}
```

2. The compiler will represent every suspension point as a state in the finite **state machine**
``` kotlin
fun loginUser(userId: String, password: String, completion: Continuation<Any?>) {  
	when(label) {  
		0 -> { // Label 0 -> first execution  
			userRemoteDataSource.logUserIn(userId, password)  
		}  
		1 -> { // Label 1 -> resumes from userRemoteDataSource  
			userLocalDataSource.logUserIn(user)  
		}  
		2 -> { // Label 2 -> resumes from userLocalDataSource
			completion.resume(userDb)  
		}  
		else -> throw IllegalStateException(...)
	}  
}
```

3. The compiler will use the same Continuation object in the function to share information
``` kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {  
	class LoginUserStateMachine(  
    // completion is the callback for loginUser
    completion: Continuation<Any?>  
  ): CoroutineImpl(completion) {    
	// Local variables(results)
    var user: User? = null  
    var userDb: UserDb? = null   
     
    // Common objects for all CoroutineImpls  
    var result: Any? = null  
    var label: Int = 0    
    
    /* 
    ** this function calls the loginUser again to trigger the  
    ** state machine (label will be already in the next state) and  
    ** result will be the result of the previous state's computation  
    */
    override fun invokeSuspend(result: Any?) {  
      this.result = result  
      loginUser(null, null, this) 
    }  
  }  
  ...  
}
```

>[!NOTE]
>The method will be called multiple times to trigger state machine then check label (suspension point)

4. Flow when calling a method:
	1. Unless `continuation` object is not created, then create an instance
	2. Check label
	3. Update label and do task (data are get from `continuation`)
	4. Re-call method until `resume`
``` kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {   
  ...  
  val continuation = completion as? LoginUserStateMachine ?: LoginUserStateMachine(completion)  
  
	when(continuation.label) {  
	    0 -> {  
	        throwOnFailure(continuation.result)  
	        //Update label 
	        continuation.label = 1  
	        userRemoteDataSource.logUserIn(userId!!, password!!, continuation)  
	    }  
	    1 -> {  
	        throwOnFailure(continuation.result)  
	        // Gets the result of the previous state  
	        continuation.user = continuation.result as User   
	        continuation.label = 2  
	        userLocalDataSource.logUserIn(continuation.user, continuation)  
	    }  
	    2 -> {  
	        throwOnFailure(continuation.result)  
	        continuation.userDb = continuation.result as UserDb  
	        // Resumes the execution of the function that called this one  
	        continuation.cont.resume(continuation.userDb)  
	    }  
	    else -> throw IllegalStateException(...)  
	}
}
```
# Fundamentals
## `CoroutineScope`
A `CoroutineScope` keeps track of any coroutine you create using `launch` or `async`. They can be canceled by calling `scope.cancel()` at any point in time.

>[!NOTE] 
>When creating coroutine by helper (ex: `launch`, `async`), it still runs on main thread until you switch to other threads by dispatchers (**main safety**)

>[!NOTE]
>Coroutine scope doesn't create new coroutine, it manages coroutines

## `CoroutineContext`
The `CoroutineContext` is a set of elements that define the behavior of a coroutine. Including:
- `Job`  controls the lifecycle of the coroutine.
- `CoroutineDispatcher:` dispatches work to the appropriate thread.
- `CoroutineName:` name of the coroutine, useful for debugging.
- `CoroutineExceptionHandler:` handles uncaught exceptions, [[will be covered in Part 3 of the series.]]

>[!NOTE] 
>- `CoroutineContext` can be combined using the `+` operator
>- Elements on the right side of the `+` will override those on the left, example:
>	`(Dispatchers.Main, “name”) + (Dispatchers.IO) = (Dispatchers.IO, “name”)`

## Job lifecycle
We can access properties of a Job: `isActive`, `isCancelled` and `isCompleted`.
![[job_lifecycle.webp]]

## Nested `CoroutineContext`
___New coroutine context___ = Defaults + inherited `CoroutineContext` + arguments
___Inner coroutine context___ = parent `CoroutineContext` + `Job()`

# Callback-based to Coroutine-based
## `suspendCoroutine`
``` kotlin
public suspend inline fun <T> suspendCoroutine(
    crossinline block: (Continuation<T>) -> Unit
): T
```
- You call `suspendCoroutine { cont -> ... }` inside a `suspend` function.
- Kotlin immediately suspends the coroutine and passes you a `Continuation<T>`.
- You hold onto `cont` and later call:
    - `cont.resume(value)` → success
    - `cont.resumeWithException(error)` → failure

**Example:**
``` kotlin
fun getUserAsync(callback: (Result<User>) -> Unit)

suspend fun getUser(): User = suspendCoroutine { cont ->
    getUserAsync { result ->
        if (result.isSuccess) {
            cont.resume(result.getOrThrow())
        } else {
            cont.resumeWithException(result.exceptionOrNull()!!)
        }
    }
}

launch {
    val user = getUser()   // suspends here
    println(user)          // resumes when callback calls cont.resume()
}
```

## `suspendCancellableCoroutine`

``` kotlin
public suspend inline fun <T> suspendCancellableCoroutine(
    crossinline block: (CancellableContinuation<T>) -> Unit
): T
```
This is a more powerful variant:
- Instead of a plain `Continuation<T>`, you get a `CancellableContinuation<T>`.
- It allows the coroutine to be **cancelled cooperatively** while suspended.
- You can:
    - Call `cont.invokeOnCancellation { ... }` to clean up (e.g., cancel a network request).
    - Still resume with `cont.resume(value)` or `cont.resumeWithException(error)`.

**Example:**
``` kotlin
suspend fun Call<User>.await(): User =
    suspendCancellableCoroutine { cont ->
        enqueue(object : Callback<User> {
            override fun onResponse(call: Call<User>, response: Response<User>) {
                cont.resume(response.body()!!)
            }

            override fun onFailure(call: Call<User>, t: Throwable) {
                cont.resumeWithException(t)
            }
        })

        // Handle coroutine cancellation
        cont.invokeOnCancellation {
            cancel() // cancels the HTTP request if coroutine is cancelled
        }
    }
```

# Best practices
## Inject dispatchers
Don't hardcode `Dispatchers` when creating new coroutines or calling `withContext`.

**Avoid**
``` kotlin
// DO NOT hardcode Dispatchers
class NewsRepository {
 // DO NOT use Dispatchers.Default directly, inject it instead
 suspend fun loadNews() = withContext(Dispatchers.Default) { /* ... */ }
}
```

**Use**
``` kotlin
class NewsRepository(
    private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default
) {
    suspend fun loadNews() = withContext(defaultDispatcher) { /* ... */ }
}
```

## Safely call suspend function from main thread
For main safety, you should use `withContext` to switch thread for long running task in suspend functions

## Data layers should expose suspend functions or flows
- **Suspend functions for one-shot calls**
- **Flow to notify about data changes**.
``` kotlin
class ExampleRepository {
	suspend fun makeNetworkRequest() { /* ... */ }

	fun getExamples(): Flow<Example> { /* ... */ }
}
```

## Decision when creating coroutines
If the work must be done in current screen, call it in caller's lifecycle such as: `viewModelScope` (in most case), `coroutineScope`, or `supervisorScope`
``` kotlin
class GetAllBooksAndAuthorsUseCase(
    private val booksRepository: BooksRepository,
) {
    suspend fun getBooks(): List<Books> {
        return coroutineScope {
            val books = async { booksRepository.getAllBooks() }
            books.await()
        }
    }
}
```

If the must must be run as long as the app is opened, use external scope
``` kotlin
class ArticlesRepository(
    private val articlesDataSource: ArticlesDataSource,
    private val externalScope: CoroutineScope,
) {
    suspend fun bookmarkArticle(article: Article) {
        externalScope.launch { articlesDataSource.bookmarkArticle(article) }
            .join() // Wait for the coroutine to complete
    }
}
```

## Avoid `GlobalScope`
Use external scope or inject dispatchers instead
**Avoid**
``` kotlin
class ArticlesRepository(
    private val articlesDataSource: ArticlesDataSource,
) {
    suspend fun bookmarkArticle(article: Article) {
        GlobalScope.launch {
            articlesDataSource.bookmarkArticle(article)
        }
            .join()
    }
```

**Use**
``` kotlin
class ArticlesRepository(
    private val articlesDataSource: ArticlesDataSource,
    private val externalScope: CoroutineScope = GlobalScope,
    private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default
) {
    suspend fun bookmarkArticle(article: Article) {
        externalScope.launch(defaultDispatcher) {
            articlesDataSource.bookmarkArticle(article)
        }
            .join()
    }
}
```

## Don't catch `CancellationException`
You can catch every exception but do not catch `CancellationException` to enable coroutine cancellation (always rethrow them if caught)