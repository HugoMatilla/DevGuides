# Dagger
Extracted from [Dagger Codelab](https://developer.android.com/codelabs/android-dagger)

# Adding Dagger
*app/build.gradle*
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

--- 

# ADVANCE
> Fragments - Best practices
> An Activity injects Dagger in the onCreate method before calling super.
> A Fragment injects Dagger in the onAttach method after calling super.

## Subcomponents
* Subcomponents are components that inherit and extend the object graph of a parent component. Thus, all objects provided in the parent component will be provided in the subcomponent too.
* In this way, an object from a subcomponent can depend on an object provided by the parent component.
* You need to create the factory of the subcomponent. And then expose it in the parent component.

```kotlin
@Subcomponent
interface RegistrationComponent {
// Factory to create instances of RegistrationComponent
  @Subcomponent.Factory
  interface Factory {
    fun create(): RegistrationComponent
  }

  // Classes that can be injected by this Component
  fun inject(activity: RegistrationActivity)
  fun inject(fragment: EnterDetailsFragment)
  fun inject(fragment: TermsAndConditionsFragment)
}
```

Add the subcomponent to the AppComponent

```kotlin
@Singleton
@Component(modules = [StorageModule::class])
interface AppComponent {
  ...
    // Expose RegistrationComponent factory from the graph
    fun registrationComponent(): RegistrationComponent.Factory
  ...
}

```

Now we need a module to encapsulate the Subcomponents and reference it to the AppComponent

```kotlin
// This module tells AppComponent which are its subcomponents
@Module(subcomponents = [RegistrationComponent::class])
class AppSubcomponents

```

```kotlin
@Singleton
@Component(modules = [StorageModule::class, AppSubcomponents::class])
interface AppComponent { ... }

```

**There are two different ways to interact with the Dagger graph:**
* Declaring a function that returns `Unit` and takes a class as a parameter allows field injection in that class (e.g. `fun inject(activity: MainActivity)`).
* Declaring a function that returns a type allows retrieving types from the graph (e.g. `fun registrationComponent(): RegistrationComponent.Factory`).


## Scoping
**Scoping rules:**
 * The scope annotation's name should not be explicit to the purpose it fulfills. It should be named depending on the lifetime it has since annotations can be reused by sibling Components. (`ActivityScope` instead of `RegistrationScope`)
 * When a type is marked with a scope annotation, it can only be used by Components that are annotated with the same scope.
 *  When a Component is marked with a scope annotation, it can only provide types with that annotation or types that have no annotation.
 * A subcomponent cannot use a scope annotation used by one of its parent Components.

Components also involve subcomponents in this context.


```kotlin
@Scope
@MustBeDocumented
@Retention(value = AnnotationRetention.RUNTIME)
annotation class ActivityScope
```

```kotlin
// Scopes this ViewModel to components that use @ActivityScope
@ActivityScope
class RegistrationViewModel @Inject constructor(val userManager: UserManager) {
    ...
}

```

```kotlin
// Classes annotated with @ActivityScope will have a unique instance in this Component
@ActivityScope
@Subcomponent
interface RegistrationComponent { ... }

```

# @Provides
* To tell Dagger how to provide an instance of a class inside a Dagger module. (Apart from the @Inject and @Binds)
* The return type of the @Provides function (it doesn't matter how it's called) tells Dagger what type is added to the graph. 
* The parameters of that function are the dependencies that Dagger needs to satisfy before providing an instance of that type.

```kotlin
@Module
class StorageModule {

    // @Provides tell Dagger how to create instances of the type that this function 
    // returns (i.e. Storage).
    // Function parameters are the dependencies of this type (i.e. Context).
    @Provides
    fun provideStorage(context: Context): Storage {
        // Whenever Dagger needs to provide an instance of type Storage,
        // this code (the one inside the @Provides method) will be run.
        return SharedPreferencesStorage(context)
    }
}

```

# Qualifiers
* To add different implementations of the same type to the Dagger graph. 

> A qualifier is a custom annotation that will be used to identify a dependency.

```kotlin
@Retention(AnnotationRetention.BINARY)
@Qualifier
annotation class RegistrationStorage

@Retention(AnnotationRetention.BINARY)
@Qualifier
annotation class LoginStorage

@Module
class StorageModule {

    @RegistrationStorage
    @Provides
    fun provideRegistrationStorage(context: Context): Storage {
        return SharedPreferencesStorage("registration", context)
    }

    @LoginStorage
    @Provides
    fun provideLoginStorage(context: Context): Storage {
        return SharedPreferencesStorage("login", context)
    }
}

```

To use them

```kotlin
// In a method
class ClassDependingOnStorage(@RegistrationStorage private val storage: Storage) { ... } 

// As an injected field
class ClassDependingOnStorage {

    @Inject
    @field:RegistrationStorage lateinit var storage: Storage
}

```


## @Named annotation.
* Same functionallity as qualifiers but Qualifiers are prefered because:

* They can be stripped out from Proguard or R8
* You don't need to keep a shared constant for matching the names
* They can be documented
