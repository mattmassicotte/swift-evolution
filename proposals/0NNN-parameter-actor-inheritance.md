# Parameter Actor Inheritance

* Proposal: [SE-NNNN](0NNN-parameter-actor-inheritance.md)
* Authors: [Matt Massicotte](https://github.com/mattmassicotte)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: TBD
* Review: [pitch](https://forums.swift.org/t/isolation-assumptions/69514)

## Introduction

This proposal changes how isolation inheritance works for isolated parameters. It makes the isolation inheritance rules more uniform while also making a specific concurrency pattern less-restrictive.

## Table of Contents

* [Introduction](#introduction)
* [Motivation](#motivation)
* [Proposed solution](#proposed-solution)
* [Detailed design](#detailed-design)
* [Source compatibility](#source-compatibility)
* [ABI compatibility](#abi-compatibility)
* [Implications on adoption](#implications-on-adoption)
* [Future directions](#future-directions)
* [Alternatives considered](#alternatives-considered)
* [Acknowledgments](#acknowledgments)

## Motivation

Right now, there is a difference in how isolation inheritance works for isolated parameters vs actor isolation. This makes it much more challenging to build intuition around how this inheritance works. But, it also makes some kinds of concurrency patterns impossible to use without being overly restrictive.

Here's an example of the difference:

```swift
class NonSendableType {
    @MainActor
    func globalActor() {
        Task {
            // accessing self ok
        }
    }

    func isolatedParameter(_ actor: isolated any Actor) {
        Task {
            // not ok to access self
        }
    }
}
``` 

The actor-isolated version works, but the static isolation is required only to get the desired inheritance. The second form looks like it should work, but with the current isolation inheritance rules, it does not.

## Proposed solution

This proposal changes the inheritance semantics for isolated parameters. An isolated parameter should always be inherited, even when used with an actor type.

## Detailed design

The inheritance rules for `Task` depend on a number of factors. When statically-isolated, isolation is unconditionally inherited. When isolated to an actor, the isolation depends on the actor value being captured explicitly. In practice, this form is a very natural arrangement. In order to access actor-protected state (via `self`), you have to capture self.

Isolated parameters are more complex, because `self` and the isolating actor are not the same. Consider:

```swift
class NonSendableType {
    var mutableState = 0

    func isolatedParameter(_ actor: isolated any Actor) {
        Task {
            self.mutableState += 1
        }
    }
}
``` 

We know that self is non-Sendable. In order for a callsite of this function to be valid, self must already be isolated to `actor`. But just matching the rules for actor inheritance won't fix the problem because self does not represent the isolating actor.

A potential workaround, which has already been implemented (https://github.com/apple/swift/pull/71143), is to explicitly capture the isolated parameter.

```swift
class NonSendableType {
    var mutableState = 0

    func isolatedParameter(_ actor: isolated any Actor) {
        Task {
            _ = actor
            self.mutableState += 1
        }
    }
}
``` 

The proposed solution is to make this capture uncondtionally implicit, even in the case of an actor-type capturing self.

## Source compatibility

The change proposed here is loosening a restriction. This may have the effect of making currently-invalid code valid. But it will not make any currently-valid code invalid.

It is worth noting that this does not affect the isolation semantics for actor-isolated types that make use of isolated parameters. It is currently impossible to access self in these cases, and even with this new inheritance rule that remains true.

```swift
actor MyActor {
    var mutableState = 0

    func isolatedParameter(_ actor: isolated any Actor) {
        self.mutableState += 1 // invalid

        Task {
            self.mutableState += 1 // invalid
        }
    }
}

@MainActor
class MyClass {
    var mutableState = 0

    func isolatedParameter(_ actor: isolated any Actor) {
        self.mutableState += 1 // invalid

        Task {
            self.mutableState += 1 // invalid
        }
    }
}
```

## ABI compatibility

This proposal should have no impact on ABI compatibility.

## Implications on adoption

This feature can be freely adopted and un-adopted in source code with no deployment constraints and without affecting source or ABI compatibility.

## Future directions

There is an interesting problem that comes up for actor types that use isolated parameters.

```swift
actor MyActor {
    var mutableState = 0

    func isolatedParameter(_ actor: isolated any Actor) {
        Task {
            // isolated to what here?
        }
    }
}
```

Today, this appears to be always be non-isolated. The change made in https://github.com/apple/swift/pull/71143 will apply, so explicit inheritance to `actor` could still be possible. It might be worth considering if the implicit isolation to `actor`, as is proposed here, is more appropriate in this situation.

## Alternatives considered

When this problem was originally brought up, there were several alternatives suggested.

The most obvious is to just not use `Task` in combination with non-Sendable types in this way. Restructuring the code to avoid needing to rely on isolation inheritance in the first place.

```swift
class NonSendableType {
    private var internalState = 0

    func doSomeStuff(isolatedTo actor: isolated any Actor) async throws {
        try await Task.sleep(for: .seconds(1))
        print(self.internalState)
    }
}
```

Despite this being a useful pattern, it does not address the underlying inheritance semantic differences.

There was also discussion about the ability to make synchronous methods on actors. The scope of such a change is much larger than what is covered here. And, again, would still not address the underlying differences.

## Acknowledgments

Thank you to Franz Busch and Aron Lindberg for looking at the underlying problem so closely and suggesting alternatives. Thank you to Holly Borla for helping to clarify the current behavior, as well as suggesting a path forward that resulted in a much simpler and less-invasive change.
