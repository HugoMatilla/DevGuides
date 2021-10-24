<!-- TOC -->

- [Core Concepts](#core-concepts)
  - [Launching and Builders](#launching-and-builders)
  - [The coroutines `launch` builder](#the-coroutines-launch-builder)
  - [CoroutineContext](#coroutinecontext)
  - [Canceling Jobs](#canceling-jobs)
  - [Combining Jobs (Join)](#combining-jobs-join)
- [Advanced Concepts](#advanced-concepts)
  - [Extending `CoroutineScope` Interface](#extending-coroutinescope-interface)
  - [`withContext`  Work in serie](#withcontext-work-in-serie)
  - [`async` Work in parallel](#async-work-in-parallel)
- [Android](#android)
  - [Connect to the Android Lyfecycle](#connect-to-the-android-lyfecycle)
- [Handling Errors](#handling-errors)
  - [Using Result class with `runCatching`](#using-result-class-with-runcatching)
- [Testing Coroutines](#testing-coroutines)
- [Third party integration](#third-party-integration)
  - [Use with Retrofit](#use-with-retrofit)
  - [Use with ROOM](#use-with-room)
  - [With ViewModels](#with-viewmodels)
  - [Official Docs about Coroutines and LiveData](#official-docs-about-coroutines-and-livedata)

<!-- /TOC -->

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
    start: CoroutineStart = CoroutineStart.DEFAULT, 
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
class ItemsPresenterImpl() : CoroutineScope {
  
  override val coroutineContext: CoroutineContext
      get() = Dispatchers.Main

  fun getItems() { 
    launch { // Will use the one defined in `coroutineContext` -> Dispatchers.Main
      ...
    }
  }
}
```

## `withContext`  Work in serie
* Used to change the context of the calling coroutine.
* Calls the specified suspending block with a given coroutine context, suspends until it completes, and returns the result.
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
* Allows to defer the process to computing a value  in a separate thread, without blocking and allowing to work in parallel.
* `async {}` is like `launch {}`, but returns an instance of `Deferred<T>`.
* `Deferred<T>` has an `await()` function that returns the result of the coroutine. 
* As soon as `async {}` is called the block runs.
* `await()` will retreive its data once the block ends
```kotlin
withContext(Dispatchers.IO){
    val cachedItemsDeferred = async { itemDao.getItems() } // itemDao.getItems() started
    val responseDeferred = async { itemsCloudService.getItems() } // itemsCloudService.getItems() started
                                      
    //itemDao.getItems() and itemsCloudService.getItems() run in parallel
    
    val Items = ItemsDeferred.await() // Execution is blocked until itemDao.getItems() finishes. 
                                                    // Items will then have the result of the block
    val response = responseDeferred.await() // After previous line finished (Items gets its data),
                                            // responseDeferred.await() is executed.
                                            // Response get its data once itemsCloudService.getItems() finishes.                                           
}
```

# Android

## Connect to the Android Lyfecycle
Connect the corutine scope to one parent Job
```kotlin
    var parentJob = Job()
    override fun onStart() {
      super.onStart()
        if (!parentJob.isActive)
            parentJob = Job()§
    }

    override fun onStop() {
        parentJob.cancel()
        super.onStop()
    }

    override val coroutineContext: CoroutineContext
        get() = Dispatchers.Main + parentJob
``` 

### SuperVisorJob
To avoid the recreation of the parent Job

A failure or cancellation of a child does not cause the supervisor job to fail and does not affect its other children, so a supervisor can implement a custom policy for handling failures of its children:
```kotlin
    private val parentJob = SupervisorJob()
    // no need of onStart() function
    override fun onStop() {
        parentJob.cancelChildren()
        super.onStop()
    }
```

# Handling Errors
1. Using `Try/Catch`. Not extensible
2. `ExceptionHandler`. Cancel coroutines. We might not want it.
3. Combine `Try/Catch` and `ExceptionHandler` using kotlins `Result` class

## Using Result class with `runCatching`
`runCatching` will wrapp the result into a try catch block and return the value or a Failure if the block throws a exception.
```kotlin
val result = runCatching { itemsRepository.getItems() }
result.onSuccess { items -> itemsView.showItems(items) }
      .onFailure { error -> handleError(error) }
```

# Testing Coroutines
```gradle
testImplementation 'com.nhaarman.mockitokotlin2:mockito-kotlin:2.2.0'
testImplementation 'org.jetbrains.kotlinx:kotlinx-coroutines-test:1.3.3'
testImplementation 'org.powermock:powermock-module-junit4:2.0.2'
testImplementation 'org.powermock:powermock-api-mockito2:2.0.2'
```
* `val testDispatcher = TestCoroutineDispatcher()` : To create our Dispatcher
* `Dispatchers.setMain(testDispatcher)`: There is no Ancroid so no Main Thread
* `TestCoroutineScope(testDispatcher)` : To set The main Dispatcher and be able to run blocking
* `advanceTimeBy(500)`: To skip delays etc.
* **PowerMock** and **MockitoKotlin**: To help creating Kotlin and Android Mocks

```kotlin
@RunWith(PowerMockRunner::class)
@PrepareForTest(Log::class)
class ItemsPresenterImplTest {

    private val testDispatcher = TestCoroutineDispatcher()
    private val testCoroutineScope = TestCoroutineScope(testDispatcher)
    private val repo = mock<ItemRepository>()
    private val view = mock<ItemsView>()
    private val presenter by lazy { ItemsPresenterImpl(repo) }

    @Before
    fun setUp() {
        PowerMockito.mockStatic(Log::class.java)
        Dispatchers.setMain(testDispatcher)
        presenter.setView(view)
    }

    @Test
    fun `get Data And Show Results`() = testCoroutineScope.runBlockingTest {
        whenever(repo.Items()).thenReturn(listOf())
        presenter.getData()
        advanceTimeBy(500)
        verify(repo).Items()
        verify(view).Items(any())
    }
}
```

# Third party integration

## Use with Retrofit
Add `suspend` and return the type of data you expect.

```kotlin
suspend fun Items(@Query("api_key") apiKey: String): ItemsResponse
```

Surround the call in a try/catch to handle Errors
```kotlin
val Items = try {
  itemApiService.Items().items
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
suspend fun Items(items: List<Item>)

@Query("SELECT * FROM items")
suspend fun Items(): List<Item>
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

## Official Docs about Coroutines and LiveData
[From official documentation](https://developer.android.com/topic/libraries/architecture/coroutines#livedata)

When using LiveData, you might need to calculate values asynchronously. For example, you might want to retrieve a user's preferences and serve them to your UI. In these cases, you can use the liveData builder function to call a suspend function, serving the result as a LiveData object.

In the example below, loadUser() is a suspend function declared elsewhere. Use the liveData builder function to call loadUser() asynchronously, and then use emit() to emit the result:

```kotlin
val user: LiveData<User> = liveData {
    val data = database.loadUser() // loadUser is a suspend function.
    emit(data)
}
```


* The liveData building block serves as a structured concurrency primitive between coroutines and LiveData. 
* The code block starts executing when LiveData becomes active and is automatically canceled after a configurable timeout when the LiveData becomes inactive.
* It keeps us from restarting our query every time the configuration changes (such as from device rotation).
* If it is canceled before completion, it is restarted if the LiveData becomes active again. 
* If it completed successfully in a previous run, it doesn't restart. Note that it is restarted only if canceled automatically. 
* If the block is canceled for any other reason (e.g. throwing a CancelationException), it is not restarted.

You can also emit multiple values from the block. Each emit() call suspends the execution of the block until the LiveData value is set on the main thread.

```kotlin
val user: LiveData<Result> = liveData {
    emit(Result.loading())
    try {
        emit(Result.success(fetchUser()))
    } catch(ioException: Exception) {
        emit(Result.error(ioException))
    }
}
```
You can also combine liveData with Transformations, as shown in the following example:
```kotlin
class MyViewModel: ViewModel() {
    private val userId: LiveData<String> = MutableLiveData()
    val user = userId.switchMap { id ->
        liveData(context = viewModelScope.coroutineContext + Dispatchers.IO) {
            emit(database.loadUserById(id))
        }
    }
}
```
You can emit multiple values from a LiveData by calling the emitSource() function whenever you want to emit a new value. Note that each call to emit() or emitSource() removes the previously-added source.

```kotlin
class UserDao: Dao {
    @Query("SELECT * FROM User WHERE id = :id")
    fun getUser(id: String): LiveData<User>
}

class MyRepository {
    fun getUser(id: String) = liveData<User> {
        val disposable = emitSource(
            userDao.getUser(id).map {
                Result.loading(it)
            }
        )
        try {
            val user = webservice.fetchUser(id)
            // Stop the previous emission to avoid dispatching the updated user
            // as `loading`.
            disposable.dispose()
            // Update the database.
            userDao.insert(user)
            // Re-establish the emission with success type.
            emitSource(
                userDao.getUser(id).map {
                    Result.success(it)
                }
            )
        } catch(exception: IOException) {
            // Any call to `emit` disposes the previous one automatically so we don't
            // need to dispose it here as we didn't get an updated value.
            emitSource(
                userDao.getUser(id).map {
                    Result.error(exception, it)
                }
            )
        }
    }
}
```