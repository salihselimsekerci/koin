---
title: Android Scopes
---


## Taming the Android lifecycle

Android components are mainly managed by their lifecycle: we can't directly instantiate an Activity nor a Fragment. The system
make all creation and management for us, and make callbacks on methods: onCreate, onStart...

That's why we can't describe our Activity/Fragment/Service in a Koin module. We need then to inject dependencies into properties and also
respect the lifecycle: Components related to the UI parts must be released on soon as we don't need them anymore.

Then we have:

* long live components (Services, Data Repository ...) - used by several screens, never dropped
* medium live components (user sessions ...) - used by several screens, must be dropped after an amount of time
* short live components (views) - used by only one screen & must be dropped at the end of the screen

Long live components can be easily described as `single` definitions. For medium and short live components we can have several approaches.

In the case of MVP architecture style, the `Presenter` is a short live component to help/support the UI. The presenter must be created each time the screen is showing,
and dropped once the screen is gone.

A new Presenter is created each time

```kotlin
class DetailActivity : AppCompatActivity() {

    // injected Presenter
    override val presenter : Presenter by inject()
```

We can describe it in a module:


* as `factory` - to produce a new instance each time the `by inject()` or `get()` is called

```kotlin
val androidModule = module {

    // Factory instance of Presenter
    factory { Presenter() }
}
```

* as `scope` - to produce an instance tied to a scope

```kotlin
val androidModule = module {

    scope<DetailActivity> {
        scoped { Presenter() }
    }
}
```

:::note
 Most of Android memory leaks comes from referencing a UI/Android component from a non Android component. The system keeps a reference
on it and can't totally drop it via garbage collection.
:::

## Scope for Android Components

Koin provides `ScopeActivity` & `ScopeFragment` classes, to bind to your Android component to its scope. It will close its scope automatically on destroy steps.

To use a scoped Android component, you have to declare a scope for your activity (tied our Activity type):

```kotlin
val androidModule = module {

    scope<MyActivity> {
        scoped { Presenter() }
    }
}
```

And then use the `ScopeActivity` class:

```kotlin
class MyActivity : ScopeActivity() {

    // inject Presenter instance from MyActivity's Koin scope
    val presenter : Presenter by inject()

}
```

:::note
- You can do the same with `ScopeFragment` & `ScopeService` components
- ViewModel extensions are also compatible with `ScopeActivity` & `ScopeFragment`
:::

`ScopeActivity` & `ScopeFragment` are providing the source component: You can inject your Activity or Fragment into the needed component:

```kotlin
class Presenter(val view : MyViewContract)

val androidModule = module {

    scope<MyActivity> {
        // inject current MyActivity
        scoped { Presenter(get()) }
    }
}
```

Here below, you can directly use `KoinScopeComponent` to declare some scope in your Activity/Fragment:

```kotlin
// Implementing our own Scope delegation 
class MVPActivity : AppCompatActivity(R.layout.mvp_activity), KoinScopeComponent {
  
    // Create scope
    override val scope: Scope by lazy { newScope() }

    // Inject presenter with org.koin.core.scope.inject extension
    // also can use directly the scope: scope.inject<>()
    val presenter: ScopedPresenter by inject()
  
    // Don't forget to close it when finish
    override fun onDestroy() {
        super.onDestroy()
        scope.close()
    }
}
```

:::note
 You can use `activityScope()` & `fragmentScope()` to create your own scope:
:::

```kotlin
// Implementing our own Scope delegation 
class MVPActivity : AppCompatActivity(R.layout.mvp_activity), KoinScopeComponent {
  
    // Create Activity scope (backed by ViewModel)
    override val scope: Scope by lazy { activityScope() }

    // Inject presenter with org.koin.core.scope.inject extension
    // also can use directly the scope: scope.inject<>()
    val presenter: ScopedPresenter by inject()
}
```


## Sharing instances between components with custom scopes & Scope Links

In a more extended usage, you can use a `Scope` instance across components. For example, if we need to share a `UserSession` instance.

First declare a scope definition:

```kotlin
module {
    // Shared user session data
    scope(named("session")) {
        scoped { UserSession() }
    }
}
```

When needed to begin use a `UserSession` instance, create a scope for it:

```kotlin
val ourSession = getKoin().createScope("ourSession",named("session"))

// link ourSession scope to current `scope`, from ScopeActivity or ScopeFragment
scope.linkTo(ourSession)
```

Then use it anywhere you need it:

```kotlin
class MyActivity1 : ScopeActivity() {
    
    fun reuseSession(){
        val ourSession = getKoin().createScope("ourSession",named("session"))
        
        // link ourSession scope to current `scope`, from ScopeActivity or ScopeFragment
        scope.linkTo(ourSession)

        // will look at MyActivity1's Scope + ourSession scope to resolve
        val userSession = get<UserSession>()
    }
}
class MyActivity2 : ScopeActivity() {

    fun reuseSession(){
        val ourSession = getKoin().createScope("ourSession",named("session"))
        
        // link ourSession scope to current `scope`, from ScopeActivity or ScopeFragment
        scope.linkTo(ourSession)

        // will look at MyActivity2's Scope + ourSession scope to resolve
        val userSession = get<UserSession>()
    }
}
```
