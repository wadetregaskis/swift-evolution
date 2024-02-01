# Remove parenthesis requirement for opaque and existential types

* Proposal: [SE-NNNN](NNNN-remove-parenthesis-requirement-for-opaque-and-existential-types.md)
* Authors: [Holly Borla](https://github.com/hborla), [Wade Tregaskis](https://github.com/wadetregaskis)
* Review Manager: TBD
* Status: **Awaiting implementation**

#### TBD

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN) or [apple/swift-evolution-staging#NNNNN](https://github.com/apple/swift-evolution-staging/pull/NNNNN)
* Upcoming Feature Flag: *if applicable* `MyFeatureName`
* Review: ([pitch](https://forums.swift.org/...))

## Introduction

This proposal simplifies the syntax for type declarations involving opaque or existential types, by removing the requirement that they be enclosed in parenthesis whenever the intent is unambiguous.  This is the case whenever the expression contains only one protocol, e.g.:

```swift
var userPreference: (any Preference)?
```

…may now instead be written as:

```swift
var userPreference: any Preference?
```

This applies to optionals (`?`), implicitly unwrapped optionals (`!`), and the `Type` member.  e.g.:

```swift
let outlet: any NSView!
```

```swift
var type: some Collection.Type
```

Parenthesis may still be used if the author wishes.

**TODO: Pick one of these two possible approaches:**

A.  In essence, this is changing the nature of `some` & `any` from prefix operators to expression markers. >>  **TODO: …which implies `some Optional<P>` should work too… should it?  See also Future Directions…**

B.  A counter-example, where the parenthesis remain required, is `(some T).X` whenever it is unclear what `X` is (now _or in future_).  This must consider the possibility of future language changes such as nested protocols.  This proposal errs on the side of caution by only covering cases where `X` cannot ever be a protocol or class type. **TODO: figure out if this is an allowlist of ?/!/Type or if it's actually determined at compile time based on the definition of T & X**

## Motivation

Swift eschews unnecessary ceremony, such as needless punctuation - particular parenthesis.

In the cases in question, these particular parenthesis do not provide any clarity - neither to humans nor the compiler.  They are unintuitive and a source of frustration to users.  While the presence of helpful compiler diagnostics and fix-its helps mitigate the impact when writing code, they do not help readers.

The parenthesis are not meaningful as even in their absence there is only one valid interpretation: `some P?` cannot mean `some Optional<P>` because `Optional<P>` is not a valid generic constraint for a type parameter.  Likewise for `some P!` and `some P.Type`. When a programmer writes `some P?`, it is already clear without parenthesis that the intent is to write an optional type of `some P`.

The compiler can already determine that `any P?` was intended to mean `(any P)?`, as evidenced by the fix-it provided with the error message for `any P?` that inserts the parenthesis. It is able to do this because there is no plausible ambiguity as to what the author intended.
    
For `some P.Type`, there is no semantic difference between `(some P).Type` and `some (P.Type)`. The existential type that represents any concrete metatype whose instance type conforms to `P` is already written as `any P.Type`; it's confusing that the opaque type that represents some concrete metatype whose instance type conforms to `P` cannot be written `some P.Type`.

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
2. Protocol compositions, e.g. `Q & P?`.  The only valid interpretation is as `Optional<Q & P>`, because `Q & Optional<P>` is logically invalid.  Currently the compiler requires parenthesis, like it does for `some P?` or `any P?`.

The first case, closure syntax, is distinct because there _is_ ambiguity as to the author's intent.  Parenthesis are unavoidable, and that they are not required as `() -> (P?)` is essentially syntactic sugar to smooth the more common case.

In the second case, protocol compositions are different from single-constraint opaque and existential types because there are multiple places where the `?` or `.Type` could be written. For example, one may argue that `any P & Q.Type` should mean `(any P & Q).Type`, but it's far less obvious that the constraint to `P` still applies to the instance type instead of the resulting metatype. Further, if metatypes ever gain the ability to conform to protocols in the future, we may want the `any P & Q.Type` syntax to truly mean `any P & (Q.Type)`, aka an existential metatype where the metatype itself also conforms to `P`.

## Proposed solution

We propose eliding parenthesis for optional opaque and existential types using the `?` syntax, and opaque metatypes using the `some P.Type` syntax. In these cases, there is no plausible ambiguity as to the author's intent. The parenthesis can still be written explicitly, and the shorthand can be used in structural positions, e.g. `Response<some P?>`.

Eliding parenthesis is only supported when there is a single constraint; `some` and `any` explicitly applied to a protocol composition may not elide parenthesis.

## Detailed design

Describe the design of the solution in detail. If it involves new
syntax in the language, show the additions and changes to the Swift
grammar. If it's a new API, show the full API and its documentation
comments detailing what it does. The detail in this section should be
sufficient for someone who is *not* one of the authors to be able to
reasonably implement the feature.

Counter-examples, where parenthesis will remain required in order to clarify intent, include `any P & Q?`.  **TODO: more examples**

**TODO: flesh out the details of interactions with other type declaration syntax**

## Source compatibility

This proposal is backwards compatible insofar as it merely makes the parenthesis optional, it does not ban them.  All existing code will continue to function in exaclty the same way as before.

## ABI compatibility

No impact, as this is purely an expansion of accepted syntax for existing concepts.

## Implications on adoption

Since prior Swift versions require the parenthesis, code that omits them will be incompatible with older compiler versions.  Adopters must ensure they do not need to support those older compiler versions.  This is particularly relevant to library authors.

## Future directions

### Also allow e.g. `some Optional<P>`

If `some P?` is now valid, one could argue that its long form should also be valid for consistency, i.e.: `some Optional<P>`.  The same motiviations & principles apply - most importantly, that the author's intent is unambiguous.

This may have greater ramifications, however.  The farther the `some` / `any` keyword is from the applicable part of the expression, the greater the potential for misunderstanding the definition - even if technically it is unambiguous.  Also, it increases the probability of mistakes during refactoring (e.g. a naive textual search-and-replace for `Optional<P>` to `Optional<T>` where `T` is a concrete type, and thus the resulting whole expression `some Optional<T>` is invalid).

Additionally, it introduces redundancy into the grammar, as now you can write `Optional<some P>` _or_ `some Optional<P>`, and they have the exact same effect.  Having multiple ways to write the same thing makes the language harder to learn (newcomers may tend to assume there's a difference and be waylaid trying to figure out what it is) and harder to work with (now one has to consider multiple possible grammars when searching, for example).

While that also applies in principle to optional parenthesis, parenthesis are already optional in most cases - e.g. one already has to know that `1 + 2` is the same as `(1 + 2)` or `(1) + 2` or `((1 + ((2))))` or any of infinitely many other possibilities.  So this proposal introduces no _new_ conceptual burden on learners.

**TODO: figure out if we actually believe this should be a future (or never) direction, or if actually we should include this in the proposal.**

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
