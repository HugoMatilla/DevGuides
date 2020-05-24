# Core Concepts
## Launching and Builders
1. Add configuration to the coroutines Builder
2. Launch the coroutine 

```kotlin
 GlobalScope.launch {
    getDataFromNetwork() { data -> println(data)}
  }
```

## The coroutines `launch` builder

```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT, //
    block: suspend CoroutineScope.() -> Unit 
): Job {
```

* `CoroutineScope`: a scope for new coroutines. Every coroutine builder (launch, async) is an extension on `CoroutineScope` and inherits its `coroutineContext` to automatically propagate both context elements and cancellation. 
* `context`:  Extra configurations  (Exception Handling, LifeCycle, Threading)
* `start`:  The code wrapped in the coroutine. Is susepend so it can nest other coroutines
* `Job`: A coroutine represented as an object that can be cancelled or started
 
### Job methods and properties
1. `start()`/`cancel()`
2. `isActive`/`isCancelled`
3. `join()`/`children`

* `isActive `: true if it was already started and has not completed nor was cancelled yet.

```kotlin
while (job.isActive){}
```
Will run until the coroutine is completed.

```kotlin
 GlobalScope.launch
```
 will lunch the coroutine inmediately

```kotlin
 val job = GlobalScope.launch(start= CoroutineStart.LAZY)
```
 will wait until `job.start()` is called

## CoroutineContext
A `Set` of elements `CoroutineContext.Element`

There are 3 types
  * Single Purpouse
  * Unique
  * Combined Use

And 3 Elements
* `ContinuationInterceptor`
* `ExceptionHandler`
* `Job`

### ContinuationInterceptor (Controls Threading)
Founded at the `Dispatchers` class
* `Default`
* `IO`
* `Unconfined`
* `Main`

```kotlin
 GlobalScope.launch(context= Dispatchers.IO)
```

### ExceptionHandler (Controls  Exception Handling)
```kotlin
 val exceptionHandler = CoroutineExceptionHandler { _, throwable ->
    throwable.printStackTrace()
    ...
  }
  val job = GlobalScope.launch(context = Dispatchers.Default + exceptionHandler) {...}
```
### Job (Controls LifeCycle)
LifeCycle States:
1. New
2. Active
3. Completing 
4. Completed
5. Cancelling
6. Cancelled

```kotlin
New-[start]->Active-[wait children]-> Completing -[finish]->Completed
                 \                     /
                   \                 /
                     \             /
                     [cancel / fail] 
                            |
                            V
                        Cancelling-[finish]->Cancelled
```

```kotlin
val parentJob = Job()
val childJob = GlobalScope.launch(context= Dispatchers.IO + parentJob){...}
parentJob.cancel()
```
The `childJob` will not execute because we cancelled the `parentJob` first.

#### Job Operations
* Start
* Join
* Cancel

#### Lyfecycle States
* Active
* Finished
* Cancelled

#### Start types
* Lazy
* Eager
  
## Canceling Jobs
* Jobs cancellation propagates to its children.
* To receive the message the children job must be able to receive it.
* It can happend that the thread is occuppied and therefore can not receive the cancellation message.

### Making code cooperative
If there are expensive operations in the async code, checking `isActive` will guarante inmediate response of parent messages.
```kotlin
val parentJob = Job()

val childJob = GlobalScope.launch(context= Dispatchers.IO + parentJob){
  fun expensiveOperation(){
    for (i in 1..100_000_000){
      if (isActive) println(i) // Cooperation
      else break
    }
  }
  expensiveOperation()
}
childJob.start()
parentJob.cancel()
```

## Combining Jobs (Join)
Wait for another job to finish to continue with the current job is done by `join`ing them.

```kotlin
    val job1 = GlobalScope.launch(start = CoroutineStart.LAZY) {
        println("Launching Job1")
        println("Finished Job1")
    }

    val job2 = GlobalScope.launch {
        println("Launching Job2")
        job1.join()
        println("Finished Job2")
    }

    Thread.sleep(500) // to give time to Job 2 to initialize
    job1.start()
```
```
> Launching Job2
> Launching Job1
> Finished Job1
> Finished Job2
```

# Advanced Concepts
## Extending `CoroutineScope` Interface
By  extending `CoroutineScope` interface we can define the scope used in a class without doing it in the call of the coroutine builder.
```kotlin
class MoviesPresenterImpl() : CoroutineScope {
  
  override val coroutineContext: CoroutineContext
      get() = Dispatchers.Main

  fun getMovies() { 
    launch { // Will use the one defined in `coroutineContext` -> Dispatchers.Main
      ...
    }
  }
}
```

## `withContext`  Work in serie
Used to change the context of the calling coroutine.
> Calls the specified suspending block with a given coroutine context, suspends until it completes, and returns the result.
 ```kotlin
public suspend fun <T> withContext(
    context: CoroutineContext,
    block: suspend CoroutineScope.() -> T
): T
```

```kotlin 
suspend fun getDataInADifferntContextComingFromACoroutine(): Result<T> = 
  withContext(Dispatchers.IO) {
    longFunctionExecutedinIO()
  }
```

Needs the `suspend` keyword

## `async` Work in parallel
Allows to defer the process to computing a value  in a separate thread, without blocking and allowing to work in parallel.
withContext(Dispatchers.IO){
  val cachedMoviesDeferred = async { movieDao.getSavedMovies() }
  val responseDeferred = async { movieApiService.getMovies().execute() }
  val cachedMovies = cachedMoviesDeferred.await()
  val response = responseDeferred.await()
}

# Android
## Connect to the Android Lyfecycle
Connect the corutine scope to one parent Job
```kotlin
    var parentJob = Job()
    override fun start() {
        if (!parentJob.isActive)
            parentJob = Job()
    }

    override fun stop() {
        parentJob.cancel()
    }

    override val coroutineContext: CoroutineContext
        get() = Dispatchers.Main + parentJob
``` 
### SuperVisorJob
To avoid the recreation of the parent Job

A failure or cancellation of a child does not cause the supervisor job to fail and does not affect its other children, so a supervisor can implement a custom policy for handling failures of its children:
```kotlin
    private val parentJob = SupervisorJob()
    // no need of start() function
    override fun stop() {
        parentJob.cancelChildren()
    }
```

# Handling Errors
1. Using `Try/Catch`. Not extensible
2. `ExceptionHandler`. Cancel coroutines. We might not want it.
3. Combine `Try/Catch` and `ExceptionHandler` using kotlins `Result` class
## Using Result class with `runCatching`
`runCatching` will wrapp the result into a try catch block and return the value or a Failure if the block throws a exception.
```kotlin
val result = runCatching { movieRepository.getMovies() }
result.onSuccess { movies -> moviesView.showMovies(movies) }
    .onFailure { error -> handleError(error) }
```

# Testing Coroutines
```gradle
testImplementation 'com.nhaarman.mockitokotlin2:mockito-kotlin:2.2.0'
testImplementation 'org.jetbrains.kotlinx:kotlinx-coroutines-test:1.3.3'
testImplementation 'org.powermock:powermock-module-junit4:2.0.2'
testImplementation 'org.powermock:powermock-api-mockito2:2.0.2'
```
* val testDispatcher = TestCoroutineDispatcher() : To create our Dispatcher
* Dispatchers.setMain(testDispatcher): There is no Ancroid so no Main Thread
* TestCoroutineScope(testDispatcher) : To set The main Dispatcher and be able to run blocking
* advanceTimeBy(500): To skip delays etc.
* PowerMock and MockitoKotlin: To help creating Kotlin and Android Mocks

```kotlin
@RunWith(PowerMockRunner::class)
@PrepareForTest(Log::class)
class MoviesPresenterImplTest {

    private val testDispatcher = TestCoroutineDispatcher()
    private val testCoroutineScope = TestCoroutineScope(testDispatcher)
    private val repo = mock<MovieRepository>()
    private val view = mock<MoviesView>()
    private val presenter by lazy { MoviesPresenterImpl(repo) }

    @Before
    fun setUp() {
        PowerMockito.mockStatic(Log::class.java)
        Dispatchers.setMain(testDispatcher)
        presenter.setView(view)
    }

    @Test
    fun `get Data And Show Results`() = testCoroutineScope.runBlockingTest {
        whenever(repo.getMovies()).thenReturn(listOf())
        presenter.getData()
        advanceTimeBy(500)
        verify(repo).getMovies()
        verify(view).showMovies(any())
    }
}
```
# Third party integration
## Use with Retrofit
Add `suspend` and return the type of data you expect.
`suspend fun getMovies(@Query("api_key") apiKey: String): MoviesResponse`

Surround the call in a try/catch to handle Errors
```kotlin
val apiMovies = try {
  movieApiService.getMovies().movies
} catch (e: Throwable) {
  //handleError
}
```

## Use with ROOM
```
implementation "androidx.room:room-ktx:$room_version"
```
Same as Retrofit add `suspend` to the functions
```kotlin
@Insert(onConflict = OnConflictStrategy.REPLACE)
suspend fun saveMovies(movies: List<Movie>)

@Query("SELECT * FROM movies")
suspend fun getSavedMovies(): List<Movie>
```
## With ViewModels
* It has automatic cancellation.
* Call `launch` in the `viewModelScope`.
* Add the exception handler to the call.

```kotlin
viewModelScope.launch(exceptionHandler){
...
}
```