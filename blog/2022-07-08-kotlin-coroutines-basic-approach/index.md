---
slug: Kotlin coroutines
title: Kotlin coroutines a basic approach
authors: [luiscarbonel]
tags: [kotlin,ktor,profiles,profile configuration]
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
    suspend fun getProfile(id: Int): Profile {
        val user = findUserInfo(id)
        val trackingInfo = findTrackingInfo(id)
        return Profile(user, trackingInfo)
    }

    private suspend fun findUserInfo(id: Int): UserInfo? {
        delay(1500)
        return users[id]
    }
    private suspend fun findTrackingInfo(id: Int): TrackingInfo? {
        delay(1500)
        return tracking[id]
    }
}
```
