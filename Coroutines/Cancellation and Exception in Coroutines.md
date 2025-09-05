# 1. Cancellation
## 1.1. Calling `cancel`
``` kotlin
val job1 = scope.launch { … }  
val job2 = scope.launch { … }
scope.cancel()
```

👉 ___Cancelling the scope cancels its children___
👉 ___A cancelled child doesn’t affect other siblings___

Under the hood, the child job notifies parent about the cancellation via the exception. The parent uses the **cause** determine whether it needs to handle the exception. If the child was cancelled due to `CancellationException`, then parent do **nothing**

👉 ___Once you cancel a scope, you can't launch new coroutines in the cancelled scope___

## 1.2. Maybe can't cancel properly
If you’re running heavy tasks, like reading from multiple files. It can be run continuously without stopping even if you've already called `cancel`
To fix this, you can use:
- Checking:
	- `isActive`: returns a `Boolean`, expose whether the coroutine is still active
	- `ensureActive()`: checks if coroutine is active. If cancelled, it **throws `CancellationException` immediately**
- Let other work happen using `yield()`
**Best practice:**
- In **app-level code** (loops, background jobs): prefer `isActive` to break cleanly.
- In **library/framework code**: prefer `ensureActive()` to fail fast if someone cancelled the coroutine.

| Use case                                        | `isActive` | `ensureActive()` |
| ----------------------------------------------- | ---------- | ---------------- |
| Polling in loops                                | ✅ yes      | not ideal        |
| “Don’t run if cancelled” (soft check)           | ✅          | ❌                |
| “Stop immediately if cancelled” (hard check)    | ❌          | ✅                |
| Defensive checks in library code (to fail fast) | rare       | ✅                |

## 1.3. Operation after cancelled
### 1.3.1. Use `!isActive`
If you've already checked for `isActive`, you just do side-effect operations outside checking statement:
``` kotlin
while (i < 5 && isActive) {  
    // print a message twice a second  
    if (…) {  
        println(“Hello ${i++}”)  
        nextPrintTime += 500L  
    }  
}
// the coroutine work is completed so we can cleanup  
println(“Clean up!”)
```

### 1.3.2. Use try catch finally
``` kotlin
val job = launch {  
	try {  
		work()  
	} catch (e: CancellationException){  
		println(“Work cancelled!”)  
	} finally {  
		println(“Clean up!”)  
	}
}
```

>[!WARNING]
> If the cleanup task is a suspension, the code above won’t work anymore, because the coroutine is in Cancelling state, it can’t suspend anymore.

To fix this, you need to switch to `NonCancellable` context:
``` kotlin
val job = launch {  
	try {  
		work()  
	} catch (e: CancellationException){  
		println(“Work cancelled!”)  
	} finally {  
		withContext(NonCancellable){  
			delay(1000L) // or some other suspend fun  
			println(“Cleanup done!”)  
		} 
	}
}
```

# 2. Exception
## 2.1. Exception propagation
When a coroutine fails with an exception, it will propagate the exception up to its parent, 
then the parent will:
1. Cancel the rest of its children
2. Cancel itself
3. Propagate the exception up to its parent
The exception will be propagated to the root of hierarchy and cancel all coroutines in scope

## 2.2. `SupervisorJob`
A `SupervisorJob` won’t cancel itself or the rest of its children. `SupervisorJob` won’t propagate the exception either, and will let the child coroutine handle it.
If the exception is not handled, doesn't have a `CoroutineExceptionHandler`, it'll use the default handler, log exception to console then crash your app.
To avoid this you can use `CoroutineScope(SupervisorJob)`:
``` kotlin
val scope = CoroutineScope(SupervisorJob())

scope.launch {  
    // Child 1  
}

scope.launch {  
    // Child 2  
}
```
or `supervisorScope`:
``` kotlin
val scope = CoroutineScope(Job())

scope.launch {  
    supervisorScope {  
        launch {  
            // Child 1  
        }  
        launch {  
            // Child 2  
        }  
    }  
```

**Notice:**
``` kotlin
val scope = CoroutineScope(Job())
scope.launch(SupervisorJob()) {  
	launch { 
        // Child 1  
    }  
    launch {  
        // Child 2  
    }  
}
```
If the `Child 1` is failed then the parent and `Child 2` are still cancelled

>[!NOTE]
>`SupervisorJob` only works when it’s **part of a scope**: either created using `supervisorScope` or `CoroutineScope(SupervisorJob())`. Passing a `SupervisorJob` as a parameter of a coroutine builder (like `scope.launch(SupervisorJob())`) will **NOT** have the desired effect.

## 2.3. Exception handling
You can use `try/catch` or `async/await`
``` kotlin
supervisorScope {  
    val deferred = async {  
        codeThatCanThrowExceptions()  
    } try {  
        deferred.await()  
    } catch(e: Exception) {  
        // Handle exception thrown in async  
    }  
}
```
If you use `Job`, then it will automatically propagate it up in the hierarchy so the `catch` block won’t be called:
``` kotlin
coroutineScope {  
	try {  
		val deferred = async {  
			codeThatCanThrowExceptions()  
		}  
		deferred.await()  
	} catch(e: Exception) {  
		// Exception thrown in async WILL NOT be caught here  
		// but propagated up to the scope
	}  
}
```
> [!NOTE]
> - If you use `async`/`await`, exception only can be caught in `.await()`
> - Only can catch exception in supervisor scope. If you use normal scope, the exception still be throwed because the propagation doesn't have supervisor


## 2.4. `CoroutineExceptionHandler`
- The exception is thrown by a coroutine that _automatically throws exceptions_ (works with `launch`, not with `async`).
- If it’s in the `CoroutineContext` of a `CoroutineScope` or a root coroutine (**DIRECT** child of `CoroutineScope` or a `supervisorScope`).
``` kotlin
val handler = CoroutineExceptionHandler { context, exception -> 
	println("Caught $exception")  
}
```

The exception _will_ be caught by the handler:
``` kotlin
val scope = CoroutineScope(Job())  
scope.launch(handler) {  
	launch {  
		throw Exception("Failed coroutine")  
	}  
}
```

The exception _won't_ be caught by the handler:
``` kotlin
val scope = CoroutineScope(Job())  
scope.launch {  
	launch(handler) {  
		throw Exception("Failed coroutine")  
	}  
}
```
The inner launch will propagate the exception up to the parent as soon as it happens, since the parent doesn’t know anything about the handler, the exception will be thrown, and won't be caught.
