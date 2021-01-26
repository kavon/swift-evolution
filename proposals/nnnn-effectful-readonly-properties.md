# Effectful Read-only Properties

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Kavon Farvardin](https://github.com/kavon)
* Review Manager: TBD
* Status: **Awaiting implementation**

<!--
*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN) or [apple/swift-evolution-staging#NNNNN](https://github.com/apple/swift-evolution-staging/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)
-->

## Introduction

<!-- A short description of what the feature is. Try to keep it to a
single-paragraph "elevator pitch" so the reader understands what
problem this proposal is addressing. -->

Nominal types such as classes, structs, and enums in Swift support [computed properties](https://docs.swift.org/swift-book/LanguageGuide/Properties.html), which are members of the type that invoke programmer-specified computations when getting or setting them; instead of being tied to storage like stored properties.  The recently accepted proposal [SE-0296](0296-async-await.md) introduced asynchronous functions via `async`, in conjunction with `await`, but did not specify that computed properties can support effects like asynchrony.  Furthermore, to take full advantage of `async` properties, the ability to specify that a property `throws` is also important.  This document aims to partially fill in this gap by proposing a syntax and semantics for effectful read-only computed properties.

<!-- Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/) -->

#### Terminology
A *read-only computed property* is a computed property that only defines a `get`-ter, which can be `mutating`. Throughout the remainder of this proposal, any unqualified mention of a "property" refers to a read-only computed property.  Furthermore, unless otherwise specified, the concepts of synchrony, asynchrony, and the definition of something being "async" or "sync" are as described in [SE-0296](0296-async-await.md).

An *effect* is an observable behavior of a function. Swift's type system tracks a few kinds of effects: `throws` indicates that the function may return along an exceptional failure path with an `Error`, `rethrows` indicates that a throwing closure passed into the function may be invoked, and `async` indicates that the function may reach a suspension point.

The [Swift concurrency roadmap](https://forums.swift.org/t/swift-concurrency-roadmap/41611) outlines a number of features that are referenced in this proposal, such as [structured concurrency](https://forums.swift.org/t/concurrency-structured-concurrency/41622) and [actors](https://forums.swift.org/t/concurrency-actors-actor-isolation/41613).  Overviews of these features are out of the scope of this proposal, but basic understanding of the importance of these features is required to fully grasp the motivation of this proposal.

## Motivation

<!-- Describe the problems that this proposal seeks to address. If the
problem is that some common pattern is currently hard to express, show
how one can currently get a similar effect and describe its
drawbacks. If it's completely new functionality that cannot be
emulated, motivate why this new functionality would help Swift
developers create better Swift code. -->

An asynchronous function is designed for computations that may or always will suspend to perform a context switch before returning.  Of primary concern in this proposal are scenarios where the future use of Swift concurrency features are limited due to the lack of *effectful read-only computed properties* (which I will refer to as simply "effectful properties" from now on), so we will consider those first.  Then, we will consider programming patterns in existing Swift code where the availability of effectful properties would help simplify the code.

#### Future Code

An asynchronous call cannot appear within a synchronous context. This fundamental restriction means that properties will be severely limited in their ability to use Swift's new concurrency features. The only capability available to them is spawning detached tasks, but the completion of those tasks cannot be awaited in synchronous contexts:

```swift
// ...
class Socket {
  // ...
  public var alive : Bool {
    get {
      let handle = Task.runDetached { await self.checkSocketStatus() }
      //     /-- ERROR: cannot 'await' in a sync context
      //     v
      return await handle.get()
    }
  }

  private func checkSocketStatus() async -> Bool { /* ... */}
}
```

As one might imagine, a type that would like to take advantage of actors to isolate concurrent access to resources, while exposing information about those resources through properties, is not possible because one must use `await` to interact with the actor from outside of its isolation context:

```swift
struct Transaction { /* ... */ }
enum BankError : Error { /* ... */}

actor class AccountManager {
  // `lastTransaction` is viewed as async from outside of the actor
  func lastTransaction() -> Transaction { /* ... */ }
}

class BankAccount {
  // ...
  private let manager : AccountManager?
  var lastTransactionAmount : Int {
    get {
      guard manager != nil else {
      // /-- ERROR: cannot 'throw' in a non-throwing context
      // v
        throw BankError.notInYourFavor
      }
      //     /-- ERROR: cannot 'await' in a sync context
      //     v
      return await manager!.lastTransaction().amount
    }
  }
}
```

While the use of `throw` in the `lastTransactionAmount` getter is rather contrived, realistic uses of `throw` in properties have been [detailed in a prior pitch](https://github.com/beccadax/swift-evolution/blob/throwing-properties/proposals/0000-throwing-properties.md) and detached tasks [can be formulated to throw](https://github.com/DougGregor/swift-evolution/blob/structured-concurrency/proposals/nnnn-structured-concurrency.md#cancellation-1) a `CancellationError`.

Furthermore, without effectful read-only properties, actor classes cannot define *any* computed properties that are accessible from outside of its isolation context as a consequence of their design: a suspension may be performed before entering the actor's isolation context. Thus, we cannot turn `AccountManager`'s `lastTransaction()` method into a computed property without treating it as `async`.


#### Existing Code

According to the [API design guidelines](https://swift.org/documentation/api-design-guidelines/), computed properties that do not quickly return, which includes asynchronous operations, are not what programmers typically expect:

> **Document the complexity of any computed property that is not O(1).** People often assume that property access involves no significant computation, because they have stored properties as a mental model. Be sure to alert them when that assumption may be violated.

but, computed properties that may block or fail do appear in practice (see the motivation in [this pitch](https://github.com/beccadax/swift-evolution/blob/throwing-properties/proposals/0000-throwing-properties.md)). 

As a real-world example of the need for effectful properties, [the SDK defines a protocol](https://developer.apple.com/documentation/avfoundation/avasynchronouskeyvalueloading) `AVAsynchronousKeyValueLoading`, which is solely dedicated to querying the status of a type's property, while offering an asynchronous mechanism to load the properties.  The types that conform to this protocol include [AVAsset](https://developer.apple.com/documentation/avfoundation/avasset), which relies on this protocol because its read-only properties are blocking and failable.

Let's distill the problem solved by `AVAsynchronousKeyValueLoading` into a simple example.  In existing code, it is impossible for property `get` operation to also accept a completion handler, i.e., a closure for the property to invoke with the result of the operation. This is because a computed property's `get` operation accepts zero arguments (excluding `self`). Thus, existing code that wished to use computed properties in scenarios where the computation may be blocking must use various workarounds. One workaround is to define an additional asynchronous version of the property as a method that accepts a completion handler:

```swift
class NetworkResource {
  var isAvailable : Bool {
    get { /* a possibly blocking operation */ }
  }
  func isAvailableAsync(completionHandler: ((Bool) -> Void)?) {
    // method that returns without blocking.
    // completionHandler is invoked once operation completes.
  }
}
```

The problem with this code is that, even with a comment on `isAvailable` to document that a `get` on this property may block, the programmer may mistakenly use it instead of `isAvailableAsync` because it is easy to ignore a comment. But, if `isAvailable`'s `get` were marked with `async`, then the type system will force the programmer to use `await`, which tells the programmer that the property's operation may suspend until the operation completes. Thus, this effect specifier enhances the recommendation made in the [API design guidelines](https://swift.org/documentation/api-design-guidelines/) by leveraging the type checker to warn users that the property access may involve significant computation.

<!--
```swift
// code that interfaces with an API that requires a completion handler,
// and thus we want to use `withUnsafeContinuation`. Note that this
// was not previously possible without hacky workarounds.
```
-->

## Proposed solution

<!-- Describe your solution to the problem. Provide examples and describe
how they work. Show how your solution is better than current
workarounds: is it cleaner, safer, or more efficient? -->

For the problems detailed in the motivation section, the proposed solution is to allow `async`, `throws`, or both of these effect specifiers to be marked on a read-only computed property's `get` definition:

```swift
// ...
class BankAccount {
  // ...
  var lastTransactionAmount : Int {
    get async throws {   // <-- proposed: effects specifiers!
      guard manager != nil else {
        throw BankError.notInYourFavor
      }
      return await manager!.lastTransaction().amount
    }
  }
}
```

At corresponding use-sites of these properties, the expression will be treated as having the effects listed in the `get`-ter, requiring the usual `await` or `try` to surround it as-needed:

```swift
let acct = BankAccount()
let amount = 0
do {
  amount = try await acct.lastTransactionAmount
} catch { /* ... */ }

extension BankAccount {
  func hadRecentWithdrawl() async throws -> Bool {
    return try await lastTransactionAmount < 0
  }
}
```

The usual short-hands for do-try-catch, `try!` and `try?`, work as usual.

Computed properties that can be modified, such as via a `set`-ter, will not be allowed to use effects specifiers on their `get`-ter, regardless of whether the setter would require any effects specifiers.  The main purpose of imposing such a restriction is to limit the scope of this proposal to a simple, useful, and easy-to-understand feature. Limiting effects specifiers to read-only properties in this proposal does _not_ prevent future proposals from offering them for all computed properties. For more discussion of why effectful setters are tricky, see the "Extensions considered" section of this proposal.

## Detailed design

<!-- Describe the design of the solution in detail. If it involves new
syntax in the language, show the additions and changes to the Swift
grammar. If it's a new API, show the full API and its documentation
comments detailing what it does. The detail in this section should be
sufficient for someone who is *not* one of the authors to be able to
reasonably implement the feature. -->


This section takes a deep-dive into what changes will be made to Swift and its implementation if this proposal were accepted.

#### Syntax and Semantics

Under the [grammar rules for declarations](https://docs.swift.org/swift-book/ReferenceManual/Declarations.html), under "Type Variable Properties", the proposed modifications and additions are:


```
getter-clause → attributes? mutation-modifier? "get" getter-effects? code-block
getter-effects → "throws"
getter-effects → "async" "throws"?
```

where `getter-effects` is a new production in the grammar. This production allows you to write one of the three possible combinations of effects keywords between `get` and `{`, while enforcing an order between `async` and `throws` that mirrors the existing one on functions.  Additionally, one can declare (but not define) an effectful property by appending the effect keywords following the `get`, like so:

```
getter-setter-keyword-block → "{" getter-keyword-clause setter-keyword-clause? "}"
getter-setter-keyword-block → "{" setter-keyword-clause getter-keyword-clause "}"
getter-keyword-clause → attributes? mutation-modifier? "get" getter-effects?
```

For example, one can write:

```swift
protocol P {
  associatedtype T
  var prop : T { get async throws }
}
```

to enforce that a type conforming to `P` provides an effectful read-only property `prop`. While the modification to the grammar above means that `getter-setter-keyword-block` will accept the string `{get async throws set}`, such declarations formed from it will be rejected during semantic checking for having a setter while being effectful.

The interpretation of an effectful property definition is straightforward: the `code-block` appearing in such a `get`-ter definition will be allowed to exhibit the effects specified, i.e., throwing and/or suspending such that `await` and `try` expressions are allowed in that `code-block`. Furthermore, expressions that evaluate to an access of the property will be treated as having the effects that are declared on that property. One can think of such expressions as a simple desugaring to a method call on the object. It is always possible to determine whether a property has such effects, because the declaration of the property is always known statically. Thus, it is a static error to omit the appropriate `await`, `try`, *etc*.


#### Protocol conformance

In order for a type to conform to a protocol containing effectful properties, the type must contain a property that exhibits *the same or fewer effects than the protocol specifies*. This rule mirrors how conformance checking happens for functions with effects. Here is a well-typed example without any superfluous `await`s or `try`s that follows this rule:

```swift
protocol P {
  associatedtype T
  var someProp : T { get async throws }
}

class NoEffects : P { var someProp : Int { get { 1 } } }

class JustAsync : P { var someProp : Int { get async { 2 } } }

class JustThrows : P { var someProp : Int { get throws { 3 } } }

class Everything : P { var someProp : Int { get async throws { 4 } } }

func exampleExpressions() async throws {
  let _ = NoEffects().someProp
  let _ = await JustAsync().someProp
  let _ = try! JustThrows().someProp
  let _ = try! await Everything().someProp

  let _ = try! await (NoEffects() as P).someProp
  let _ = try! await (JustAsync() as P).someProp
  let _ = try! await (JustThrows() as P).someProp
  let _ = try! await (Everything() as P).someProp
}
```

Formally speaking, let us consider a getter `G` to have a set of effects `effects(G)` associated with it. This proposal adds one additional rule to conformance checking: if a getter definition `D` is said to satisfy the requirements of a protocol's getter declaration `DP`, then `effects(D)` is a subset of `effects(DP)`.


<!--
#### Property Inheritance

TODO: works the same as before, I think.
-->

#### Actors

The [actor model](https://forums.swift.org/t/concurrency-actors-actor-isolation/41613) has been proposed as part of the [Swift concurrency roadmap](https://forums.swift.org/t/swift-concurrency-roadmap/41611), and as of writing the actor portion of the roadmap is still in the pitch stage. Nevertheless, the current design of actor classes will isolate all properties to contexts that are part of that actor's domain, e.g., anything accessable from the actor's `self`. Thus, the following code would be invalid:

```swift
actor class MyActor {
  public var prop : Int {
    get {
      return 42
    }
  }
}

func foo() async -> Int {
  let act = MyActor()
  return await act.prop // error: actor-isolated property 'prop' can only be referenced inside the actor
}
```

but, if `prop` were a method instead (whether that method were `async` or not), then calling it from outside of the actor's domain would be valid: the call-site outside of the actor becomes a suspension point that requires an `await`.

Given the syntax and semantics for effectful read-only properties discussed earlier, `await act.prop` is a well-defined expression that is a possible suspension point.  Thus, this proposal provides the ability to access *any* read-only computed property belonging to an actor, from _outside_ of that actor, as a natural extension of its design.


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

The proposed syntactic changes are such that if they appeared in previous versions of the language, they would have been rejected as an error by the parser.

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

This proposal is additive and limits its scope intentionally to avoid breaking ABI stability.

## Effect on API resilience

<!-- API resilience describes the changes one can make to a public API
without breaking its ABI. Does this proposal introduce features that
would become part of a public API? If so, what kinds of changes can be
made without breaking ABI? Can this feature be added/removed without
breaking ABI? For more information about the resilience model, see the
[library evolution
document](https://github.com/apple/swift/blob/master/docs/LibraryEvolution.rst)
in the Swift repository. -->

As an additive feature, this will not affect API resilience. But, existing APIs that adopt effectful read-only properties will break backwards compatibility, because users of the API will be required to wrap accesses of the property with `await` and/or `try`.

## Extensions considered

In this section, we will discuss extensions and additions to this proposal, and why they are not included in the proposed design above.

#### Effectful settable properties

Defining the interactions between async and/or throwing writable properties and features such as:

1. `inout`
2. `_modify`
3. property observers, i.e., `didSet`, `willSet`
4. property wrappers
5. subscripts

is a large project that requires a significant implementation effort. By limiting effectful properties to those that only define a `get` operation, we can avoid all of those complexities. The proposed design for effectful read-only properties is small and straightforward to implement, while still providing a notable benefit to real-world programs and Swift's concurrency efforts.

#### Objective-C bridging

There is precedent for automatically synthesizing Swift interfaces to ObjC types. For example, [SE-0297](0297-concurrency-objc.md) describes the generation of `async` functions in Swift by recognizing completion-handler arguments in ObjC signatures. Becca's [original pitch for throwing properties](https://github.com/beccadax/swift-evolution/blob/throwing-properties/proposals/0000-throwing-properties.md) also included discussion of how bridging might work. Effectful read-only properties can similarly be synthesized, and would provide some benefit.  Based on some cursory searching, here is the rough breakdown of how often methods that might be candidates for importing as an effectful computed property appear in the SDK:

* A `throws` getter is rare and `throws` setters are more common.
* There are a fair number of `async` getters and `async` setters.

Thus, it seems that providing a one-sided ObjC bridging capability, without also bridging effectful setters, would not be ideal and has been excluded from this proposal. But, it is believed that even without ObjC bridging, adding effectful, read-only properties to Swift through this proposal would still be an overall benefit with a high power-to-weight ratio.

## Alternatives considered

<!-- Describe alternative approaches to addressing the same problem, and
why you chose this approach instead. -->

The `rethrows` specifier is excluded from this proposal because one cannot pass a closure (or any other explicit value) during a property `get` operation.

The forthcoming `async`/`await` feature is purpose-built for enabling asynchronous programming, so no consideration is given for alternative solutions that do not rely on that feature for asynchronous properties. The same reasoning applies to `throws`/`try`.

## Acknowledgments

Thanks to Doug Gregor and John McCall for their guidance while crafting this proposal. The feasibility and design choices for this proposal were influenced by [Becca Royal-Gordon's proposal for throwing property accessors](https://github.com/beccadax/swift-evolution/blob/throwing-properties/proposals/0000-throwing-properties.md) and recent discussions with her.