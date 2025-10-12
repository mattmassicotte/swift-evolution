# Feature name

* Proposal: [SE-0NNN](0NNN-awaitable-assignment.md)
* Authors: [Matt Massicotte](https://github.com/mattmassicotte)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: -
* Review: ([pitch](https://forums.swift.org/...))

## Introduction

Swift provides a lot of flexibility around effect handling.
Both the `try` and `await` keywords can appear, often optionally, in a variety of positions within statements and expressions.
They are only actually are required to appear once, at the outer-most effectful element.

However, there are two exceptions to this rule:
property assignment and subscript setters that need to cross isolation domains.
This proposal removes these exceptions, making the use of `await` more uniform and intuitive.

## Motivation

### Current Behavior

Using property assignment as a means of publishing a result is quite common.
But, understanding the relationship between actually producing that result asynchronously
and using the `await` keyword is surpringly subtle.

```swift
@concurrent
func produceValue() async -> Int {
  0
}

class AsyncTest {
  var storage: Int = 5

  subscript(index: Int) -> Int {
    get { 0 }
    set(newValue) { }
  }

  func performAssignment() async {
    await self.storage = produceValue()
    await self[0] = produceValue()
  }
}
```

Having the `await` keyword preceeding the entire statement is one of the arrangment explicitly supported by [SE-0296](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md).
However, this particular structure only works when the assignment does not involve crossing isolation domains.
If we modify the isolation of this type, perhaps even implicitly via default isolation,
the code no longer compiles.

```swift
@MainActor
class IsolatedTest {
  var storage: Int = 5

  subscript(index: Int) -> Int {
    get { 0 }
    set(newValue) { }
  }

  nonisolated func performAssignment() async {
    // ERROR: Main actor-isolated property 'storage' can not be mutated from a nonisolated context
    await self.storage = produceValue()

    // ERROR: Main actor-isolated subscript 'subscript(_:)' can not be mutated from a nonisolated context
    await self[0] = produceValue()
  }
}
```

We have made a significant modification to the concurrent behavior of this type,
but the ramifications aren't immediately intuitive.
Nothing about the structure of `performAssignment` has obviously changed,
and we can work around the issue trivially.

```swift
func assignValue(_ value: Int) {
  self.storage = value
}

nonisolated func functionalAssignment() async {
  await assignValue(produceValue())
}
```

Further highlighting the asymmetry here,
we can even introduce code that omits an `await` for a read of the same property.

```swift
nonisolated func functionalAssignment() async {
  await assignValue(self.storage)
}
```

In many real-world examples,
programmers make use of an isolated helper function to work around this issue.

```swift
nonisolated func performAssignment() async {
  let value = await produceValue()

  await MainActor.run {
    self.storage = value
  }
}
```

While this does work, it comes with downsides.
First, it requires the programmer be aware of the type's isolation.
This isn't necessarily a given when working with default isolation control.
Second, it requires referencing that isolation at all affected call sites.
And third, it reinforces a pattern that isn't necessary for regular function calls.

### Transactionality

An important consideration here is accidentally exposing intermediate states.
Properties often directly hold isolated state
and assignment makes introducing non-transactional mutations easier.

Consider this code:

```swift
actor Stateful {
  var a: Int = 1
  var b: Int = 2
}

await stateful.a = stateful.b
```

The proposed change make logical races even easier,
because it makes suspensions points less obvious in this situation.
However, there are three things worth noting about awaiting assignment.

First, diagnostic produced does explain what cannot be done,
but it does not help to understand **why**.

```
Actor-isolated property 'a' can not be mutated from a nonisolated context
```

The property cannot be mutated directly, but a trivial wrapper function can?
Why is this?
A programmer encountering this would have to do considerable research to learn
this is actually about a potentially problematic pattern and not just a syntactic limitation.
Worse, it could foster confusion around the concept of isolation,
because it strongly implies that mutation specifically is special in some way.

Second, this is the API that the author of the `Stateful` type has decided to publish.
Understanding the intention here, what operations may or may not make sense,
and how much transactionality cannot be known.
The **visible** interface could be a simple property,
but the underlying implementation could be quite complicated.
Swift a number of high-profile state observation libraries that use properties an an interface.

But, perhaps most importantly,
understanding the implications of suspensions points on transactional state mutation is an essential skill.
This is a phenomenon that a Swift programmer will be exposed to,
one that requires they develop a sense of how to recognize and deal with.
Building APIs that encourage logical races isn't a good thing.
But, disallowing this one particular construct does not further develop recognition,
thought it might indirectly encourage some limited mitigations.

Ultimately, thinking in terms of synchronous transactions while writing asynchronous code is unavoidable.
Encouraging awareness of the problem is essential,
but in order to achieve that goal, the programmer must understand the problem first.
Allowing isolated assignment will help build that awareness in a way that remains
consistent with all other uses of the `await` keyword.

## Proposed solution

This proposal relaxes the rules around suspensions points for assignment and subscripting,
such that the following code would now compile.

```swift
@concurrent
func produceValue() async -> Int {
  0
}

@MainActor
class IsolatedTest {
  var storage: Int = 5

  subscript(index: Int) -> Int {
    get { 0 }
    set(newValue) { }
  }

  nonisolated func performAssignment() async {
    await self.storage = produceValue()
    await self[0] = produceValue()
  }
}
```

The changes would apply to all isolated assignments, including for actor types.
This makes the use of the keyword more uniform and most importantly,
eliminates a fairly common source of confusion.

It is worth calling out that nothing about the setters themselves,
as covered in [SE-0310](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0310-effectful-readonly-properties.md),
is being changed.

## Detailed design

*forthcoming*

## Source compatibility

This change is purely additive.
It will not have any impact on existing source.

## ABI compatibility

This proposal has no ABI impact on existing code.

## Implications on adoption

This feature can be freely adopted and un-adopted in source
code with no deployment constraints and without affecting source or ABI
compatibility.

## Future directions

The obvious future direction here are the same as is covered in [SE-0310](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0310-effectful-readonly-properties.md).
That proposal covers the implications of effectful property setters well.

## Alternatives considered

*forthcoming*

## Acknowledgments

John McCall helped clarify the motivation behind the current behavior.
