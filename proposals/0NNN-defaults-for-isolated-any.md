# Implicit defaults for `@isolated(any)` function parameters

* Proposal: [SE-NNNN]()NNN-defaults-for-isolated-any.md)
* Authors: [Matt Massicotte](https://github.com/mattmassicotte) + more I'm sure
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: TBD
* Upcoming Feature Flag: **needed, name?**
* Previous Revision: [1](https://github.com/sophiapoirier/swift-evolution/blob/closure-isolation/proposals/nnnn-closure-isolation-control.md?plain=1)
* Review: ([pitch](https://forums.swift.org/...))

## Introduction

It can be surprisingly error-prone and complex to emulate the behavior of `Task` initializers.
These APIs, as well as any wrappers around them,
expose programmers to a considerable amount of subtle concepts.
And because `Task` is a foundational part of the system,
this exposure can happen very early in the process of learning about concurrency.

This proposal makes two changes to `@isolated(any)` function parameters.
First, it makes all `@isolated(any)` function parameters implicitly `sending`.
Second, it encorporates the behavior of the `@_inheritActorContext` attribute
as the default for these parameters.

To accomodiate these changes, it also formally deprecates `@_inheritActorContext`,
and adds `@noninheriting` as a parameter attribute to disable the inheritance.

These changes make the default behaviors line up with the common usage patterns while also
helping to improve progressive disclosure for the systems that use `@isolated(any)` functions.

## Motivation

It is a very common to wrap unstructured `Task` creation in a way
that introduces undesirable changes in both runtime and compile-time behavior.
Here's an example that, despite getting quite close, has problems.

```swift
func wrapper(_ operation: sending @escaping @isolated(any) () async -> Void) {
  Task(operation: operation)
}
```

This wrapper function has a sutble difference because it is missing the `@_inheritActorContext` parameter attribute.

The relationship between `@_inheritActorContext` and `@isolated(any)` functions is frequently confused.
This is partially because experimental attributes are suppressed from rendered documentation,
making its presence very easy to miss.
But, it is also such a common misunderstanding because the bahavior that `@_inheritActorContext` provides
is an intuitive default.
The `Task` creation APIs are used extensively,
especially when first learning at about the concuerrency system,
and their behavior is often internalized as expected.

Looking at it from the other direction,
functions that either accept or should accept `@isolated(any)` parameters **without** `@_inheritActorContext` are rare.
The most prominent use is the `TaskGroup` APIs.
In that context it make sense, because introducing concurrency here is the point.
And while this functionality is very important,
it is not nearly as common for a programmer to need to wrap this APIs.
Diagnostics around problems with isolation assumptions or sendability are
also is also much more likely to be encountered immediately.

A further complexity is the pairing of `sending` and `@isolated(any)`.
You cannot meanginfully make use of the isolation carried by an `@isolated(any)` function
if it cannot be safely transferred to a different isolation domain.
In fact, this requirement was specifically called out in the introduction of [`@isolated(any)`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0431-isolated-any-functions.md).
However, the timing of that proposal and the introduction of `sending` overlapped in an unforutnate way.

There's actually an existing diagnostic that shows how related these concepts are to each other,
which is produced by the following code:

```swift
// Error: @_inheritActorContext only applies to 'sending' parameters or parameters with '@Sendable' function types; this will be an error in a future Swift language mode
func test(@_inheritActorContext _ arg: () -> Void) {
}
```

Misunderstandings around these aspects of the language abound.
Not only is the language poorly-optimized for the commmon case,
it's very easy to produce invalid or unhelpful combinations.
Not to mention getting things right can require a considerable amount of verbosity.

## Proposed solution

This proposal incorporates actor context inheritance into all `@isolated(any)` function parameters
and makes them implicitly `sending`.
The existing `@_inheritActorContext` attribute will be formally deprecated.
APIs that do not want isolation inheritance can use the `@noninheriting` attribute in the parameter position.

The `Task` and `TaskGroup` APIs will be updated to adopt the changes.

(I'm having an extremely hard time coming up with a code example that illustrates the problems with a non-Sendable function.)

## Detailed design

### Implicit `sending`

Making all `@isolated(any)` function parameters implicitly `sending` is just formally recognizing an existing requirement.
An `@isolated(any)` parameter that cannot be sent today is unhelpful,
as this would require that its dynamic and static isolation be the same.
And if this is true, there's no need for the `@isolated(any)` in the first place.

A diagnostic will be included to indicate that an existing `sending` is redundant and can be removed.

### Implicit inheritance

A function parameter that accepts an `@isolated(any)` function parameter today will implicitly also inherit actor context.

Today, the default is to not inherit any context on closure formation.
Function signatures need to opt into inheritance with `@_inheritActorContext`.
This is done for all of the `Task` initializers.

This, combined with an implicit `sending` makes it possible to implement this `Task` initializers similarly to this prototype:

```swift
extension Task where Failure == Never {
  @discardableResult
  public init(
    priority: TaskPriority? = nil,
    operation: @escaping @isolated(any) () async -> Success
  ) {
    // ...
  }
}
```

### Disabling inheritance

Flipping the default here requires there be a way to accomidate systems that do not want the inheritance.
Two APIs that do not inherit actor context are `Task.detached` and `TaskGroup`.

The `TaskGroup.addTask` API can be changed to omit `sending` and adopt `@noninheriting`.

```swift
extension TaskGroup {
  mutating func addTask(
    executorPreference taskExecutor: (any TaskExecutor)? = nil,
    priority: TaskPriority? = nil,
    @noninheriting operation: @escaping @isolated(any) () async -> ChildTaskResult
  ) {
    // ...
  }
}
```

The `Task.detacted` API can also be changed in a similar way.

```swift
extension Task where Failure == Never {
  @discardableResult
  static func detached(
    name: String? = nil,
    priority: TaskPriority? = nil,
    @noninheriting operation: @escaping @isolated(any) () async throws -> Success
  ) -> Task<Success, any Error> {
    // ...
  }
}
```

There's an interesting distinction here worth noting.
`TaskGroup.addTask` disables **isolation inheritance** only,
whereas `Task.detached` disables all context: isolation, priority, and task locals.

This is an asymmetricity that existed in the current behavior as well.
What it meant to **not** have `@_inheritActorContext` depended on
context and this proposal does not change that.

(I may need help here more formally defining what is going on in this case.)

## Source compatibility

These changes can be source-incompatible in a variety of ways.
Implying `sending` for `@isolated(any)` function parameters will break the following code:

```swift
class NonSendable {
}

func acceptIsolatedAny(_ operation: @escaping @isolated(any) () async -> Void) {
}

@MainActor
func test(_ ns: NonSendable) {
  acceptIsolatedAny({ print(ns) })
  acceptIsolatedAny({ print(ns) })
}

```

However, this is an example of subtly-incorrect usage of an `@isolated(any)` function.
Because `@isolated(any)` functions require `sending` in the general case anyways,
it seems reasonable to unconditionally accept this instead as a missing diagnostic.

On the other hand, changing actor context isolation defaults has the potential to
both break source compatibility and runtime behavior.
This is something that could even conceivably go through a migration
process.

However, because of the expected distribution of `@isolated(any)` and `@_inheritActorContext`,
either present or accidentally missing,
changing this default is far more likely to fix a latent bug than introduce one.
For that reason, a migration seems unnecessary and a simple feature flag should be sufficient.

## ABI compatibility

(I need help figuring this out. I believe that `sending` will affect mangling.)

Adopting these changes in `Task` and `TaskGroup` are compatible,
because these functions are all annotated with `@_alwaysEmitIntoClient`.

## Implications on adoption



## Future directions

(mention the relationship with closure isolation control)

## Alternatives considered

(hmm)

## Acknowledgments

Thanks to Ole Begemann, Jamieq, Sean Herber, and Jared Sinclair for feedback on an early revision of these ideas. Thanks to Holly Borla for brainstorming and helping to shape a final concept.

