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
      // -- self fully initialized here --
      addDefaultData(data) // OK
    }
    addEmptyRow() // error: 'self' used in method call 'addEmptyRow' before all stored properties are initialized
  }
}
```

In this example, `self` escapes the initializer through the call to its method `addEmptyRow` (all methods take `self` as an implicit argument). But this call is flagged as an error, because it happens before `self.rows` is initialized _on all paths_ to that statement from the start of the initializer's body. Namely, if `data` is `nil`, then `self.rows` will not be initialized prior to it escaping from the initializer.
Stored properties with default values can be viewed as being initialized immediately after entering the `init`, but prior to executing any of the `init`'s statements.

Determining whether `self` is fully-initialized is a flow-sensitive analysis performed by the compiler. Because it's flow-sensitive, there are multiple points where `self` becomes fully-initialized. In the example above, there is only one point, after the rows are assigned to `[]`. Thus, it is permitted to call `addDefaultData` right after that point within the same block, because all paths leading to that statement are guaranteed to have assigned `self.rows` ahead-of-time.
 

## Motivation

While there is no existing specification for how actor initialization *should* work, that in itself is not the only motivation for this proposal.
The de facto expected behavior, as induced by the existing implementation, admits data races due to ambiguous isolation semantics.


<!-- editing stopped here -->
--------------------------------

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

Relative to the Swift 3 evolution process, the source compatibility
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
aid in migration?

## Effect on ABI stability

Does the proposal change the ABI of existing language features? The
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
here.

## Effect on API resilience

API resilience describes the changes one can make to a public API
without breaking its ABI. Does this proposal introduce features that
would become part of a public API? If so, what kinds of changes can be
made without breaking ABI? Can this feature be added/removed without
breaking ABI? For more information about the resilience model, see the
[library evolution
document](https://github.com/apple/swift/blob/master/docs/LibraryEvolution.rst)
in the Swift repository.

## Alternatives considered

Describe alternative approaches to addressing the same problem, and
why you chose this approach instead.

## Acknowledgments

If significant changes or improvements suggested by members of the 
community were incorporated into the proposal as it developed, take a
moment here to thank them for their contributions. Swift evolution is a 
collaborative process, and everyone's input should receive recognition!
