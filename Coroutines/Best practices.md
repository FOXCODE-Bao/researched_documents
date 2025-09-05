# 1. Long-running task
If you don't want to limit the [[Coroutines]] in current screen lifecycle because of `viewModelScope`, just create a scope at `Application` level, then inject it into wherever you want:
``` kotlin
class MyApplication : Application() {  
	val applicationScope = CoroutineScope(SupervisorJob() + otherConfig)  
}
```

``` kotlin
class Repository(  
	private val externalScope: CoroutineScope,  
	private val ioDispatcher: CoroutineDispatcher  
) {  
	suspend fun doWork(): Any {  
		withContext(ioDispatcher) {  
			doSomeOtherWork()  
			return externalScope.async { 
				veryImportantOperation()  
			}.await()
		}  
	}  
}
```

# 2. Inject dispatchers
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

# 3. Safely call suspend function from main thread
For main safety, you should use `withContext` to switch thread for long running task in suspend functions

# 4. Data layers should expose suspend functions or flows
- **Suspend functions for one-shot calls**
- **Flow to notify about data changes**.
``` kotlin
class ExampleRepository {
	suspend fun makeNetworkRequest() { /* ... */ }

	fun getExamples(): Flow<Example> { /* ... */ }
}
```

# 5. Decision when creating coroutines
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

# 6. Avoid `GlobalScope`
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

# 7. Don't catch `CancellationException`
You can catch every exception but do not catch `CancellationException` to enable coroutine cancellation (always rethrow them if caught)