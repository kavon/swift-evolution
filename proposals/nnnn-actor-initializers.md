# Actor Initializers and Deinitializers

* Proposal: [SE-NNNN](NNNN-actor-initializers.md)
* Authors: [Kavon Farvardin](https://github.com/kavon), [John McCall](https://github.com/rjmccall), [Konrad Malawski](https://github.com/ktoso)
* Review Manager: TBD
* Status: **Partially implemented in `main`.**
* Previous Discussions:
  * [On Actor Initializers](https://forums.swift.org/t/on-actor-initializers/49001) 

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
This proposal aims to shore up the definition of an actor, to clarify when the isolation of the data begins and ends for an actor instance, along with what can be done inside the body of an actor's `init` and `deinit` declarations.

## Background

Before diving into this proposal, it is important to review the behaviors of initializer and deinitializer declarations in Swift.

As with classes, actors support both synchronous and asynchronous initializers, along with a user-provided deinitializer, like so:

```swift
actor Database {
  var rows: [String]
  init() { /**/ }
  init(with: [String]) async { /**/ }
  deinit { /**/ }
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

Determining whether `self` is fully-initialized is a flow-sensitive analysis performed by the compiler. Because it's flow-sensitive, there are multiple points where `self` becomes fully-initialized, and these points are not explicitly marked in the source program. In the example above, there is only one such point, immediately after the rows are assigned to `[]`. Thus, it is permitted to call `addDefaultData` right after that assignment statement within the same block, because all paths leading to the call are guaranteed to have assigned `self.rows` beforehand.
 

## Motivation

While there is no existing specification for how actor initialization *should* work, that in itself is not the only motivation for this proposal.
The de facto expected behavior, as induced by the existing implementation, admits data races due to ambiguous isolation semantics.

Unlike other synchronous methods of an actor, a synchronous (or "ordinary") `init` is special in that it is treated as being `nonisolated` from the outside, meaning that there is no `await` (or actor hop) required to call the `init`. But, an `init`'s purpose is to bootstrap an actor-instance called `self`. Thus, at various points within the `init`'s body, `self` is considered a fully-fledged actor instance whose members must be protected by isolation. The existing implementation of actor initializers does not perform this enforcement, leading to data races with the code appearing in the `init`:

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

This example exhibits a race because `self`, once fully-initialized, is ready to provide isolated access to its members, i.e., it does not start off in a reserved state. Isolated access is obtained by "hopping" to the executor corresponding to `self`. But, because `init` is synchronous, a hop to `self` cannot be performed. Thus, while this `init` is nessecarily treated as `nonisolated` from the outside, once `self` is initialized, the remainder of the `init` is subject to data races.

If the `init` in the previous example only changed to be `async`, this data race does still does not go away. The existing implementation does not perform a hop to `self` in such initializers, even though it now could. This is not just a bug that has a straightforward fix, because if an asynchronous actor `init` were isolated to the `@MainActor`: 

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

then which executor should be used? It is both valid and desirable to be able to isolate an actor's `init` to a global actor, such as the `@MainActor`, to ensure that the right executor is used for the operations it performs. We'd also like to perform the initialization while on `@MainActor` so that the `ConnectionStatusDelegate` can be updated without any possibility of suspension (i.e., no `await` needed). 

The existing implementation makes it impossible to write a correct `init` for the example above, because 
the `init` is considered to be entirely isolated to the `@MainActor`. Thus, it's not possible to initialize `self.status` _at all_. It's not possible to `await` and hop to `self`'s executor to perform an assignment to `self.status`, because `self` is not even a valid actor-instance yet!

<!-- TODO: motivate discussing `deinit` too! -->

<!-- TODO: motivate a decision on how actors should support constructor delegation. It's a reference type without inheritance, so should `convenience` still be required, in case inheritance is added later on? -->

## Proposed solution

Describe your solution to the problem. Provide examples and describe
how they work. Show how your solution is better than current
workarounds: is it cleaner, safer, or more efficient?

## Detailed design

Describe the design of the solution in detail. If it involves new
syntax in the language, show the additions and changes to the Swift
grammar. If it's a new API, show the full API and its documentation
comments detailing what it does. The detail in this section should be
sufficient for someone who is *not* one of the authors to be able to
reasonably implement the feature.

## Source compatibility

<!-- Relative to the Swift 3 evolution process, the source compatibility
requirements for Swift 4 are *much* more stringent: we should only
break source compatibility if the Swift 3 constructs were actively
harmful in some way, the volume of affected Swift 3 code is relatively
small, and we can provide source compatibility (in Swift 3
compatibility mode) and migration.

Will existing correct Swift 3 or Swift 4 applications stop compiling
due to this change? Will applications still compile but produce
different behavior than they used to? If "yes" to either of these, is
it possible for the Swift 4 compiler to accept the old syntax in its
Swift 3 compatibility mode? Is it possible to automatically migrate
from the old syntax to the new syntax? Can Swift applications be
written in a common subset that works both with Swift 3 and Swift 4 to
aid in migration? -->

## Alternatives considered

Describe alternative approaches to addressing the same problem, and
why you chose this approach instead.

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
