---
slug: Kotlin coroutines
title: Kotlin coroutines a basic approach
authors: [luiscarbonel]
tags: [kotlin, coroutines, async, suspend]
---


When developing applications we often want our functions to behave asynchronously or non-blocking, to ensure this there are many mechanisms depending on the language you use. In this article we are going to make an introduction of how Kotlin solves this issue in a very elegant way through its coroutines.

Are coroutines  threads ? Ummm, the answer is nooo, coroutines and threads are similar in that they can be executed simultaneously and they need a block of code for their construction. But the truth is that a coroutine is not linked to any particular thread, in fact when we launch a coroutine it suspends its execution in a thread and when it has the result ready it can be resumed in any of the free threads, not necessarily in the thread where it started its execution.

Next we are going to start from a requirement that usually appears when we develop in enterprise environments.

Imagine that you are asked to obtain the information of a Profile with its basic information (UserInfo) and the last 10 web sites where it was (TrackinInfo).

Let's do the first sequential approach.
```kotlin
import kotlinx.coroutines.delay

class UserInfo(name: String, lastName: String) // class to represent a user
class TrackingInfo(webSites: List<String>)  // class to represent a tracking
class Profile(userinfo: UserInfo?, trackingInfo: TrackingInfo?) // class to represent a profile

val users = mapOf(1 to UserInfo(name = "Luis", lastName = "Foo"), 2 to UserInfo(name = "Carlos", lastName = "Bar")) // contains all user in a map[idUser, UserInfo]
val tracking = mapOf(1  to TrackingInfo(listOf("vercel.com", "devlach.com")), 2  to TrackingInfo(listOf("react.com", "devlach.com"))) // contains all tracking in a map[idUser, TrackingInfo]

// Sequential approach
class SequentialApproach {
    fun getProfile(id: Int): Profile {
        val user = findUserInfo(id)
        val trackingInfo = findTrackingInfo(id)
        return Profile(user, trackingInfo)
    }

    private fun findUserInfo(id: Int): UserInfo? {
        Thread.sleep(1500)  // Block the thread for 1500 milliseconds
        return users[id]
    }
    private  fun findTrackingInfo(id: Int): TrackingInfo? {
        Thread.sleep(1500) // Block the thread for 1500 milliseconds
        return tracking[id]
    }
}
```

```kotlin
fun main() {
    val sequentialTime = measureTimeMillis {
        SequentialApproach().getProfile(1).apply { println(this) }
    }
    println("SequentialTime was : $sequentialTime")
}
```
Print:
Profile(userinfo=UserInfo(name=Luis, lastName=Foo), trackingInfo=TrackingInfo(webSites=[vercel.com, devlach.com]))
SequentialTime : 3002

As you can see our program took 3002 milliseconds. But we were not happy with that time and we were able to identify that if we could get the user and tracking information in a concurrent way it would help our program to be faster.

Let's take the first asynchronous approach:

```kotlin
// Asynchronous approach
class ConcurrentApproach {
    suspend fun getProfile(id: Int): Profile = runBlocking {
        val user = async { findUserInfo(id) }
        val trackingInfo = async { findTrackingInfo(id) }
        return@runBlocking Profile(user.await(), trackingInfo.await())
    }

    private suspend fun findUserInfo(id: Int): UserInfo? {
        delay(1500) // Not blocking thread
        return users[id]
    }
    private suspend fun findTrackingInfo(id: Int): TrackingInfo? {
        delay(1500) // Not blocking thread
        return tracking[id]
    }
}
```

Run again:
```kotlin
suspend fun main() {
   val sequentialTime = measureTimeMillis {
        SequentialApproach().getProfile(1).apply { println(this) }
    }
    println("SequentialTime : $sequentialTime")

    val concurrentTime = measureTimeMillis {
        ConcurrentApproach().getProfile(1).apply { println(this) }
    }

    println("ConcurrentTime : $concurrentTime")
}
```
Print:
Profile(userinfo=UserInfo(name=Luis, lastName=Foo), trackingInfo=TrackingInfo(webSites=[vercel.com, devlach.com]))
SequentialTime : 3002

Profile(userinfo=UserInfo(name=Luis, lastName=Foo), trackingInfo=TrackingInfo(webSites=[vercel.com, devlach.com]))
ConcurrentTime : 1549

Now it's better, as we see ConcurrentTime : 1549, but how could we improve our processing time... That's because of the coroutines, let's see what changes we made.

First our business functions:
*findUserInfo*, *findTrackingInfo* and *getProfile* are now marked as **suspend** functions, but this is because instead of using *Thread.sleep* we use *delay* (This function delays coroutine for a given time without blocking a thread and resumes it after a specified time), which being a suspend function then the function that uses it must also be a suspend function.

On the other hand we see something different in getProfile, now we make our functions findUserInfo and findTrackingInfo to be inside launch{....} and with this we tell it to execute concurrently.

```kotlin
val user = async { findUserInfo(id) }
val trackingInfo = async { findTrackingInfo(id) }
```
Finally, to obtain both results we call **await** (Awaits for completion of this value without blocking a thread and resumes when deferred computation is complete).
We can also see the **runBlocking** function which is nothing more than a coroutine builder.

As I'm a bit picky about my code structure and organization, we'll try to improve this example. 

First let's move the functions *findUserinfo* and *finTrackingInfo* to repositories.

```kotlin
class UserRepository {
    suspend fun findUserInfo(id: Int): UserInfo? {
        delay(1500) // Not blocking thread
        return users[id]
    }
}

class TrackingRepository {
    suspend fun findTrackingInfo(id: Int): TrackingInfo? {
        delay(1500) // Not blocking thread
        return tracking[id]
    }
}
```
Now we are going to create a helper class to handle our own scope and in this way to execute any
suspend function with async.

```kotlin
object CoroutineUtils {
    private val defaultScope = CoroutineScope(Dispatchers.IO + CoroutineName("General Purpose")) // Default scope
    suspend fun <T> runAsync( block: suspend () -> T) = defaultScope.async { block() }
}
```

Now we are going to create a service that will have the logic to obtain the profile information.

```kotlin
class ProfileService(private val userRepository: UserRepository, private val trackingRepository: TrackingRepository) {
    
    suspend fun getProfile(id: Int): Profile {
        val userInfo = CoroutineUtils.runAsync { userRepository.findUserInfo(id) }
        val trackingInfo = CoroutineUtils.runAsync { trackingRepository.findTrackingInfo(id) }
        return Profile(userinfo = userInfo.await(), trackingInfo = trackingInfo.await())
    }
}
```

ProfileService receives both repositories via contructor and now we can see our business functions that are called by each one of our repositories
one of our repositories userRepository.findUserInfo(id) and trackingRepository.findTrackingInfo(id).

```kotlin
val userInfo = CoroutineUtils.runAsync { userRepository.findUserInfo(id) }
val trackingInfo = CoroutineUtils.runAsync { trackingRepository.findTrackingInfo(id) }
```
If you look closely now the function getProfile does not have runBlocking and we do not need return@runBlocking and it is because we are handling our own scope.
we are handling our own coroutine scope.

It is time to run our program.

```kotlin
suspend fun main() {
    val sequentialTime = measureTimeMillis {
        SequentialApproach().getProfile(1).apply { println(this) }
    }
    println("SequentialTime : $sequentialTime")

    val concurrentTime = measureTimeMillis {
        ConcurrentApproach().getProfile(1).apply { println(this) }
    }

    println("ConcurrentTime : $concurrentTime")

    val structuredTime = measureTimeMillis {
        // Init instances
        val profileService = ProfileService(UserRepository(), TrackingRepository())
        // Call the service
        profileService.getProfile(id = 1).also { println(it) }
    }
    println("StructuredTime : $structuredTime")
}
```

Print
```kotlin

// Profile@7c0e2abd
// SequentialTime : 3018
// Profile@6b57696f
// ConcurrentTime : 1561
// Profile@25cc8e0a
// StructuredTime : 1538
```
Congratulations. If we compare ConcurrentTime : 1561 with StructuredTime : 1538 the times are similar.

Will it be a good approach to have a single scope and run all the coroutines there? Of course not, now let's
customize our example a little.

First we are going to make some modifications to our CoroutineUtils class

```kotlin
object CoroutineUtils {
    private val defaultScope = CoroutineScope(Dispatchers.IO + CoroutineName("General Purpose")) // Default scope
    suspend fun <T> runAsync(coroutineScope: CoroutineScope = defaultScope, block: suspend () -> T) = coroutineScope.async {
        println("${this.coroutineContext[CoroutineName.Key]} is executing in ${Thread.currentThread().name}")
        block() }
}
```
As you can see our runAsync function receives by parameter a coroutineScope and takes defaultScope by default in case we do not specify any scope.

We also add for debugging purposes.
```kotlin
println("${this.coroutineContext[CoroutineName.Key]} is executing in ${Thread.currentThread().name}")
```

With the above modification in ProfileService we can manage our own scope if we wish. For debugging purposes we are going to launch the function userRepository.findUserInfo(id) function with our custom scope and
trackingRepository.findTrackingInfo(id) under the default scope. Let's see how it works.

```kotlin
class ProfileService(private val userRepository: UserRepository, private val trackingRepository: TrackingRepository) {

    private val scope = CoroutineScope(Dispatchers.IO + CoroutineName("Service")) // this is optional, but you can create you own scope

    suspend fun getProfile(id: Int): Profile {
        val userInfo = CoroutineUtils.runAsync(scope) { userRepository.findUserInfo(id) }
        val trackingInfo = CoroutineUtils.runAsync { trackingRepository.findTrackingInfo(id) }
        return Profile(userinfo = userInfo.await(), trackingInfo = trackingInfo.await())
    }
}
```

Let's run our program again.

```kotlin
// Profile@7c0e2abd
// SequentialTime : 3028
// Profile@6b57696f
// ConcurrentTime : 1585
// CoroutineName(Service) is executing in DefaultDispatcher-worker-1
// CoroutineName(General Purpose) is executing in DefaultDispatcher-worker-2
// Profile@2ddab2e3
// StructuredTime : 1548
```

As you can see in the lines:

```kotlin
// CoroutineName(Service) is executing in DefaultDispatcher-worker-1
// CoroutineName(General Purpose) is executing in DefaultDispatcher-worker-2
```
Our suspension functions are running with different scopes and in different threads.









