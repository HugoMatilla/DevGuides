# Dagger
Extracted from [Dagger Codelab](https://developer.android.com/codelabs/android-dagger)

# Adding Dagger
*app//build.gradle*
```kotlin
apply plugin: 'com.android.application'
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-android-extensions'
apply plugin: 'kotlin-kapt'

...

dependencies {
    ...
    def dagger_version = "2.27"
    implementation "com.google.dagger:dagger:$dagger_version"
    kapt "com.google.dagger:dagger-compiler:$dagger_version"
}
```

# @Inject 
Is Used for 2 things
### 1.- To tell Dagger how to provide instances of this type
* Dagger also knows that UserManager is a dependency

```kotlin
class RegistrationViewModel @Inject constructor(val serManager: UserManager) {
    ...
}
```

### 2.- To inject what dagger provides into fields
`@Inject` annotated fields will be provided by Dagger
```kotlin
  @Inject
  lateinit var registrationViewModel: RegistrationViewModel
```  

> When @Inject is annotated on a class constructor, it's telling Dagger how to provide instances of that class. When it's annotated on a class field, it's telling Dagger that it needs to populate the field with an instance of that type.

# @Component
To tell dagger where to look for `@Inject` in fields.

```kotlin
// Definition of a Dagger component
@Component
interface AppComponent {
  // Classes that can be injected by this Component
  fun inject(activity: RegistrationActivity)
}
```
> A @Component interface gives the information Dagger needs to generate the graph at compile-time. The parameter of the interface methods define what classes request injection.
> 
# @Module and  @Binds 
* To tell dagger how to inject interfaces
  
## @Module 
```kotlin
// Tells Dagger this is a Dagger module
@Module
class StorageModule {

}
```
> Modules are a way to encapsulate how to provide objects in a semantic way. As you can see, we called the class StorageModule to group the logic of providing objects related to storage. If our application expands, we could also include how to provide different implementations of SharedPreferences, for example.


Modules must be added in the AppComponent
```kotlin
@Component(modules = [StorageModule::class])
interface AppComponent {
...
}
```


## @Binds
* Tells Dagger to provide an implementation (`SharedPreferencesStorage`) when a interface (`Storage`) is requested.

```kotlin
// Because of @Binds, StorageModule needs to be an abstract class
@Module
abstract class StorageModule {
    abstract fun provideStorage(storage: SharedPreferencesStorage): Storage
}
```
   
# @BindsInstance 
* For objects constructed outside of the graph (ie. Android Application **Context**)

```kotlin
@Component(modules = [StorageModule::class])
interface AppComponent {

    // Factory to create instances of the AppComponent
    @Component.Factory
    interface Factory {
        // With @BindsInstance, the Context passed in will be available in the graph
        fun create(@BindsInstance context: Context): AppComponent
    }

    fun inject(activity: RegistrationActivity)
}

```
# Creating the graph
Now we need to call the `create` function to actually create the Dagger graph. We do it in the Android Application class.

```kotlin
open class MyApplication : Application() {

  // Instance of the AppComponent that will be used by all the Activities in the project
  val appComponent: AppComponent by lazy {
    // Creates an instance of AppComponent using its Factory constructor
    // We pass the applicationContext that will be used as Context in the graph
    DaggerAppComponent.factory().create(applicationContext)
  }
  ...
}

```

And we need to tell Dagger to use the AppComponent to inject objects in our fields.

```kotlin
class RegistrationActivity : AppCompatActivity() {

  @Inject
  lateinit var registrationViewModel: RegistrationViewModel

  override fun onCreate(savedInstanceState: Bundle?) {

    // Ask Dagger to inject our dependencies
    (application as MyApplication).appComponent.inject(this)
    ...
  }
  fun inject(activity: RegistrationActivity)
}
```

> Important: When using Activities, inject Dagger in the Activity's onCreate method before calling super.onCreate to avoid issues with fragment restoration. In super.onCreate, an Activity during the restore phase will attach fragments that might want to access activity bindings.

> Important: Dagger-injected fields cannot be private. They need to have at least package-private visibility.

# @Singleton
* The only scope annotation. 
* Add it to the AppComponent
* And to all classes you want to be singleton for the Apps lifecycle

```kotlin
@Singleton
@Component(modules = [StorageModule::class])
interface AppComponent {
  ...
}
```

```kotlin
@Singleton
class UserManager @Inject constructor(private val storage: Storage) {
  ...
}
```