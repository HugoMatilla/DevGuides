# Hilt
Extracted from [Hilt Codelab](https://developer.android.com/codelabs/android-hilt)
[Official Docs](https://developer.android.com/training/dependency-injection/hilt-android)
# Init
Plugin
```kotlin
buildscript {
    ...
    ext.hilt_version = '2.38.1'
    dependencies {
        ...
        classpath "com.google.dagger:hilt-android-gradle-plugin:$hilt_version"
    }
}

```

Use the Plugin
```kotlin
...
apply plugin: 'kotlin-kapt'
apply plugin: 'dagger.hilt.android.plugin'

android {
    ...
}

```

Add the dependencies
```kotlin
...
dependencies {
    ...
    implementation "com.google.dagger:hilt-android:$hilt_version"
    kapt "com.google.dagger:hilt-android-compiler:$hilt_version"
}

```
# @HiltAndroidApp 
* Triggers Hilt's code generation.
* The application container is the parent container for the app.
* Other containers can access the dependencies that it provides.

```kotlin
@HiltAndroidApp
class LogApplication : Application() {
    ...
}
```


# @AndroidEntryPoint 
* Creates a dependencies container that follows the Android class lifecycle.

```kotlin
@AndroidEntryPoint
class LogsFragment : Fragment() {
    ...
}
```
>Hilt currently supports: Application (by using @HiltAndroidApp), Activity, Fragment, View, Service and BroadcastReceiver.

>Hilt only supports activities that extend FragmentActivity (like AppCompatActivity) and fragments that extend the Jetpack library Fragment.


# @Inject
* To perform **field injection**.
* Inject instances of different types 
```kotlin
@AndroidEntryPoint
class LogsFragment : Fragment() {

    @Inject lateinit var logger: LoggerLocalDataSource
    @Inject lateinit var dateFormatter: DateFormatter

    ...
}
```

* Hilt will populate those fields in the `onAttach()` lifecycle method with instances built in the dependencies container that Hilt automatically generated for LogsFragment [More here](https://developer.android.com/training/dependency-injection/hilt-android#generated-components)

* @Inject is also used to create bindings
> The information that tells Hilt how to provide instances of different types are also called bindings.

```kotlin
class DateFormatter @Inject constructor() { ... }
```

# Scoping
`@Singleton` and others from Android components are already built in Hilt [List](https://developer.android.com/training/dependency-injection/hilt-android#component-scopes)

# Modules
* Modules are used to add bindings to Hilt
* To tell Hilt how to provide instances of different types. 
* In Hilt modules, you can include bindings for types that cannot be constructor-injected such as interfaces or classes that are not contained in your project.
  
* `@Module` tells Hilt that this is a module 
* `@InstallIn` tells Hilt the containers where the bindings are available by specifying a Hilt component. 
* A Hilt component is similar to a container. The full list of components can be found [here](https://developer.android.com/training/dependency-injection/hilt-android#generated-components).

```kotlin

@InstallIn(SingletonComponent::class)
@Module
object DatabaseModule {

}
```

# @Provides
* To tell Hilt how to provide types that cannot be constructor injected.
    
>  Kotlin, modules that only contain @Provides functions can be object classes. This way, providers get optimized and almost in-lined in generated code.

```kotlin
@InstallIn(SingletonComponent::class)
@Module
object DatabaseModule {

    @Provides
    fun provideLogDao(database: AppDatabase): LogDao {
        return database.logDao()
    }
}
```

* Each Hilt container comes with a set of default bindings that can be injected as dependencies into your custom bindings. 
* This is the case with `applicationContext`. To access it, you need to annotate the field with `@ApplicationContext`.

```kotlin
@InstallIn(SingletonComponent::class)
@Module
object DatabaseModule {

    @Provides
    fun provideLogDao(database: AppDatabase): LogDao {
        return database.logDao()
    }

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext appContext: Context): AppDatabase {
        return Room.databaseBuilder(
            appContext,
            AppDatabase::class.java,
            "logging.db"
        ).build()
    }
}

```

# @Binds
* To bind interfaces
* `@Binds` must annotate an abstract function (since it's abstract, it doesn't contain any code and the class needs to be abstract too). 
* The return type of the abstract function is the interface we want to provide an implementation for (i.e. `AppNavigator`). 
* The implementation is specified by adding a unique parameter with the interface implementation type (i.e. `AppNavigatorImpl`). `AppNavigatorImpl` has `@Inject constructor` so Hilt know how to construct it.
* Hilt Modules cannot contain both non-static and abstract binding methods, so you cannot place `@Binds` and `@Provides` annotations in the same class.

```kotlin
@InstallIn(ActivityComponent::class)
@Module
abstract class NavigationModule {

  @Binds
  abstract fun bindNavigator(impl: AppNavigatorImpl): AppNavigator
}
```

# Qualifiers
* A qualifier is an annotation used to identify a binding.
## Two implementations for the same interface

*LoggingModule.kt*
```kotlin
@Qualifier
annotation class InMemoryLogger

@Qualifier
annotation class DatabaseLogger

@InstallIn(SingletonComponent::class)
@Module
abstract class LoggingDatabaseModule {

    @DatabaseLogger
    @Singleton
    @Binds
    abstract fun bindDatabaseLogger(impl: LoggerLocalDataSource): LoggerDataSource
}

@InstallIn(ActivityComponent::class)
@Module
abstract class LoggingInMemoryModule {

    @InMemoryLogger
    @ActivityScoped
    @Binds
    abstract fun bindInMemoryLogger(impl: LoggerInMemoryDataSource): LoggerDataSource
}

```

```kotlin
@AndroidEntryPoint
class LogsFragment : Fragment() {

    @InMemoryLogger
    @Inject lateinit var logger: LoggerDataSource
    ...
}

```

```kotlin
@AndroidEntryPoint
class OtherFragment : Fragment() {

    @DatabaseLogger
    @Inject lateinit var logger: LoggerDataSource
    ...
}

```

# Testing
* Testing with Hilt requires no maintenance because Hilt automatically generates a new set of components for each test.

## Dependencies

```kotlin
dependencies {
    // Hilt testing dependency
    androidTestImplementation "com.google.dagger:hilt-android-testing:$hilt_version"
    // Make Hilt generate code in the androidTest folder
    kaptAndroidTest "com.google.dagger:hilt-android-compiler:$hilt_version"
}
```

## CustomTestRunner
```kotlin
class CustomTestRunner : AndroidJUnitRunner() {

    override fun newApplication(cl: ClassLoader?, name: String?, context: Context?): Application {
        return super.newApplication(cl, HiltTestApplication::class.java.name, context)
    }
}
```

Replace the default testInstrumentationRunner content with this:

```kotlin
testInstrumentationRunner content with this:

...
android {
    ...
    defaultConfig {
        ...
        testInstrumentationRunner "com.example.android.hilt.CustomTestRunner"
    }
    ...
}
...

```

## Run the tests
Next, for an emulator test class to use Hilt, it needs to:
* Be annotated with `@HiltAndroidTest` which is responsible for generating the Hilt components for each test
* Use the `HiltAndroidRule` that manages the components' state and is used to perform injection on your test.

```kotlin
@RunWith(AndroidJUnit4::class)
@HiltAndroidTest
class AppTest {

    @get:Rule
    var hiltRule = HiltAndroidRule(this)

    ...
}
```

# @EntryPoint
* Used to inject dependencies in classes not supported by Hilt.
* An entry point is an interface with an accessor method for each binding type we want
* The best practice is adding the new entry point interface inside the class that uses it.
  
```kotlin
class LogsContentProvider : ContentProvider() {

  @InstallIn(SingletonComponent::class)
  @EntryPoint
  interface LogsContentProviderEntryPoint {
    fun logDao(): LogDao
  }

  private fun getLogDao(appContext: Context): LogDao {
    val hiltEntryPoint = EntryPointAccessors.fromApplication(
      appContext,
      LogsContentProviderEntryPoint::class.java
    )
    return hiltEntryPoint.logDao()
  }
  ...
}
```
Notice that the interface is annotated with the `@EntryPoint` and it's installed in the `SingletonComponent` since we want the dependency from an instance of the Application container. Inside the interface, we expose methods for the bindings we want to access, in our case, `LogDao`.

* To access an entry point, use the appropriate static method from EntryPointAccessors. 
* The parameter should be either the component instance or the `@AndroidEntryPoint` object that acts as the component holder.
* Make sure that the component you pass as a parameter and the `EntryPointAccessors` static method both match the Android class in the `@InstallIn` annotation on the `@EntryPoint` interface: