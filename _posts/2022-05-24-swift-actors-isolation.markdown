---
layout: post
title:  "Swift Actors: Isolation"
date:   2022-05-24
categories: ios swift concurrency
---
By default, each mutable property and method is isolated, which means they can’t be accessed from the external code. If you want to call them, you need to use await keyword.
Immutable, constant properties and methods that you mark explicitly with nonisolated keywords can be accessed directly from the external code because they are in a non-isolated state.
Here is an example:

```swift
actor GameScore {
    let name: String
    
    //...
}
```

The property `name` is immutable, so there’s no need to mark it directly with nonisolated keyword; it is safe to access from non-isolated state. Let’s add one more immutable property `place` and computed property `venue`:

```
actor GameScore {
    let name: String
    let place: String
    
    var venue: String {
        "See a game: \(name) at place: \(place)"
    }
    
    //...
}
```

If we accessed the venue, we would get the following error:
> Actor-isolated property `venue` can not be referenced from a non-isolated context

To fix this, we must tell the compiler it is safe to access the property `venue` by marking the property nonisolated:

```swift
actor GameScore {
    let name: String
    let place: String
    
    nonisolated var venue: String {
        "See a game: \(name) at place: \(place)"
    }
    
    //...
}
```

Let’s try to remove `venue` and add CustomStringConvertible protocol conformance. We will get this:

```swift
extension GameScore: CustomStringConvertible {
    var description: String {
        "See a game: \(name) at place: \(place)"
    }
}
```

And when we try to use it, we will get the error:
> Actor-isolated property ‘description’ cannot be used to satisfy a protocol requirement

To fix this and to use `description,` we need to mark it with a nonisolated keyword:

```swift
extension GameScore: CustomStringConvertible {
    nonisolated var description: String {
        "See a game: \(name) at place: \(place)"
    }
}
```

## Conclusion

Actors are straightforward and neat ways to deal with asynchronous access to mutable data.
We can use isolated and nonisolated keywords to control mutable and immutable states’ access precisely.
