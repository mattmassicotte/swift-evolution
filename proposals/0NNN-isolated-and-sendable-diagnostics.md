# Feature name

* Proposal: [SE-NNNN](0NNN-isolated-and-sendable-diagnostics.md)
* Authors: [Matt Massicotte](https://github.com/mattmassicotte)
* Review Manager: TBD
* Status: **Awaiting review**
* Review: ([pitch](https://forums.swift.org/t/diagnostics-for-global-isolation-and-sendable-conformance/69951))

## Introduction

It is possible to use both a global actor annotation and also a Sendable conformance on a type. These are in conflict, though today the compiler prioritizes the global isolation. This proposal introduces a diagnostics that highlights the inconsistency with an option to remove the Sendable conformance.

## Motivation

Applying global actor isolation and Sendable conformance simultaneously is a logical mistake. Types that are globally-isolated are already Sendable. At best, this can lead to confusion. Is this type only usable from the actor, or from any actor?

```swift
@MainActor
class MyClass: Sendable {
}
```

As it turns out, isolation checking is currently prioritizing the global actor annotation. The following code compiles without warning:

```swift
class NonSendable {
}

@MainActor
class MyClass: Sendable {
	var value = NonSendable() // explicitly not Sendable
}
```

## Proposed solution

This proposal introduces a diagnostic for any type except closures that are simultaneously globally-isolated and also Sendable.

## Detailed design

The diagnostic serves two purposes. First, it highlights the logical inconsistency to the developer. Second, by offering a fix-it to remove the Sendable conformance, it indicates that the global isolation is a stronger promise and is mostly likely what should be kept.

Closures are omitted from this checking because, today, globally isolated closures are not implicitly Sendable. So, there are situations where you must use both annotations to be warning-free.

## Source compatibility

This change is just surfacing the existing isolation checking rules. The resulting warnings could potentially be surprising, but they will have no impact on the semantics of existing code.

## ABI compatibility

This proposal should have no impact on ABI compatibility.

## Implications on adoption

This feature can be freely adopted and un-adopted in source code with no deployment constraints and without affecting source or ABI compatibility.

## Future directions

[Another proposal](https://forums.swift.org/t/implicitly-sendable-closures/69950) has been made to extend this diagnostic to cover closures as well.

## Alternatives considered

TBD
