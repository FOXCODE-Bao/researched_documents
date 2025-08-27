# Cancellation
## Calling `cancel`
``` kotlin
val job1 = scope.launch { … }  
val job2 = scope.launch { … }
scope.cancel()
```

👉 ___Cancelling the scope cancels its children___
👉 ___A cancelled child doesn’t affect other siblings___

Under the hood, the child job notifies parent about the cancellation via the exception. The parent uses the **cause** determine whether it needs to handle the exception. If the child was cancelled due to `CancellationException`, then parent do **nothing**

👉 ___Once you cancel a scope, you can't launch new coroutines in the cancelled scope___

## Maybe can't cancel properly
If you’re running heavy tasks, like reading from multiple files. It can be run continuously without stopping even if you've already called `cancel`
To fix this, you can use:
- Checking `job.isActive` or `ensureActive()`
- Let other work happen using `yield()`

## Operation after cancelled
### Use `!isActive`
If you've already checked for `isActive`, you just do side-effect operations outside checking statement:
``` kotlin
while (i < 5 && **isActive**) {  
    // print a message twice a second  
    if (…) {  
        println(“Hello ${i++}”)  
        nextPrintTime += 500L  
    }  
}
// the coroutine work is completed so we can cleanup  
println(“Clean up!”)
```

### Use try catch finally
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
