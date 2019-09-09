# Support passing `Array<T>` to parameters of type `T...`

* Proposal: [SE-NNNN](NNNN-forward-array-to-variadic-args.md)
* Authors: [Gwynne Raskind](https://github.com/gwynne)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)
* Bugs:
  - [SR-128 Pass array to variadic function](https://bugs.swift.org/browse/SR-128)
  - [SR-1644 Issue with varargs](https://bugs.swift.org/browse/SR-1644)
  - [SR-4386 variadic parameters are interpreted as single argument when passed on](https://bugs.swift.org/browse/SR-4386)
  - [SR-9733 No Syntax for Array Splat](https://bugs.swift.org/browse/SR-9733)
* Previous Revision: [Early proposal #1](https://github.com/ArtSabintsev/swift-evolution/blob/master/proposals/array-variadic-function.md)

## Introduction

A variadic function parameter of type `T...` is available in the function body as a value of type `[T]`, but callers of such functions may not pass a value that is already an array. There are many situations where alternatives to "splatting" an array in this fashion are either slow, convoluted, or entirely unavailable. This proposal provides syntax for performing the "splat" operation in a manner consistent with existing behavior.

Swift-evolution thread: [TBD: Discussion thread topic](https://forums.swift.org/)

Previous evolution pitch threads:
 - [https://forums.swift.org/t/variadic-parameters-that-accept-array-inputs/12711/](https://forums.swift.org/t/array-splatting-for-variadic-parameters/7175/)
 - [https://forums.swift.org/t/variadic-parameters-that-accept-array-inputs/12711/](https://forums.swift.org/t/variadic-parameters-that-accept-array-inputs/12711/)
 - [https://forums.swift.org/t/explicit-array-splat-for-variadic-functions/11326](https://forums.swift.org/t/explicit-array-splat-for-variadic-functions/11326)
 - [https://forums.swift.org/t/pitch-variadic-arguments-should-accept-arrays/5327](https://forums.swift.org/t/pitch-variadic-arguments-should-accept-arrays/5327)

## Motivation

### Callsite Ergonomics

Consider `print()`'s three-argument form:

```swift
public func print(_ items: Any..., separator: String = " ", terminator: String = "\n")
```

A developer with an array of items they wish to output through this function signature can not simply pass the array as an input:

```swift
print(1, 2, 3, 4, separator: "|")
// 1|2|3|4

print([1, 2, 3, 4], separator: "|")
// [1, 2, 3, 4]
```

To get the desired result, the developer must reimplement behavior that `print()` already provides:

```swift
print([1, 2, 3, 4].map { String($0) }.joined(separator: "|"))
// 1|2|3|4
```

### API Complexity

In `print()`'s case, the languge limitation is annoying and costs a little performance at most to work around. Unfortunately, not all cases are so easily handled. **Combine** deals with this limitation by declaring two overloads, one for explicit arrays and one for variadic arguments:

```swift
extension Publisher {
    public func append(_ elements: Self.Output...) ->
        Publishers.Concatenate<Self, Publishers.Sequence<[Self.Output], Self.Failure>>

    public func append<S>(_ elements: S) ->
        Publishers.Concatenate<Self, Publishers.Sequence<S, Self.Failure>>
        where S : Sequence, Self.Output == S.Element
}
```

The `Self.Output == S.Element` constraint on the second method avoids most - but not all - cases of ambiguous overload resolution in this case, but the duplication creates a maintenance burden and is a potential source of confusion for both users and maintainers.

Generic constraints can't always make up for the limitation either. Consider `Foundation.NSArray`:

```swift
extension NSArray {
    public convenience init(objects elements: Any...)
    @nonobjc public convenience init(array anArray: NSArray)
}
```

The legacy design of `NSArray`'s Objective-C API accidentally avoided what would otherwise be an obvious problem when creating an array of arrays, thanks to the `array` label on the second initializer, but lacking that, the use of `Any` as the only possible element type would be an accident waiting to happen. An extra burden in maintenance and API bloat is once again imposed (if only theoretically in this particular instance).

### Composability Gap

It's currently impossible to "forward" conformance to the `ExpressibleByArrayLiteral` and `ExpressibleByDictionaryLiteral` protocols, as seen here:

```swift
class MyUsefulWrapper<T: InterestingProtocol> {
    /* useful wrapper things go here */
}

extension MyUsefulWrapper: ExpressibleByArrayLiteral where T: ExpressibleByArrayLiteral {
    typealias ArrayLiteralElement = T.ArrayLiteralElement

    init(arrayLiteral elements: Self.ArrayLiteralElement...) {
        self.init(T.init(arrayLiteral: /* ... What goes here?? */))
    }
}

extension MyUsefulWrapper: ExpressibleByDictionaryLiteral where T: ExpressibleByDictionaryLiteral {
    typealias Key = T.Key, Value = T.Value

    init(dictionaryLiteral elements: (Self.Key, Self.Value)...) {
        self.init(T.init(dictionaryLiteral: /* ... What goes here?? */))
    }
}
```

The most straightforward workaround is to require separate `init(forwardedArrayLiteral: [Self.ArrayLiteralElement])` and `init(forwardedDictionaryLiteral: [(Self.Key, Self.Value)])` methods in a protocol the target object must conform to, but that just pushes the problem out onto anything that ever uses the wrapper object.

There is also an "arrays of Hanoi" solution, where the original variadic overload branches based on the length of the input array and repeats the forwarding call multiple times. Unfortunately, this technique results in ugly, unmaintainable code and scales very poorly. This example of wrapping `os_log()`'s C interface demonstrates:

```swift
// Note: This doesn't actually add anything to os_log(), it's just a demo of the problem.
func my_os_log(_ message: StaticString, log: OSLog = .default, type: OSLogType = .default, _ args: CVarArg...) {
    switch args.count {
        case 0: os_log(message, log: log, type: type)
        case 1: os_log(message, log: log, type: type, args[0])
        case 2: os_log(message, log: log, type: type, args[0], args[1])
        case 3: os_log(message, log: log, type: type, args[0], args[1], args[2])
        case 4: os_log(message, log: log, type: type, args[0], args[1], args[2], args[3])
        case 5: os_log(message, log: log, type: type, args[0], args[1], args[2], args[3], args[4])
        default: fatalError("Can only handle five format specifiers.")
    }
}
```

## Proposed solution

Given that the intended desire of Swift authors is to "splat" the elements of an array into a variadic format required by a method signature, it is proposed to add a `#variadic()` directive. The directive performs a one-way, zero-cost, compile-time reinterpretation of an `Array<T>` into a `T...`. Example usage:

```swift
extension MyUsefulWrapper: ExpressibleByArrayLiteral where T: ExpressibleByArrayLiteral {
    typealias ArrayLiteralElement = T.ArrayLiteralElement

    init(arrayLiteral elements: Self.ArrayLiteralElement...) {
        self.init(T.init(arrayLiteral: #variadic(elements)))
    }
}
```

_See the [Alternatives Considered](#Alternatives-Considered) section for a detailed discussion of why a compiler directive was chosen as the proposed syntax despite repeated attempts to propose an operator for the purpose._

## Detailed design

This proposal introduces a single additional compiler directive with the name `#variadic`, which **MUST** occur within a function call's argument list.

When the compiler encounters the `#variadic` directive, it forwards the provided array argument to the called function's implementation, rather than constructing an array from the individual arguments.

The directive has a simple syntax:
- It takes a single array argument as if it were a function, and has no directly usable return value.
- It is syntactically valid only as a function call argument.
- The result "value" can only exist as the sole argument to a variadic parameter of an identical type, and can not be stored or captured in any way.

Grammatically, `function-call-argument` is amended to include the following additional productions:

```
function-call-argument → splat-expression | identifier : splat-expression
splat-expression → #variadic(expression)
````

_Note: `splat-expression` is **NOT** added to any productions for prefix, binary, primary, or postfix expressions. It is syntactically valid **ONLY* within a `function-call-argument`._

A `splat-expression` is _semantically_ valid only when ALL of the following conditions are true:

- It is syntactically valid as per the above grammar productions.
- The complete `parameter-list` of the called function includes exactly one `type-annotation` having the `...` specifier identifying a variadic parameter.
  - _Note: It is already a rule of the grammar that a function may have at most one variadic parameter._
- The `splat-expression` appears in the same ordinal position as the variadic parameter, and with a matching external parameter name, if any.
- The `expression` interior to the `splat-expression` has a resolved type of `Swift.Array<Element>`, such that the resolved type of `Element` and the declared type of the variadic parameter are directly compatible.

In practical terms, a `splat-expression` allows a variadic function to be called as if a parameter of type `T...` was temporarily given the type `Swift.Array<T>` instead.

Consider the following code snippet with reards to the syntactical, grammatical, and semantic rules which apply to `splat-expression`s:

```swift
typealias Value = /* any type */;

func f<Value>(input: [Value])  { g(param: #variadic(input.map { $0 })) }
func g<Value>(param: Value...) { /* function body */ }

func h<Value>(input: [Value])  { i(param: input.map { $0 }) }
func i<Value>(param: [Value])  { /* function body */ }
```

- At no time does any usage, or lack thereof, of the splat expression in any way augment, supplement, limit or otherwise alter the signature of the called function.
- It is ill-formed for the evaluated type of a `splat-expression` to be anything other than `Array<T>`.
- It is ill-formed for the type of the array itself to be e.g. `Optional<Array<T>>`, even if `T` is itself optional.
- It is ill-formed for `#variadic()` to appear in any lexical structure other than a function call argument.
- It is ill-formed for a `splat-expression` to appear in place of a parameter whose type is already an array. For example, the expression `i(param: #variadic(input))` is ill-formed.
- It is ill-formed for a `splat-expression` to appear alongside other arguments which would otherwise be considered part of the variadic parameter. For example, the expression `g(param: input.first!, #variadic(input))` is ill-formed.

## Source compatibility

This proposal is purely additive.

- The `#` symbol is not valid for general use by Swift programs
- The identifier `#variadic` is not alredy in use by the language.

## Effect on ABI stability

This proposal has no effect on ABI stability.

Because the `splat-expression` does not in any way alter the signature or calling convention of functions receiving variadic parameters as arguments, there is no concern regarding use across module boundaries.

## Effect on API resilience

Likewise, the use of `#variadic()` has no impact on calling convention or signature, and thus offers no API resilience concerns.

## Alternatives considered

A number of alternatives were considered and rejected for this functionality:

### Provide additional overloads

Providing overloads which take both variadic and array forms of a given input leads to trivial cases of resolution ambiguity:

```swift
func multizip<T>(arrays: T...) -> [T] where T: Collection { multizip(arrays: arrays) }
func multizip<T>(arrays: [T]) -> [T] where T: Collection { /* body */ }

// Which version is invoked? Is this "zip two arrays of String" or "zip one array of arrays of String"?
multizip([["a"], ["b"]])
```

Even where the ambiguity can be resolved, it is often not possible for consumers of a given module to alter that module to do so.

### Allow `Array<T>` to implicitly convert to `T...`

This is exactly the same as providing an explicit overload and suffers from the same ambiguity, with even less ability to work around it.

### Call the directive `#splat()`, `#spread()`, `#forward()` etc.

While the terms "splat" or "spread" for the operation in question are familiar to many, this is not universal. The author had never heard of the usage of "spread" for it until doing the research for this proposal. The term `forward` does not cover the full scope of the operation involved - it is valid for sending arrays to variadic parameters, not just forwarding inputs from other variadics.

In the end, "`variadic`" was settled on as:
- Simultaneously both specific enough and encompassing enough to address the feature's scope without exceeding it.
- Consistent with Swift's usage of the same word to describe the very function parameters it affects.
- Being just unique enough a word to give it a low false positive rate when searched for - in other words, easy for beginners to find in the docs.

### Use an operator for the splat operation instead of a compiler directive

There are plenty of languages which provide the "splat" operation using an operator:

- PHP and JavaScript use unary prefix `...`, though JavaScript calls it "spreading"
- Scala uses `_*`
- Perl, Ruby, and Python have `*` (and the latter two additionally use `**`)
- C# has the `params` keyword which sidesteps the issue by accepting both arguments lists and arrays transparently
- C has an exceedingly awkward ability to "forward" variadic parameters using `va_list`, but arrays are not first-class types in C, so there isn't any real equivalent to the "splat" operation.

The `*` operator has a long, sordid history in C and Objective-C. Using it, or Scala's similar `_*`, would be confusing at best to many Swift developers, not to mention introducing ambiguity with vector multiply operations and the scalar multiplication operator. Nor does it necessarily suggest the splat operation at first or even second glance.

As a mental excercise, a number of other possible operators were considered, but none were sufficiently intuitive or unambiguous:

- `~array`
- `->array`
- `>>array`
- `array&`
- `^array`
- `array^`
- `\\array`
- `>array<`

Many pitches and forum threads for variadic generics present arguments in favor of the `...` operator for a sense of consistency. In consideration of that option, the following justifications are offered for rejecting both the ellipsis operator and operators in general:

- `...` already has many use cases: As a variadic function parameter marker, as a shorthand for constructing `ClosedRange`s (infix), as a shorthand for `PartialRangeUpTo` (prefix), as shorthand for `PartialRangeFrom` (postfix), and as the interface to constructing `UnboundedRange`s. This is already too many meanings to assign to any one operator.

- The use of `...` for variadic parameters dates back to before Swift was open source; it was effectively copied from [Objective-]C, at a time when the language had different goals, no supporting community, and a necessary lack of experience with how it would be used. That `...` is still used this way today is just one of many historical curiosities of Swift. The present-day Swift community should be mindful of this historical context, but said context offers no support to the idea that any consistency is provided by adding even more uses to an operator that already has so many.

- While the ellipsis operator is unambiguous in a function declaration's `parameter-clause`, in a `function-call-argument-list` it can appear almost anywhere. Even if the compiler experiences no ambiguity (which is difficult to guarantee), figuring out the actual effect of using `...` in any given call would be very difficult in many cases.

- In present-day Swift, any non-restricted operator - including `...` - is subject to overloading by both the current module and any imported modules. The use of `...` for closed and unbounded ranges is implemented this way, by providing overloads strategically in the standard library.

  However, adding the "splat" operation to any given operator - a compile-time operation with a result that is inexpressible in the language itself - forces the compiler to either:
  - Disable the "splat" operation entirely whenever an overload is in scope (effectively making the feature's existence moot), or
  - Maintain additional semantics, machinery, and/or interfaces to resolve whether the user overload or the "splat" operation is invoked in any given context.

### A different language construct

The most serious problem is the Swift language lacks a construct designed for situational modifiers of this nature:

- A standard library function would not only misrepresent the level at which "splat" occurs, but run afoul of the type system, which has no means of expressing "only valid where a variadic parameter goes" or "must be the only item passed to a variadic parameter" constraints.
- The `@identiifer` syntax is already used by property wrappers and built-in attributes. The splat operation is an active behavior of the compiler; it does not describe any particular innate characteristic of the related code.
- `$` is already used by identifiers and property wrappers; reuse in this context would be ambiguous at best, directly conflicting at worst.
- `\` also already has multiple meanings, and once again ambiguity would be quick to arise.

On the other hand, each directive currently introduced by `#` describes an active, (usually) compile-time effect:

- OS and compilation condition detection
- Objective-C selector and keypath construction
- compile-time diagnostics

The splat operation in Swift expresses a decision at compile-time to avoid synthesizing an array in favor of using one already available, which seems a reasonable fit.
