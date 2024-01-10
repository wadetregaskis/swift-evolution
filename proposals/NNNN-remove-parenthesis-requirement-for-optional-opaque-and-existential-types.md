# Remove parenthesis requirement for optional opaque and existential types

* Proposal: [SE-NNNN](NNNN-remove-parenthesis-requirement-for-optional-opaque-and-existential-types.md)
* Authors: [Holly Borla](https://github.com/hborla), [Wade Tregaskis](https://github.com/wadetregaskis)
* Review Manager: TBD
* Status: **Awaiting implementation**

#### TBD

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN) or [apple/swift-evolution-staging#NNNNN](https://github.com/apple/swift-evolution-staging/pull/NNNNN)
* Upcoming Feature Flag: *if applicable* `MyFeatureName`
* Review: ([pitch](https://forums.swift.org/...))

## Introduction

This proposal simplifies the syntax for type declarations involving optional opaque or optional existential types, by removing the requirement that the type inside the `Optional` be declared inside parenthesis.  e.g.:

```swift
var userPreference: (any Preference)?
```

…may now instead be written as:

```swift
var userPreference: any Preference?
```

Parenthesis may still be used if the author wishes.

## Motivation

These particular parenthesis do not provide any clarity and are unintuitive.  While the presence of helpful compiler diagnostics and FixIts help mitigate the impact, it is still a frustration to users.

The parenthesis are not meaningful since even in their absence there is only one valid interpretation:

1. From a technical perspective, `some P?` cannot mean `some Optional<P>` because that is syntactically invalid.
2. From an intent perspective, the result is the same no matter what ordering is chosen.  `some Optional<P>`, while not syntactically invalid, can still only be attempting one thing:  to declare an optional type of some P.

    In fact, for the existential case the compiler already provides a FixIt to rewrite `any Optional<P>` as the surely intended `Optional<any P>`.  It is able to do this because there is no plausible ambiguity as to what the author intended.

Part of the reason the current parenthesis requirement is confusing is that it is inconsistent with how concretely-typed cases work.  Consider, for example:

```swift
func a1() -> Int?
func a2() -> Optional<Int>
```

These are equivalent ways of writing the same thing, and are accepted by the compiler, but if you try to generalise them to opaque or existential types the results are inconsistent:

```swift
func b1() -> some BinaryInteger? // ❌ A 'some' type must specify only 'Any', 'AnyObject', protocols, and/or a base class
func b2() -> Optional<some BinaryInteger> // ✅

func c1() -> any BinaryInteger? // ❌ Optional 'any' type must be written '(any BinaryInteger)?'
func c2() -> Optional<any BinaryInteger> // ✅
```

Similarly, if you start with non-optional types and then make them optional, there is currently inconsistency as to how you do so.

This also confounds otherwise straightforward refactorings (e.g. textual search & replace).

[SE-0328: Structural opaque result types](https://github.com/apple/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md) [considered not requiring the parenthesis but rejected it](https://github.com/apple/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md#syntax-for-optionals-1) on the belief that it was insufficient convenience for the inconsistency it introduces.

The inconsistency it was concerned with is syntactically similar cases:

1. Closure syntax, e.g. `() -> P?`.  This can plausibly be taken as intending to make the whole closure optional, but it is more likely intending to make the closure's return value optional.  The compiler assumes the latter intent, and requires parenthesis to spell the former.
2. Type unions, e.g. `Q & P?`.  The only valid interpretation is as `Optional<Q & P>`, because `Q & Optional<P>` is logically invalid.  Currently the compiler requires parenthesis, like it does for `some P?` or `any P?`.

The first case, closure syntax, is distinct because there _is_ ambiguity as to the author's intent.  Parenthesis are unavoidable, and that they are not required as `() -> (P?)` is essentially syntactic sugar to smooth the more common case.

The second case, type unions, is quite similar to optional opaque or existential types, but not quite the same.  Although there is only one _technically_ valid interpretation in the absence of parenthesis, it is _not_ clear what the author's intent was.  They might genuinely be intending to express "_requires_ Q and _might_ also be P", even though that is not supported in Swift today (and whether it has practical merit, it is logically possible).

Lastly, that we have these omitting-parenthesis sugars at all is well-precedented and pervasive, in operator syntax.  `1 + 2 * 3` is ambiguous, requiring knowledge of arbitrary precedence rules to correctly interpret - rules that are complex and not consistent between programming languages - and yet it is permitted in Swift.  Experience has shown that even when there _is_ room for misinterpretation, omitting parenthesis can still be the right trade-off.

**Re. the last paragraph:  selling past the close?  too tangential?  might open up a distracting can of worms regarding whether parenthesis _should_ be required for multi-operator expressions?**

## Proposed solution

The parenthesis will become optional in cases where there is no plausible ambiguity as to the author's intent, such as the simple and common forms of standalone `some P?` and `any P?`, as well as where such simple expressions appear already enclosed in parenthesis or brackets, e.g. `Response<some P?>`.

Counter-examples, where parenthesis will remain required in order to clarify intent, include `any P & Q?`.  **TODO: more examples**

**TODO: flesh out the details of interactions with other type declaration syntax**

## Detailed design

Describe the design of the solution in detail. If it involves new
syntax in the language, show the additions and changes to the Swift
grammar. If it's a new API, show the full API and its documentation
comments detailing what it does. The detail in this section should be
sufficient for someone who is *not* one of the authors to be able to
reasonably implement the feature.

## Source compatibility

This proposal is backwards compatible insofar as it merely makes the parenthesis optional, it does not ban them.  All existing code will continue to function in exaclty the same way as before.

## ABI compatibility

No impact, as this is purely an expansion of accepted syntax for existing concepts.

## Implications on adoption

Since prior Swift versions require the parenthesis, code that omits them will be incompatible with older compiler versions.  Adopters must ensure they do not need to support those older compiler versions.  This is particularly relevant to library authors.

## Future directions

None suggested.

## Alternatives considered

### Improving compiler diagnostics

The optional existential case is better covered, currently, than the optional opaque type case.  The former has a simple, clear message and associated FixIt:

```
Optional 'any' type must be written '(any BinaryInteger)?'
```

The opaque type case has a less friendly message:

```
A 'some' type must specify only 'Any', 'AnyObject', protocols, and/or a base class
```

…which does not suggest a solution.

It does also have a FixIt, although in Xcode 15 that FixIt is broken (it turns `some P?` into `(some P)??`, which is likely to compound the user's confusion).

If the parenthesis requirement is kept, it would be wise to at least improve the optional opaque type compiler diagnostics.

## Acknowledgments

**TODO**