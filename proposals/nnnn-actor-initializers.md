# Actor Initializers and Deinitializers

* Proposal: [SE-NNNN](NNNN-actor-initializers.md)
* Authors: [Kavon Farvardin](https://github.com/kavon), [John McCall](https://github.com/rjmccall), [Konrad Malawski](https://github.com/ktoso)
* Review Manager: TBD
* Status: **Partially implemented in `main`.**
* Previous Discussions:
  * [On Actor Initializers](https://forums.swift.org/t/on-actor-initializers/49001)
  * [Deinit and MainActor](https://forums.swift.org/t/deinit-and-mainactor/50132)

<!-- *During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN) or [apple/swift-evolution-staging#NNNNN](https://github.com/apple/swift-evolution-staging/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md) -->

## Introduction

Actors are a relatively new nominal type in Swift that provides data-race safety for its mutable state.
The protection is achieved by _isolating_ the mutable state of each actor instance to at most one task at a time.
The proposal that introduced actors ([SE-0306](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)) is quite large and detailed, but misses some of the subtle aspects of creating and destroying an actor's isolated state.
This proposal aims to shore up the definition of an actor, to clarify *when* the isolation of the data begins and ends for an actor instance, along with *what* can be done inside the body of an actor's `init` and `deinit` declarations.

## Background

To get the most out of this proposal, it is important to review the existing behaviors of initializer and deinitializer declarations in Swift.

As with classes, actors support both synchronous and asynchronous initializers, along with a user-provided deinitializer, like so:

```swift
actor Database {
  var rows: [String]

  init() { /* ... */ }
  init(with: [String]) async { /* ... */ }
  deinit { /* ... */ }
}
```

An actor's initializer respects the same fundamental rules surrounding the use of `self` as other nominal types: until `self`'s stored properties have all been initialized to a value, `self` is not a fully-initialized instance.
This concept of values being *fully-initialized* before use is a fundamental invariant in Swift.
To prevent uses of ill-formed, incomplete intances of `self`, the compiler restricts `self` from escaping the initializer until all of its stored properties are initialized:

```swift
actor Database {
  var rows: [String]

  func addDefaultData(_ data: String) { /* ... */ }
  func addEmptyRow() { rows.append(String()) }

  init(with data: String?) {
    if let data = data {
      self.rows = []
      // -- self fully-initialized here --
      addDefaultData(data) // OK
    }
    addEmptyRow() // error: 'self' used in method call 'addEmptyRow' before all stored properties are initialized
  }
}
```

In this example, `self` escapes the initializer through the call to its method `addEmptyRow` (all methods take `self` as an implicit argument). But this call is flagged as an error, because it happens before `self.rows` is initialized _on all paths_ to that statement from the start of the initializer's body. Namely, if `data` is `nil`, then `self.rows` will not be initialized prior to it escaping from the initializer.
Stored properties with default values can be viewed as being initialized immediately after entering the `init`, but prior to executing any of the `init`'s statements.

Determining whether `self` is fully-initialized is a flow-sensitive analysis performed by the compiler. Because it's flow-sensitive, there are multiple points where `self` becomes fully-initialized, and these points are not explicitly marked in the source program. In the example above, there is only one such point, immediately after the rows are assigned to `[]`. Thus, it is permitted to call `addDefaultData` right after that assignment statement within the same block, because all paths leading to the call are guaranteed to have assigned `self.rows` beforehand. Keep in mind that these rules are not unique to actors, as they are enforced in initializers for other types like structs and classes.
 

## Motivation

While there is no existing specification for how actor initialization and deinitialization *should* work, that in itself is not the only motivation for this proposal.
The *de facto* expected behavior, as induced by the existing implementation, is also problematic. In summary, the major problems are:

  1. Initializers can exhibit data races due to ambiguous isolation semantics.
  2. Initializer delegation requires the use of a `convenience` keyword, which does not have meaning without inheritance.
  3. Deinitializers are run without obtaining access to the actor's executor, yet they are permitted to invoke any actor-isolated method.

The following subsections will discuss these three high-level problems in more detail.

### Initializer Races

Unlike other synchronous methods of an actor, a synchronous (or "ordinary") `init` is special in that it is treated as being `nonisolated` from the outside, meaning that there is no `await` (or actor hop) required to call the `init`. This is because an `init`'s purpose is to bootstrap a fresh actor-instance, called `self`. Thus, at various points within the `init`'s body, `self` is considered a fully-fledged actor instance whose members must be protected by isolation. The existing implementation of actor initializers does not perform this enforcement, leading to data races with the code appearing in the `init`:

```swift
actor StatsTracker {
  var counter: Int

  init(_ start: Int) {
    self.counter = start
    // -- self fully-initialized here --
    Task.detached { await self.tick() }
    
    // ... do some other work ...
    
    if self.counter != start { // 💥 race
      fatalError("state changed by another thread!")
    }
  }

  func tick() {
    self.counter = self.counter + 1
  }
}
```

This example exhibits a race because `self`, once fully-initialized, is ready to provide isolated access to its members, i.e., it does *not* start in a reserved state. Isolated access is obtained by "hopping" to the executor corresponding to `self` from an asynchronous function. But, because `init` is synchronous, a hop to `self` fundamentally cannot be performed. Thus, once `self` is initialized, the remainder of the `init` is subject to the kind of data race that actors are meant to eliminate.

If the `init` in the previous example were only changed to be `async`, this data race still does not go away. The existing implementation does not perform a hop to `self` in such initializers, even though it now could to prevent races. This is not just a bug that has a straightforward fix, because if an asynchronous actor `init` were isolated to the `@MainActor`: 

```swift
class ConnectionStatusDelegate {
  @MainActor
  func connectionStarting() { /**/ }

  @MainActor
  func connectionEstablished() { /**/ }
}

actor ConnectionManager {
  var status: ConnectionStatusDelegate
  var connectionCount: Int

  @MainActor
  init(_ sts: ConnectionStatusDelegate) async {
    // --- on MainActor --
    self.status = sts
    self.status.connectionStarting()
    self.connectionCount = 0
    // --- self fully-initialized here ---
    
    // ... connect ...
    self.status.connectionEstablished()
  }
}
```

then which executor should be used? Should it be valid to isolate an actor's `init` to a global actor, such as the `@MainActor`, to ensure that the right executor is used for the operations it performs? The example above serves as a possible use case for that capability: being able to perform the initialization while on `@MainActor` so that the `ConnectionStatusDelegate` can be updated without any possibility of suspension (i.e., no `await` needed). 

The existing implementation makes it impossible to write a correct `init` for the example above, because 
the `init` is considered to be entirely isolated to the `@MainActor`. Thus, it's not possible to initialize `self.status` _at all_. It's not possible to `await` and hop to `self`'s executor to perform an assignment to `self.status`, because `self` is not a fully-initialized actor-instance yet!

### Initializer Delegation

All nominal types in Swift, except actors, explicitly support initializer delegation, which is when one initializer calls another one to perform initialization.
For classes, initializer [delegation rules](https://docs.swift.org/swift-book/LanguageGuide/Initialization.html#ID216) are complex due to the presence of inheritance.
So, classes have a required and explicit `convenience` modifier to make, for example, a distinction between initializers that *must* delegate and those that do not.
In contrast, value types do *not* support inheritance, so [the rules](https://docs.swift.org/swift-book/LanguageGuide/Initialization.html#ID215) are much simpler: any `init` can delegate, but if it does, then it must delegate or assign to `self` in all cases:

```swift
struct S {
  var x: Int
  init(_ v: Int) { self.x = v }
  init(b: Bool) {
    if b {
      self.init(1)
    } else {
      self.x = 0 // error: 'self' used before 'self.init' call or assignment to 'self'
    }
  }
}
```

Actors, which are reference types (like a classes), do not support inheritance. But, currently they must use the `convenience` modifier on an initializer to perform any delegation. That modifier appears to serve little use for actors, so is it still needed?

<!-- TODO: look into NSObject-inheriting actors and other funky stuff -->

### Deinitializer Isolation

<!-- TODO: this is a pretty weak "problem", since it's entirely dependent
on how custom executors are structured. The main problem is that if there
is another reference to an actor's executor, which is a serial executor, then it's not correct to allow its methods & computed properties to be called from its deinit. -->

A user-defined `deinit` plays an important role in programming idioms such as [RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization). In Swift, only reference types support such a `deinit` and it is automatically called whenever the last reference to the object is destroyed, which can happen virtually anywhere. The implicit contract of a `deinit` is that, at the beginning of the `deinit`, no other references to `self` exist. In addition, after `deinit` has finished executing, any copies of `self` created during the `deinit` are not valid.

The single-reference nature of `self` in a `deinit` means that, in theory, we should not need to `await` or synchronize with an actor's executor in order to access its isolated state. This is because any task that is still awaiting access to an instance must also hold a reference to it. Thus, when there are no references to the instance left, then the executor should also be idle.

The only exception to this is when the executor is *not* exclusively owned by the actor. In the pitch for custom executors, the serial executor of an actor can be exposed and shared with other actors. In particular, this creates the possibility of an actor instance that is being dealloated, but its executor is busy servicing jobs for another instance:

```swift
actor A {
  let friend: B

  nonisolated public final 
    var serialExecutor: UnownedExecutorRef {
      return friend.serialExecutor
  }

  func f() {
    print("A: access begin!")
    // ...
    print("A: access end!")
  }

  deinit {
    f()
  }
}

actor B { 
  func f() {
    print("B: access begin!")
    // ...
    print("B: access end!")
  }
}
```

In the example above, every instance of `A` has an associated `friend` of type `B`, whose serial executor is used by the `A` instance.
There may be references to `friend` that outlive an instance of `A`.
Thus, when entering `A`'s `deinit`, the executor `friend.serialExecutor` has not been synchronized with, and may be actively running other jobs.
This presents a problem: the serial executor's invariant is that only one job is ever active at a time. Yet, if `A`'s `deinit` calls `self.f(_:)`, we may observe an illegal event ordering such as:

```
B: access begin!
A: access begin!
```

which breaks the invariant of serial execution.

## Proposed solution

The previous sections described problems with the current state of actor initialization and deinitialization:

  1. Initializers can exhibit data races due to ambiguous isolation semantics.
  2. Initializer delegation requires the use of a `convenience` keyword, which does not have meaning without inheritance.
  3. Deinitializers are run without obtaining access to the actor's executor, yet they are permitted to invoke any actor-isolated method.

The remainder of this section details the proposed solution to those problems.

**Problem 1: Initializer Data Races**

<!-- TODO: Refresh your memory on what was decided here and expand more. -->

Synchronous initializers reject all escaping uses of `self` throughout the initializer. This means that `self` can only be used to access stored properties, because invoking a computed property or method passes `self` implicitly. In addition, this means `self` cannot be captured in a closure, which prevents the main data race example from the Motivation section.

An asynchronous initializer performs an actor-hop to `self` immediately after `self` becomes fully-initialized, on all paths.

<!-- TODO: What about global actor isolation? -->


**Problem 2: Initializer Delegation**

Actors will no-longer require the `convenience` modifier on an initializer.
<!-- TODO: describe the isolation changes made to delegating initializers. -->

**Problem 3: Deinitializers and Executors**

<!-- TODO: this depends on whether we'll be able to statically determine whether an actor's executor is customized or not. If so, then we can just say that the method/computed property restriction only applies to such actors. -->

<!-- TODO: is this needed? --> 
## Detailed design

<!-- TODO: explain briefly where it is implemented. -->

<!-- TODO: What about failable and throwing initializers? -->


## Source compatibility

There is no simple way to automatically migrate applications that use `self` in actor initializers in the ways that are considered to be errors by this proposal.
At its core, the simplest migration path is to mark the initializer `async`, but that would introduce `async` requirements on callers.

<!-- TODO: The problem: delegating inits make `self` nonisolated, so the code that plays with `self` would need to move to one of those initializers. But then, you would need to make that delegating initializer `async`, which fundamentally changes the type signature. -->

## Alternatives considered

<!-- Describe alternative approaches to addressing the same problem, and
why you chose this approach instead. -->

### Deinitializers

One workaround for the lack of ability to synchronize with an actor's executor prior to destruction is to implicitly wrap the body of the `deinit` in a task. 

TODO: explain why this wouldn't work.

## Effect on ABI stability

<!-- Does the proposal change the ABI of existing language features? The
ABI comprises all aspects of the code generation model and interaction
with the Swift runtime, including such things as calling conventions,
the layout of data types, and the behavior of dynamic features in the
language (reflection, dynamic dispatch, dynamic casting via `as?`,
etc.). Purely syntactic changes rarely change existing ABI. Additive
features may extend the ABI but, unless they extend some fundamental
runtime behavior (such as the aforementioned dynamic features), they
won't change the existing ABI.

Features that don't change the existing ABI are considered out of
scope for [Swift 4 stage 1](README.md). However, additive features
that would reshape the standard library in a way that changes its ABI,
such as [where clauses for associated
types](https://github.com/apple/swift-evolution/blob/master/proposals/0142-associated-types-constraints.md),
can be in scope. If this proposal could be used to improve the
standard library in ways that would affect its ABI, describe them
here. -->

## Effect on API resilience

<!-- API resilience describes the changes one can make to a public API
without breaking its ABI. Does this proposal introduce features that
would become part of a public API? If so, what kinds of changes can be
made without breaking ABI? Can this feature be added/removed without
breaking ABI? For more information about the resilience model, see the
[library evolution
document](https://github.com/apple/swift/blob/master/docs/LibraryEvolution.rst)
in the Swift repository. -->

## Acknowledgments

<!-- If significant changes or improvements suggested by members of the 
community were incorporated into the proposal as it developed, take a
moment here to thank them for their contributions. Swift evolution is a 
collaborative process, and everyone's input should receive recognition! -->
