---
layout: post
title:  "Swift Actors: Prevent data races"
date:   2022-05-23
categories: ios swift concurrency
---
[Swift Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md) are a new type in Swift 5.5. They are part of concurrency changes. The Swift concurrency changes aim to detect data races and prevent all the concurrency bugs that can be tricky to find and fix.
It is essential to understand concurrency, what data races are, and how to tackle them. This article will explore Actors, how they work and how to use them in your project.

## What is an Actor?

Swift has various types that we can use: classes, structs, and enums. Classes are reference types, and structs are value types. 
When you use classes, you must remember that they declare shared mutable states across the program. We can see that that can be tricky when you mix them with concurrency. You must do manual synchronization to avoid data races. 
Data races can occur when two threads simultaneously access and write the same data. Data races often lead to strange and unpredictable behavior in your application.

An Actor is a reference type that protects access to its mutable state from data races. You define an Actor using an `actor` keyword:

```swift
actor GameScore {...}
```

An Actor can have initializers, methods, properties, and subscripts. It can conform to protocols, extend them and work with generics.

An Actor is a reference type as a class, but there is one thing that you can’t do with an Actor, and that is inheritance:

```swift
actor BasketballGameScore: GameScore {}
```
You will get this error:
>*Actor types do not support inheritance*

## Synchronization - preventing Data Races

We would use various locks to create synchronized access to mutable data to prevent data races. Let’s look at the following example where we want to track the latest game score, for example, basketball. We want to know the latest score, and we want to add the new score:

```swift
class GameScore {
    private var score = [String]()
    
    var latestScore: String? {
        return score.last
    }
    
    func add(_ newScore: String) {
        score.append(newScore)
    }
}
```

This code is fine until we don’t use it in a multi-threaded environment. We can get into a lot of problems if we use this concurrently. To fix this, we can go by the route to do all our reads and writes on the specific DispatchQueue to ensure that all the operations execute in serial order:

```swift
class GameScore {
    private var score = [String]()
    private let queue = DispatchQueue(label: "gameScore.queue")
    
    var latestScore: String? {
        queue.sync {
            return score.last
        }
    }
    
    func add(_ newScore: String) {
        queue.sync {
            score.append(newScore)
        }
    }
}
```

If we want to have a concurrent specific DispatchQueue, we can sync reads and writes by the barrier flag:

```swift
class GamesScoreQueue {
    private var score = [String]()
    private let queue = DispatchQueue(label: "gamesCore.queue.concurrent", attributes: .concurrent)
    
    var latestScore: String? {
        queue.sync(flags: .barrier) {
            return score.last
        }
    }
    
    func add(_ newScore: String) {
        queue.sync(flags: .barrier) {
            score.append(newScore)
        }
    }
}
```

As we can see, we have some manual synchronization to do. To tackle this, we use an Actor. It automatically serializes all synchronized access to its properties and methods. When we refactor our GameScore  class to an Actor, we get this:

```swift
actor GameScore {
    var score = [String]()
    
    var latestScore: String? {
        return score.last
    }
    
    func add(_ newScore: String) {
        score.append(newScore)
    }
}
```

### Accessing Actor's data?

We don’t know when access is allowed and when another thread will perform access to mutable data, so we can’t just access the mutable property directly. To create asynchronous access, we use await keyword:

```swift
let gameScore = GameScore()
await gameScore.latestScore
```

### Data Races can still occur when using Actors

Actors can help prevent data races, and they give us an effortless way to synchronize access preventing weird crashes and behaviors. But we need to be careful; they are not a universal solution in a multi-threaded environment where we can forget about data races and race conditions. What if we have two queues, one accessing our score and one storing it:

```swift
queueOne.async {
    print(await gamesScore.latestScore)
}

queueTwo.async {
    await gamesScore.add("30:32")
}
```

We don’t know which one will fire first, so we can have different behavior and inconsistent game score.

## Conclusion

Actors are straightforward and neat ways to deal with asynchronous access to mutable data. They provide exquisite, readable, and one-way solutions to prevent data races. 
In future articles, I’ll cover Actor Isolation, Sendable, and MainActor. So stay tuned!
