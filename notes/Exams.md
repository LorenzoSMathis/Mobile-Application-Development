# Exam questions

## [2023-09] Android allows certain operations to be done in some threads, and some others not. How and why coroutines help in this sense? Give also an example

Android limit certain operation only into specific kinds of threads, for example update operation on the UI can only be done in the main thread (due to avoid conflicts in GUI render process) or the long running operation (like network requests or particularly heavy computations) must be done on a different thread from the main one, otherwise, if they take more the 3 seconds, they cause the OS to kill the application (that will show the ANR message).

Coroutines are an abstraction level put on top of the thread abstraction by the Kotlin language. A coroutine is based on the continuation programming concept, this means that the operations are modelled as suspendable computation.

A coroutine is the execution of a suspandable function (in Kotlin a suspend fun), this kind of functions are pieces of code that contain one or more suspension point. A suspension point is the point on which the function can be suspended and resumed later on (generally thus points are call to operation that require to wait some result).

Coroutines can help to manage threads in terms of:

1. Composability and cancellability: the threads a generally difficult of cancel and the operation that they will execute is difficult to be split in task and sub-task. The coroutines allows split in subtask and to track the status of the execution (cancellation, normal termination and error handling).

2. The code can be written maintaining a sequential logic, without recurring to complex callback nesting.

3. The threads a build to parallelize the execution, this means that if a routing running on a thread reach a point where it need to wait for some result, the thread will block until it does not receive the result. A coroutine is designed to parallelize the waiting first, when the suspend fun running on a coroutine reach a suspension point, the coroutine goes in paused state and the coroutine scheduler can allocate the thread to another coroutine that does not need to wait. The coroutine scheduling is handled by the Kotlin execution environment, a program can create any number of coroutines, thus coroutines will mapped on the available threads (this allow also to parallelize the actual execution).

4. It's easy to choose which thread(s) will execute the coroutine by choosing the Dispatcher

5. It's easy to synchronize the operation with Mutex and Semaphore abstraction provided by the coroutine library.

6. It's easy to handle with the exceptions by installing an exception handler for the fire-and-forget coroutines (launch() ones) or by using the try catch construct for the awaited ones. 

Example: 

```kotlin
suspend fun wait_for_something(Int a) : Int {
    delay(1000)
    return a*a
}

suspend fun do_something() {
    delay(3000)
    println("Completed")
}

fun main() {
    runBlocking {
        val job = launch { do_something() }
        val def = wait_for_something(3)
        val res = def.await()
    }
}
```

## [2023-09] ViewModel, what is its role and how is its lifecycle compared to the other components it interacts with

The ViewModel, together with the View and the Model, is one of the three components of the View-Model-ViewModel pattern. It is designed to orchestrates the interaction between the model (the component that provide the business logic and the application data) and the view (the component that render the graphical interface). The ViewModel is an object designed to store and manage the UI related data in a lifecycle conscious way.

This object is an instance of the `android.lifecycle.ViewModel` class or one of subclasses like `AndroidViewModel` (for Compose, it extends `androidx.lifecycle.viewmodel.compose.ViewModel`). The ViewModel is the natural place where the programmer can hoist the state of the current view, is a good practice design the application in a way in which each view (fragment or screen) relies on its own dedicated view model that interact with the model (that is one object for the entire application). 

The MVVM pattern is the current evolution of the architectural patterns of an Android application (starting from the older Model-View-Controller, throw the Model-View-Prensenter). The ViewModel provides a set of methods to the View allowing the interaction with the current UI state (the one stored inside the view model) and with the application state by forwarding the operation to the model. An simple example of viewmodel can be the following:

```kotlin
class myVM(private val model) : ViewModel() {
    var myLocalState by mutableStateOf<String>("")
        private set

    fun updateLocalState(newValue: String) : Boolean {
        if (newValue.isBlank()) return false
        myLocalState = newValue
        return true
    }

    val myApplicationStateVar = model.appStateVar

    val updateAppStateVar(newValue: Int) {
        val newValueElaborated = (newValue*10).toString()
        model.updateAppStateVar(newValueElaborated)
    }
}
```

The ViewModel can also start coroutines (for example to call model suspend function or to start long time operation on a different thread) by using its ViewModelScope.

It's important to remember that the view model must never reference a view or any class that may hold a reference to an activity level context, this because the view model survive to configuration changes and, maintaining the reference to an object that will be destroyed and recreated, the garbage collector cannot free that memory causing memory leakage. The only exception to this important rule is the reference to the application context, that is provided to the view model manually with a factory or via dependency injection (using Hilt for example). This is not surprising due to the fact that the application has a lifecycle scope bigger then the view model (the application context will live for all the application life).

The lifecycle of the view model starts when the corresponding component is created (for example an Activity or a composable function), for an activity it is created during the first activity onCreate phase and will start its cleaning phase only after the moment on which the application is finished:

| Event | Activity | View Model |
|-------|----------|------------|
| The activity is created | onCreate() | The view model is instantiated |
| | onStart() | |
| | onResume() | |
| The activity is rotated | onPause() | The view model continue its lifecycle |
| | onStop() | | 
| | onDestroy() | |
| The activity is recreated | onCreate() | The view model is still alive | 
| | onStart() | | 
| | onResume() | | 
| The activity finish is execution | finish() | The view model is still alive |
| | onPause() | | 
| | onStop() | | 
| | onDestroy() | |
| The activity is finished | | onCleared() |

A analogous schema can be written if the view model is bound to a fragment or to a composable function.


## [2023-09] Jetpack Compose library

The Jetpack Compose library is a modern approach to build a graphical user interface in android based on a declarative approach (inspired by the JavaScript React Library). The GUI is composed by @Composable functions that emits the GUI elements, thus functions, at low level, are mapped to a set of GPU instruction that will be used to draw the GUI into the canvas (the screen surface) provided by the activity that will host them.

We can summarize the pros (+) and cons (-) in the following points

* (+) Declarative approach: the programmer describe via a set of composable functions the GUI.
* (+) Data driven: the programmer is not responsible to write the logic for mutate the GUI, he defines only how the GUI will react to data changes. The Composable library will tracks internally the application state and its changes and will automatically recompose the GUI.
* (+) Unique codebase: the entire application is written in idiomatic Kotlin (and the Kotlin DSL for Compose).
* (+) Designed to meet the Material UI guidelines.
* (+) Only one tool is required to write the GUI (for example Android Studio, without any additional feature like the XML Editor used for the native approach).
* (-) More complex learning curve.
* (-) Requires programming skills, as consequence of the fact that the GUI is written in Kotlin and there are any graphical tool to build the GUI also designers needs to know how to programm.

The core of Jetpack Compose the the @Composable function, a pure function that does not return any value, it only emit graphical elements (that can be other composable function defined by the programmer or the composable versions of the native android widgets). 

Thus functions need to be pure, this means no side effect can be put in their code. Any side effect must be launched with `LaunchEffect` or inside a callback (like the `onClick()`).

```kotlin
@Composable
fun MyGUIElement(title: String) {
    var state by remember { mutableStateOf(0)}
    
    LaunchedEffect(key1 = state) {
        Lod.d("MyGUI", "State changed: $state")
    }

    Text(text = title)
    Button(
        onClick = { state = state + 1}
    ) {
        Text(text = "Current: $state, press to increment")
    }
} 
```

The state is managed via ad hoc hooks this allow the framework to track and observe the states. The remember block allow the Compose Library to observe the state and trigger recomposition when it value changes. It propagate down in the elements tree, in general it must be hoisted in the common root element of all widgets that use it.

We can see the state as a top to bottom flow the events will come bottom to up to update that state.

The LaunchedEffects execute its code at first recomposition and after any changes of its key (in this case only the state, it may be multiple values or Unit). The onClick of the button will launch the side effect that increments the state by 1.

The Library provides a large amount of components to define both the layout and the elements of the GUI. In addition provides a quite large number of modifier## [2023-06] Describe the ViewModel lifecycle with respect to the activity/fragments it refers to
 to customize the style of the components.

Finally the concept of "composition" and "recomposition". The (re)composition is the process of rendering all the elements that will compose the GUI. 

The first time that a composable function will be invoked, start the process of composition, the framework register all the elements that compose the GUI, build their tree, allocate the memory to store and track their state and render the GUI.

The recomposition is the process of update of this tree, a composable will be updated if one or more of the following conditions are triggered:

- The parameters of the composable function changes. This means that one or more components upper in the tree change their state.

- The internal state changes.

- One or more states stored in the view model and observed by the component change.

The approach used by the library to handle the complexity of the GUI update is limit as more possible the number of elements that will be rerendered (minimize the computational effort), so the an incorrect usage of the states may lead to not trigger this mechanism.

## [2023-07] Dependency injection, how it’s done in Hilt and the lifecycle of its components

The dependency injection is process of bind the actual instances of two classes that are related by the delegation pattern.

This means that one class delegates part of its behavior to another class, that become a dependency for that class. So when the first class is instantiated into an object, also the the second one need to be instantiated and provided to the first one (or one of its existing instances).

We can do this manually (that can be complex) or rely on a dependency injection library like Hilt.

An example can be:

```kotlin

// Dependent class
class MyRepository (
    private val myAPI: MyAPI                    // by constructor
) {
    private lateinit var myDB: MyDatabase       // by property

    ...    

    init {
        myDB = MyDatabaseImpl()
    }
}

// Dependencies

interface MyAPI { ... }
interface MyDatabase { ... }

// Dependencies implementation

class MyAPIImpl : MyAPI { ... }
class MyDatabaseImpl : MyDatabase { ... }

```

Now this behavior can be implemented via hilt: first of all, after have setup all the dependency and the compiler plugins, the application my be an HiltApplication (a custom Application must be created and must be annotated with `@HiltAndroidApp`). 

Any component that have dependency must be annotated with `@AndroidEntryPoint`. The previous example becomes:

```kotlin

@HiltAndroidApp
class MyCustomApp : Application() { ... }

@AndroidEntryPoint
class MyActivity : AppCompactActivity() {
    @Inject 
    private lateinit var repository : MyRepository

    ...
}

class MyRepository @Inject constructor (
    private val myAPI : MyAPI,
    private val myDB : MyDatabase
) { ... }
 
interface MyAPI { ... }
interface MyDatabase { ... }

class MyAPIImpl @Inject constructor () : MyAPI { ... }
class MyDatabaseImpl @Inject constructor () : MyDatabase { ... }
```

In compose we can inject dependency into the ViewModel has done before, the only thing that we must add is the annotation `@HiltViewModel` the the view model class.

The lifecycle of the dependency can be bound to the lifecycle of many things: this bound is defined by the scope of the dependency, it can be Singleton (only one instance), Component (one instance for that component) or View (one instance for that view). In particular, the lifecycle is bound to the component that depends on the dependency.

From the most wide to the most restrictive:

1. Singleton, it is bound to the Application Lifecycle and only one instance will be created.
2. Activity (Retained), it is bound to the Activity Lifecycle and, for the Retained one, also survive to configuration changes.
3. ViewModel it is bound to the ViewModel.
4. Fragment, View and service, it is bound to the corresponding lifecycle.

Excluding the ViewModelScoped and ViewScoped, the instance of the dependency is created during the Component::onCreate and survive until the onDestroy of that component. For the ViewScoped, it is created during the construction the view and destroyed together with the view. An analog process for the ViewModel.

## [2023-07] Animations in Jetpack Compose, the different kinds of and the functions for

Animations are a powerful tool for the designer to increase the User Experience and the appeal of the app. In Jetpack Compose there is a quite large support for the animation. 

First of all we can split the animations in three groups:

- Animations that changes the structure of the screen, this means that the components' hierarchy is modified.

- Animations that changes one or more properties of the components the structure of the screen remains unaltered.

- Animations supporting the navigation.

Starting from the last group, the in the NavHost components (where the navigation graph is defined), the programmer can define the animation for the transition to and from the destination route.

The animations that change the structure of the screen, can be implemented in compose by several composable function and one modifier:

- AnimatedVisibility: this component, relying on a visibility state, allows to show and hide the components in its sub-tree (the components defined in its content)

- Crossfade: this components allows to animate the transition between one block of content to another one by implementing, by default, a fadeIn/fadeOut transition.

- AnimatedContent: this composable allows to define a set of specialized transitions for its content on the basis of the current state (that is not a simple boolean as for animatedVisibility).

The animations that change only the properties can be implemented by the `Animatable` wrapper, this is a function that encapsulate the value of the property animated. An example of usage is the following:

```kotlin
val property by remember { Animatable(0f) }

LaunchedEffect(property) {
    launch { property.animateTo(finalValue, animationSpec = tween(3000)) }
}

MyComposable(property)
```

This is abstract example, is common to assign that property to a Modifier or to a parameter of a canvas. Note that the animation is launched into the LaunchEffect and it is handled by a coroutine.

Compose also provide the `animate*AsState` where `*` is a data type to create an animated state that can be used as the property defined before, the value stored inside is generally piloted by an external state that act as animation trigger. An example of usage is the following:

```kotlin

val state by remember { mutableStateOf(false) }
val property by animateIntAsState(
    if (state) finalValue else initialValue
)

MyComposable(
    prop = property,
    triggerAnimation = {state = !state}
)
```

The `AnimationSpec` object allows to define the enter and exit transition by composing a set of existing transitions to create custom animations, some default instances of animationSpec are provided by the framework.

## [2023-07] The RecyclerView and its abstractions

The RecyclerView is a fundamental component to build lists in a flexible and efficient way in Android. 

This abstraction comes to handle the following problem, let suppose to have a big list of items, maybe complex, maybe containing heavy data like images. If we create a list in the traditional way (like a <Column> with inside a foreach that emits the items) even if we enable scrolling with the related modifier, all the elements will be rendered and every time we scroll or do something to trigger recomposition the list will be rendered again. The RecyclerView comes to us with an abstraction that can render only the elements that can be viewed, instead the whole set.

Here's how the abstractions plays a role in the RecyclerView:

1. The RecyclerView.Adapter, this is the core of the of the abstraction and it acts as a bridge between the data and the actual RecyclerView. The adapter defines methods to:
    - Specify the number of items in the list.
    - Create the ViewHolder object for each item.
    - Bind the data to the ViewHolder in a particular position.

2. The ViewHolder, this class encapsulate the view of each item of the list. It take a reference to the UI elements within the item layout and update them with the data coming from the Adapter.

3. The LayoutManager, this abstraction defines how the items views are positioned inside the RecyclerView. Some common managers are the LinearLayoutManager (lazy columns and lazy rows) and the GridLayoutManager (lazy grids).

The roles of thus components can be abstracted in the following associations: the Adapter manages the data, the ViewHolder updates the UI and the LayoutManager is responsible for the positioning of the items

## [2023-07] Describe the Information Architecture and related principal patterns

The information architecture is the science and the art of organizing and labeling the informational content within a digital product like a mobile application. Its goals are defining:

- the layout of the elements of each screen,
- the navigation flow between the screens.

The information architecture is point of merging between the user expectation and needs, the context where the application is used and the actual content that the application what to handle. 

The basic concept on which the IA is rooted is the fact that the user expect to find a solution to one problem using the application, this solution should be reached with the less effort possible. The user should concentrate its energies (its cognitive load) on solving the problem not on to understand how to use the application to solve the problem. 

A good IA make an application fast to use and intuitive. A complex or slow information flow can lead to lose the user (in term of attention or effective usage) if it is not able to focus on its own tasks.

The six architectural patterns defined by the IA are:

1. Hierarchy: the navigation between the screens is modeled as a tree, the user uses a index page to navigate to the content. This architecture is pretty common into web application, the advantages are maintaining a consistent UX between the various version of the application (web based, mobile, desktop, ...). The drawback are the potential complex navigation flow forcing the user to navigate across many level; in addition, mobile application are generally hosted by small screen devices, thus kinds of navigation cannot be optimal for small screens.   

2. Hub and Spoke: the homepage hosts a set of links that move the user to the particular feature of the app. This kind of architecture is particular useful when the application features are mutual exclusive, the user can reach the wanted content directly (potentially with a single tap). The main drawback of this solution is the difficulty of navigation from a screen to another, the user is forced to go back to the homepage and choose the new branch of the application. The goal of this pattern is to force the user to concentrate on a single task at time (so users which prefer multi-tasking cannot appreciate this architecture). 

3. Dashboard: this solution is useful to provide a quick view of all the contents (an overall summary), the user can analyze and prioritize the content immediately by itself. Then it can navigate to the section with more detailed information. This solution, if well designed, is useful when the amount of data is very big (think about monitoring application or news pages); but it can lead to chaotic and dense screens, specially on small devices.

4. Filtered view: this pattern is useful to present big lists of data, the user can filter by itself the presented information. Note that the filter must be consistent, so can be not good for sparse or unrelated data. The advantages are providing a flexible way to present big amount of data allowing the user to discovery them with a good grade of freedom. The main drawback is related to the fact that the filters can explode and become to heavy or chaotic for the user that will be not able to set them and can be lost.  

5. Tabbed view: the screens are reachable by switching a tab, this view remember the way in which the desktop browsers manages different pages. Generally on the bottom of the screen there is a common navigation bar allowing the user to switch the screen by using its thumb, this menu can be put also on top to maintain coherency with the web applications world. This solution is useful to make the navigation easy to find and understand, the user can reach all the tabs from each of them, allowing multi-tasking. But this navigation bar can become too dense if the number of tabs is high.

6. Nested doll: the screens are nested one into the other, this means that the user navigate from a more generic presentation of the information down to the most detailed one, this mechanism is useful to not overload the user with too many information in a time. The navigation between two detailed information can be complex due to the fact that the user needs to come up and down again, the cross links between pages can be used but can present a too detailed view of the information (the user reach the most level of details without the contextualization provided by the navigation down).

Thus patterns are specific to solve a particular problem, but in real application this problems are quite blurred; the designer may mix thus approaches to build a better user experience. 

For example tabbed view, filtered view and nested doll can be merged to create a good item list screen. The tabbed view can be used to switch between the various types of content, the filtered view can present the items for a particular content (providing a way to aggregate and filter the information), finally the nested doll can be used to move down the user of one level to present the details of the single item.

## [2023-06] What are kotlin coroutines and how do they support cross thread invocations? Make a practical example

Coroutines are an abstraction level put on top of the thread abstraction provided by the OS, they provide a more consistent, scalable and maintainable way to write asynchronous code. The basic concept is the suspendable computation, in other words, it is a computation that can be paused in a certain point (for example in the moment on which an asynchronous request is made), the state is freezed and the resumed later on.

Thus functions in Kotlin are called suspend fun(s) and rely on the concepts of continuation. The idea is to keep the state of the computation inside an object instead in a stack frame, thus object is called continuation, it is stored in the heap (so it is not related to the actual stack of the program) and it is passed as hidden parameter to thus function (this operation is done by the compiler).

The strengths of this approach are:

1. The asynchronous code is written using a sequential logic, it is very similar to the normal code and does not require callbacks (no callback hell).

2. Composability and cancellability, the logic can be easily divided in task and sub-tasks, the coroutine allows to track very easily their state in case of normal termination, failure and cancellation.

3. The approach of the coroutine is to parallelize in primis the waiting, a normal threads, when it reaches a point in which it needs to wait for some result, the thread is blocked and the only way to perform other operations is create another thread. Coroutines, when reach a point where they need to wait for something, the coroutine scheduler will pause the coroutine and allocate the thread to another coroutine ready to perform some job.
<br>The coroutine can create a parallelization of execution on a single thread by substituting the coroutine that need to wait with the ones that are ready to continue. If the threads available are more then one, the coroutines are distributed on thus threads.

The possible states of a coroutine are mainly three:

- Running: the coroutine is running on a thread, it is not waiting for any result and its code is normally executed.

- Paused: the coroutine is stopped waiting for some asynchronous result. Its thread is freed and allocated to other coroutine.

- Runnable: the coroutine is stopped but the asynchronous result is available, so the coroutine is awaiting for a free thread that can run it.

The coroutine is an running instance of a suspending function, the suspending function is the code. Thus functions are mostly written like a normal function but they can only be run inside a coroutine.

A suspend fun is modelled by the compiler with a finite state machine, each state correspond to a suspension point. The suspension point is a point in the code where the function can be suspended, normally thus point are invocations of other suspend function (or the yield function that introduce manually a suspension point). Only in thus points the coroutine can be stopped (for heavy computation the use of yield function is appreciated to allow to stop the coroutine). 

The sprint is the portion of code between two suspension points.

In Kotlin the coroutines can be started with two possible approaches:

- The `launch` that is a fire an forget invocation, this is useful when the function performs some tasks and does not return any result. The launch function will return a Job object that allows to track the state and eventually cancel the coroutine.

- The `async/await` that invokes the function and return a deferred object representing the promise of the result. The `async` is used to invoke the function, the `await` to block the execution until the result is available.

A coroutine runs into a CoroutineScope, that is an interface that encapsulate the CoroutineContext and provides some methods to manage its coroutines. In Android there are four default scopes: ViewModelScope (bound to the viewModel), LifecycleScope (bound to the lifecycle of the component that launch the coroutine), MainScope (bounded to the main thread), GlobalScope (bound to the application lifecycle).

The CoroutineContext defines the runtime behavior of the coroutine, its components are:
- The dispatcher: defines which thread(s) will execute the coroutine, for example the main thread (Dispatchers.Main), a small pool of threads optimized for CPU elaborations (Dispatchers.Default) or a big pool of threads optimized for IO operations (Dispatchers.IO).
- The job: defines the state of the computation, it can be in six states on the basis of the combination of three flags (isActive, isCompleted, isCancelled). The Job allows also to cancel the coroutine execution, if it is a supervisor job the cancellation is propagated only down (children only) otherwise the cancellation will be propagated also to the siblings.
- The exception handler: a custom handler can be installed to handle the exception rising into the coroutine.
- The name of the coroutine, used for debugging purposes.

The synchronization in the coroutines can be performed by using Mutex and Semaphore classes or by defining the maximum number of parallel executions using `limitedParallelism()` on the dispatcher. Mutex can be useful to lock a state, only one coroutine will be able to modify the value. Semaphore can be useful to limit the number of coroutines that can execute a portion of code.

## [2023-06] Write the 5 J.J Garret’s plans of UX

The five plains of the UX defined by Garret are: Strategy, Scope, Structure, Skeleton and Surface. Thus plains provide a level of abstraction of the UX, from a designer point of view we starts from the Strategy (the most abstract one, defining what the application will do) to the Surface (the most concrete one, defining the actual content of the UI). The user will discover thus plains in the reverse order.

1. Strategy: the strategy plain define the high-level goals of the application, what the designer want to provide and what the user expect from the application.

2. Scope: the scope is the translation of the goals defined in the strategy into features (and requirements).

3. Structure: the structure defines the user cases and the user interactions, the designer defines the screens and the navigation between them. Here the information architecture is defined, the designer also defines the conceptual model behind the application, the user interaction and the error handling flows. Also a set of conventions are defined, if they are needed.

4. Skeleton: the designer defines the actual collocation of the element into the screens, the result of this plain is the layout of each screen with its components (in terms of titles, texts, images, lists, ...).

5. Surface: the actual content is put into the layout of each screen. This plain is the first one perceived by the user.

## [2022-07] describe the looper/handler abstraction.

The looper/handler abstraction is composed by three elements:

1. The Looper: it is an elements (in Android it is a thread) that loops on a message queue dispatching the messages out of the queue to the handlers.

2. The Handler: it is the consumer of the message. It receives the message from the looper and elaborate it.

3. The MessageQueue: it is a FIFO queue where the messages are put, they can come from other application, from some component or from the OS.

A message can represent a some event that can trigger some reaction by the handlers interested into it, or it can contains some data that will be elaborated by the destination handler. 

Thus elements are modelled by the HandleThread abstraction in Android, it is a threads that encapsulate the Looper and a thread-safe message queue. 

## [2022-07] explain what room library is with some practical examples.

The Room library is an ORM developed by Google to improve the local persistency in Android, it provides an abstraction level on the native SQLite database integrated in Android. 

The classes and the interface will be annotated with particular annotation that will be elaborated by the room compiler, it will produce all the boilerplate code needed to make the system working.

The components of the Room system are mainly three:

- The `RoomDatabase`, it represent the database (our database will be an abstract class extending the RoomDatabase class) and is annotated with the @Database annotation (the parameters of the annotation will defines the information about the entities handled by this database and the current version)
<br>The database class will provide abstract methods to retrieve the daos of the various entities, thus methods will be implemented by the library. The Database is a thread safe singleton Object

- The `RoomDAO`, the Data Access Object is an interface responsible to provides the methods to access the record of its entity. Its methods are annotated with `@Query` to define methods that gets data or `@Insert`, `@Upsert`, `@Update`, `@Delete` to modify the data. The query method need a parameter where is defined the query that will be executed.

- The `RoomEntity`, the data class representing a table, it is annotated with `@Entity` and the primary key will be annotated with `@PrimaryKey`. The `@Entity` annotation can accept parameters to define the indexes or foreign keys. If a property is marked as `@Ignored`, it will not be mapped in the database.

The Room library support migration to align the databases of the various installations when a database update is deployed

Example:

```kotlin

@Database(entities = [MyEntity::class], version = 1)
abstract class MyDB : RoomDatabase() {
    abstract fun myDao : MyDAO

    companion object {
        private var _instance : MyDatabase? = null

        fun getInstance(ctx: Context) : MyDatabase {
            return _instance :? synchronized(this) {
                instance = instance ?: Room.databaseBuilder(ctx, MyDatabase::class.java, "my_database").build()
                instance
            }
        }
    }
}

@Dao
interface MyDao {
    @Query("SELECT * FROM my_entity")
    fun getAll() : Flow<List<MyEntity>>

    @Insert
    suspend fun insert(entity: MyEntity)

    @Update
    suspend fun update(entity: MyEntity)

    @Delete
    suspend fun delete(entity: MyEntity)
}

@Entity
data class MyEntity (
    @PrimaryKey(autogenerated = true)
    val id: Int,

    val name: String,
    val surname: String
)
```

## [documents] Illustrate the lifecycle of the activity and relate it to the underlying gui

In Android any component of an application is completely driven by the operative system in terms of lifecycle. The Activity is a central component of the application, it manages the creates and manages the UI, collect the user interaction and acquires and releases the resources of the system. In addition, this class must provide a set of methods that will be used by the operative system to manage the activity lifecycle.

Thus methods are:

- `onCreate()`, create the activity but the GUI is not yet populated. The application goes in the Created state.
- `onStart()`, the GUI is created and visible to the user but it is not yet responsive to the user interaction. The application goes in Stated state.
- `onResume()`, the GUI is enabled to respond to the user interactions. The application goes in Resumed state (this state means app normally running).
- `onPause()`, the GUI is partially visible to the user, the interactions are disabled. The application goes in Paused state. The application goes in this state when some external event comes (like an incoming call) that forces the app in the second position of the Activity stack but it is still running (its not clear if the user will move on another activity, i.e. accept the incoming call, or remain on the current activity, i.e. reject the call).
- `onStop()`, the GUI is moved in background. Now on the activity is moved down into the activity stack and is completely hidden to the user. The application goes in the Stopped state.
- `onDestroy()`, the activity is destroyed and pop out from the activity stack and the memory released. If the activity is terminated by the user or by the application (`finish()`) the state is not saved; otherwise, if the system terminated the activity to release resources, the state will be saved in a Bundle reloaded when the activity will be recreated.

Two other methods are used to revert the Paused and Stopped state:

- `onResume()`, the app, if in the Paused state, is moved to Resumed state.
- `onRestart()`, the app is moved from the Stopped state to the Started one.

In terms of resources the key moments are:

- `onCreate` should allocate the resources used by the activity
- `onRestart` should start the animations and the user interaction related operations
- `onPause` should persist the actual state of the application (align local caches with remote ones and write on the database local states).
- `onStop` should release the resources allocated to the activity.

## [documents] Describe service lifecycle, both started and bound

The services are components of the android ecosystem that performs tasks that do not require any graphical user interface or direct user interaction. Thus services are defined into the manifest.

The services are divided in two macro categories:

- Started services: this kind of services are started by an application component and runs in background providing some operations.

- Bound services: tis kind of services exposes a client-server interface used by the application to interact with them.

The started service lifecycle consists of:

1. The activity (or any other component) start a service by creating an Intent: `startService(intent)`.

2. The system will create the service by invoking first the constructor of the class, then the `onCreate` method.

3. The `onStartCommand(intent)` will provide the the started service the command that must be executed, now the service start its operation, even moving some operation on different threads.

4. The `onDestroy` method will be invoked to kill the service and terminate its lifecycle.

The bound service lifecycle consists of:

1. The Activity create a connection (by its constructor) then bind the service with an intent on the created connection (`bindService(intent, connection)`).

2. The service is created by the running environment by invoking its constructor, then the `onCreate` method. The created service is bound to the request by invoking the `onBind(intent)` method providing the intent and receiving the iBinder object. The iBinder is an interface that is used to define how interact with the service hiding the actual complexity of the api.

3. The iBinder object is bound to the connection object by invoking the `onServiceConnected(iBinder)` on the Connection instance.

4. The activity uses the service.

5. The activity unbind the service by invoking the `unbindService(connection)`, the unbind operation is notified to the service.

6. The service will be destroyed when no components are bound to it by invoking the `onDestroy`.

The lifecycle of the services, started or bound, is managed by the os internals, the operation performed by the component are only the request for the service with the `startService` or `bindService/unbindService`.

The background execution is limited by android since the 8.0 version. This is made to avoid security and privacy issues and decrease the load of the system. 

In particular, an application can run in background only for a grace period after the last user visible component terminates its lifecycle, during this period the service should terminate its operations. The service that can operate in background are quite limited (only particular service are exceptions to this rule).

To allow background operations, the service must be promoted to foreground service, the user is aware of their execution because a notification is displayed in the notification bar. A service is considered a foreground service if at least one of the following conditions is satisfied:

1. The service is linked to a visible activity.
2. Has another foreground service.
3. Another components foreground is connected is to it (bound services)

## [documents] How an Activity calls another and life cycle of both

An activity can call another one using the Intents, an intent is a sort of message or signal used to communicate to another component in the Android OS some event or to send some data.

Intents can be implicit or explicit. The first type defines an Action (what operation has been required), the URI (the resource identifier) and a Category (that provides additional information useful to delivery the intent to the correct application). The second one defines the actual Application name that want to use (for example che Activity class that wants to call).

The transition between one activity starts with the invocation of the `startActivity` function that will deliver an intent to the OS that will take the control. The OS, on the basis of the incoming intent, will invoke the `onCreate` method of the target class delivering an intent to it, if the activity is already existing it will move that activity on the top of the stack instead to create it.

Intents can be sent in broadcast, thus intents are used to call all components that are listening to that message. The sending operation is done by the `sendBroadcast`, a component can communicate to the OS that is listening for a message by invoking the `registerReceiver`

## [documents] Describe resources and how they are involved in app responsiveness

The resources can be local or remote, for the local one can be the static resources of the application (put in the folder `res`) or stored in a local SQLite database (or on the file system). The remote ones can be accessed via a network requests (the network request can be handled manually by the programmer dealing with the HTTP protocol and the Java classes used to produce and process requests and responses or by using a library that simplify thus operation like OkHTTP, Retrofit, Cronet or the firebase suite if the remote api are hosted by Firebase).

The critical part of dealing with resources is the fact that thus operation, specially the remote ones, are asynchronous:

The response may require time to come back, if we block the main thread due to an HTTP request, the whole application will freeze. In addition, the Android OS will kill an application that blocks the main thread for more then 3 seconds (ANR error).

So, the operation that will interact with the resources must be done on a secondary thread, avoiding the freezing of the application, that is a big issue for the user experience, and the more critical ANR message (that will stop the application).

The coroutines are a powerful tool provided by the Kotlin language to perform thus kind of operations. 

## [documents] Explain the definition and role of the android manifest file

The Android Manifest is a XML file that represent the entire application in terms of components and main properties. It can be view as a contract between the application and the execution environment.

Some of the information contained inside are:

1. The `<application>` block where is defined the application class (if not specified the OS will allocate the default Application). Additional information can be added, for example the name displayed in the application grid, the icon.

2. Inside the `<application>`, all the activities and services are listed (and any other component), thus components will be instantiated by the OS when the app will started. Components not listed here cannot be used.

3. The permission list, all the permission that the application will requires, some of them are automatically granted (like the internet permission) but other, more critical, will require an explicit consent by the user with the proper alert.

## [documents] Handle screens with different sizes/resolutions

Different screens and sizes can be handled in Jetpack compose by using constrained composable like BoxWithConstraints. Thus components provide in their context the information about the current width and height of the screen.

This information can be exploited to customize the UI by fixing a sort of breakpoints (like the one used by bootstrap).

Example:

```kotlin
BoxWithConstraints {
    if (maxWidth > maxHeight) {
        HorizontalLayout(this)
    }
    else {
        VerticalLayout(this)
    }
}

@Composable
fun HorizontalLayout(
    bs : BoxWithConstraintsScope
) {
    with (bs) {
        if (maxWidth < 200.dp) {
            // small screen
        }
        else if (maxWidth < 500.dp) {
            // medium screen
        }
        else {
            // big screen
        }
    }
}

// Something analogue for the Vertical Layout
```

## [documents] What is React Native, how does it works

React Native is a cross platform mobile application technology, it is a framework allowing the programmer to produce the application for the various operative systems (Android, iOS) with a single codebase; the application is written once then deployed on each system.

The React Native framework relies on the Javascript React framework designed by Facebook providing the tools to produce hybrid applications. 

This approach fall into the Native Widget based family, this means that the framework provides a set of widgets that will be mapped on the native ones (the React Native Button is mapped on the Android Native Button and on the iOS Button depending on the OS selected at deploy time); the advantages of React Native are related on the fact that the application is built on well known and mature technologies like Typescript (JS), but the need of map each widget on the native ones limit many features, in fact only the common set of widgets can be used.

A React Native application is composed by two principal threads, the main thread that execute the native code (the native application) that is responsible for the OS interactions and for the UI creation, and the Javascript thread responsible for the execution of the Javascript code. A interprocess communication mechanism is provided to make possible the communication between the two threads.