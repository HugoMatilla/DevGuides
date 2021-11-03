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