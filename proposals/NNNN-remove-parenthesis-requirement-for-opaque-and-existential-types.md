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

This proposal simplifies the syntax for type declarations involving opaque or existential types, by removing the requirement that they be enclosed in parenthesis, e.g.:

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

## Motivation

Swift eschews unnecessary ceremony, such as needless punctuation - particular parenthesis.

In the cases in question, the parenthesis do not provide any clarity - neither to humans nor the compiler.  They are unintuitive and a source of frustration to users.  While the presence of helpful compiler diagnostics and fix-its helps mitigate the impact when writing code, they do not help readers.

The reason the parenthesis are not meaningful is that even in their absence there is only the same, sole valid interpretation: `some P?` must mean `Optional<some P>` because `some Optional<P>` is not valid (`Optional` is a concrete type name).  Likewise for `some P!` and `some P.Type`.

The compiler can already determine that `any P?` was intended to mean `(any P)?`, as evidenced by the fix-it provided with the error message for `any P?` that inserts the parenthesis. It is able to do this because there is no plausible ambiguity as to what the author intended.
    
For `some P.Type`, there is no semantic difference between `(some P).Type` and `some (P.Type)`. The existential type that represents any concrete metatype whose instance type conforms to `P` is already written as `any P.Type`; it's confusing that the opaque type that represents some concrete metatype whose instance type conforms to `P` cannot be written `some P.Type`.

Another reason the current parenthesis requirement is confusing is that it is inconsistent with how concretely-typed cases work.  Consider, for example:

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

**TODO: Describe the design of the solution in detail. If it involves new
syntax in the language, show the additions and changes to the Swift
grammar. If it's a new API, show the full API and its documentation
comments detailing what it does. The detail in this section should be
sufficient for someone who is *not* one of the authors to be able to
reasonably implement the feature.**

**TODO: flesh out the details of interactions with other type declaration syntax**

## Source compatibility

This proposal is backwards compatible insofar as it merely makes the parenthesis optional, it does not ban them.  All existing code will continue to function in exaclty the same way as before.

## ABI compatibility

No impact, as this is purely an expansion of accepted syntax for existing concepts.

## Implications on adoption

Since prior Swift versions require the parenthesis, code that omits them will be incompatible with older compiler versions.  Adopters must ensure they do not need to support those older compiler versions.  This is particularly relevant to library authors.

## Future directions

The following are put forth as _possible_ future directions, not necessarily recommended by us.  In most cases we are simply currently unsure if they are the right direction.  We enumerate them here both for intellectual interest and because we anticipate readers of this proposal may wonder why this proposal does not go further in these directions.  The limited removal of parenthesis, as proposed, is a simple change that requires no meaningful trade-offs and has no downsides, whereas these potential future directions do.

### Allow separation of the keyword from the placeholder type name

e.g. permit `some Optional<P>` as equivalent to `Optional<some P>`, on the basis that while syntactically redundant, the author's intent is still unambiguous.  Permitting either form makes the language more accomodating of natural differences in intuition and ways of thinking between programmers.

Additionally, if the abbreviation `some P?` is valid, one can argue that its long form should also be valid for consistency, i.e.: `some Optional<P>`.

However, there are potential trade-offs / downsides:

* Having multiple ways to write the same thing can make the language harder to learn (newcomers may tend to assume there's a functional difference and be waylaid trying to figure out what it is).

* The further the `some` / `any` keyword is from the placeholder type name, the greater the potential for misunderstanding or misreading the definition, and the less self-explanatory the code is - even if technically it is unambiguous.

  For example, consider encountering `any Record<Expense>` in unfamiliar code.  Without knowing a priori what `Record` or `Expense` is, you cannot tell if they are protocols or concrete types.  Therefore you cannot intuit where the existential is, the position of which may have significant ramifications (most obviously for usage and semantics, but also potentially for runtime performance).

* It increases the probability of mistakes during refactoring (e.g. a naive textual search-and-replace for `Optional<P>` to `Optional<T>` where `T` is a concrete type, and thus the resulting whole expression `some Optional<T>` is invalid).

  This similarly concerns tooling that operates on Swift code at the textual or AST levels.

### Make `some` / `any` behave like `try`

A broader generalisation of the above is for `any` / `some` to become more akin to `try`, i.e. marker keywords that signal / enable use of their function anywhere in the remainder of the statement and are not necessarily required on every individual occurrence of their behaviour within the statement.

This would permit terser syntax like `some Collection<Codable>`,  meaning some `Collection` type containing some `Codable` type.  Or similarly `any Collection<Codable>` for the existential equivalent.

Heterogenous use of the two keywords would still follow naturally due to the nesting nature of generic expressions, e.g. `some Collection<any Codable>` is still clear and unambiguous.  For that reason this would not be source-breaking (but would obviously be backwards incompatible).

This of course has the same concerns, magnified.  In particular, complex generic expressions are now not only not self-explanatory as to which parts are opaque types or existentials, but you cannot even tell how many existentials or opaque types are in the expression.  e.g. does `some Record<Expense>` mean `some Record<some Expense>` or `Record<some Expense>` or `some Record<Expense>`?

## Alternatives considered

### Improved compiler diagnostics

The optional existential case is currently better covered than the optional opaque type case.  The former has a simple, clear message and associated FixIt:

```
Optional 'any' type must be written '(any BinaryInteger)?'
```

The opaque type case has a less friendly message:

```
A 'some' type must specify only 'Any', 'AnyObject', protocols, and/or a base class
```

…which does not suggest a solution.

It does also have a FixIt, although in Xcode 15 that FixIt is broken (it turns `some P?` into `(some P)??`, which is likely to compound the beginner's confusion, or worse somehow pass type checking and produce unintended semantics).

If the parenthesis requirement is kept - if this proposal is rejected - it would be wise to _at least_ improve the optional opaque type compiler diagnostics, and fix the broken FixIt(s).

## Acknowledgments

**TODO**
