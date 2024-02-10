# Implicitly Sendable Closures

* Proposal: [SE-NNNN](0NNN-implicitly-sendable-closures.md )
* Authors: [Matt Massicotte](https://github.com/mattmassicottte)
* Review Manager: TBD
* Status: **Awaiting review**
* Review: ([pitch](https://forums.swift.org/t/implicitly-sendable-functions/69950)

## Introduction

This proposal changes closures to become implicitly Sendable when isolated to a global actor, making them behave like all other non-protocol types.

## Table of Contents

* [Introduction](#introduction)
* [Motivation](#motivation)
* [Proposed solution](#proposed-solution)
* [Detailed design](#detailed-design)
    + [Diagnostic](#diagnosic)
* [Source compatibility](#source-compatibility)
* [ABI compatibility](#abi-compatibility)
* [Implications on adoption](#implications-on-adoption)
* [Alternatives considered](#alternatives-considered)
* [Acknowledgments](#acknowledgments)

## Motivation

To quote SE-0316:

> A non-protocol type that is annotated with a global actor implicitly conforms to Sendable.

However, this actually isn't true today. Globally-isolated closures do not implicitly conform to Sendable. This presents two problems. First, it's confusing that the rules should be different in this case when there are no data race implications. And second, it impacts usability. These closures cannot themselves be captured by @Sendable closures, which makes them unusable with `Task`.

## Proposed solution

The solution is to remove this special case. Closures should be implicitly Sendable when isolated to a global actor.

## Detailed design

Today this code produces a warning:

```swift
func test() {
	let closure: @MainActor () -> Void = {
		print("hmmmm")
	}

	Task {
		// warning about capturing non-Sendable
		await closure()
	}
}
```

The proposed solution is just to make global actor-isolated closures implicitly Sendable. No other semantics need to change.

### Diagnosis

Once closures are implicitly Sendable, including both an actor annotation and `@Sendable` no longer make sense. To help make that clear, there should be a diagnostic when both are present with a fix-it to remove the lower-precedence `@Sendable`. This will also help with source compatibility.

## Source compatibility

The change proposed here is loosening a restriction. This may have the effect of making currently-invalid code valid. But it will not make any currently-valid code invalid.

However, it is possible for existing code to require developer attention. Consider:

```swift
let closure: @MainActor @Sendable () -> Void = { ... }
```

Right now, `@MainActor` "wins". But was that truly the author's intention? The proposed diagnostic will call their attention to this inconsistency.

## ABI compatibility

This proposal should have no impact on ABI compatibility.

## Implications on adoption

This feature can be freely adopted and un-adopted in source code with no deployment constraints and without affecting source or ABI compatibility.

## Alternatives considered

TBD

## Acknowledgments

Thank you to Frederick Kellison-Linn for surfacing this problem, and to Kabir Oberai for exploring the implications more deeply. Thank you to Holly Borla for clarifying the implications, especially around Region-Based Isolation.
