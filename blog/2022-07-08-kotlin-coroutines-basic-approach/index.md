---
slug: Kotlin coroutines
title: Kotlin coroutines a basic approach
authors: [luiscarbonel]
tags: [kotlin, coroutines, async, suspend]
---


When developing applications we often want our functions to behave asynchronously or non-blocking, to ensure this there are many mechanisms depending on the language you use. In this article we are going to make an introduction of how Kotlin solves this issue in a very elegant way through its coroutines.

Coroutines are threads ? Ummm, the answer is nooo, coroutines and threads are similar in that they can be executed simultaneously and they need a block of code for their construction. But the truth is that a coroutine is not linked to any particular thread, in fact when we launch a coroutine it suspends its execution in a thread and when it has the result ready it can be resumed in any of the free threads, not necessarily in the thread where it started its execution.

Next we are going to start from a requirement that usually appear when we develop in enterprise environments.

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
*findUserInfo*, *findTrackingInfo* and *getProfile* are now marked as **suspend** functions, but this is because instead of using Thread.sleep we use delay (This function delays coroutine for a given time without blocking a thread and resumes it after a specified time), which being a suspend function then the function that uses it must also be a suspend function.

On the other hand we see something different in getProfile, now we make our functions findUserInfo and findTrackingInfo to be inside launch{....} and with this we tell it to execute concurrently.

```kotlin
val user = async { findUserInfo(id) }
val trackingInfo = async { findTrackingInfo(id) }
```
Finally, to obtain the result of both then we call **await** (Awaits for completion of this value without blocking a thread and resumes when deferred computation is complete).
We can also see the **runBlocking** function which is nothing more than a coroutine builder.

As I'm a bit picky about my code structure and organization, we'll try to improve this example. 






