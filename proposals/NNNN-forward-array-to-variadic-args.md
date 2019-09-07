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
- https://forums.swift.org/t/array-splatting-for-variadic-parameters/7175/
- https://forums.swift.org/t/variadic-parameters-that-accept-array-inputs/12711/
- https://forums.swift.org/t/explicit-array-splat-for-variadic-functions/11326
- https://forums.swift.org/t/pitch-variadic-arguments-should-accept-arrays/5327

## Motivation

### Call Site Ergonomics

In cases where an API author has written an overload that accepts both forms of arguments, there is room for resolution amibuity:

```swift
func multizip<T>(arrays: T...) -> [T] where T: Collection { multizip(arrays: arrays) }
func multizip<T>(arrays: [T]) -> [T] where T: Collection { /* body */ }

// Which version is invoked? Is this "zip two [String]" or "zip one [[String]]"?
multizip([["a"], ["b"]])
```

Even if the ambiguity could be resolved, such as by removing the variadic variant or constraining the type of `T.Element`, it is often not an option to even try. Similar issues will be encountered by any code importing functions from (Objective-)C functions with `CVarArg` parameters.

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

To get the desired result, the developer must do more work manually:

```swift
print([1, 2, 3, 4].map { String($0) }.joined(separator: "|"))
// 1|2|3|4
```

### API Maintenance Burden

In `print()`'s case, the language limitations are more annoying, with a slight performance overhead, than anything. However, in other cases, it becomes more serious.

**Combine** deals with the inability to pass an array to a variadic parameter by declaring both forms of some methods:

```swift
extension Publisher {
    public func append(_ elements: Self.Output...) ->
        Publishers.Concatenate<Self, Publishers.Sequence<[Self.Output], Self.Failure>>

    public func append<S>(_ elements: S) ->
        Publishers.Concatenate<Self, Publishers.Sequence<S, Self.Failure>>
        where S : Sequence, Self.Output == S.Element
}
```

While the `Self.Output == S.Element` constraint on the second method makes it relatively safe* to provide both forms of input, in this case the duplication remains a maintenance burden and potential source of confusion.

Even still, sometimes generic constraints can't provide even partial safety.

Consider `Foundation.NSArray`:

```swift
extension NSArray {
    public convenience init(objects elements: Any...)
    @nonobjc public convenience init(array anArray: NSArray)
}
```

The legacy design of `NSArray`'s Objective-C API accidentally avoided what would otherwise be an obvious problem when creating an array of arrays, thanks to the `array` label on the second initializer, but lacking that, the use of `Any` as the only possible element type would be an accident waiting to happen. An extra burden in maintenance and API bloat is once again imposed (if only theoretically in this particular instance).

### Composability Gap

It is currently impossible to "forward" conformance to the `ExpressibleByArrayLiteral` and `ExpressibleByDictionaryLiteral` protocols, as seen here:

```swift
// Note: A common use for wrappers like these is helping handle data flowing through `Codable`.
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

Another option is to require separate `init(forwardedArrayLiteral: [Self.ArrayLiteralElement])` and `init(forwardedDictionaryLiteral: [(Self.Key, Self.Value)])` methods in a protocol the target object must conform to, but that just pushes the problem out onto anything that ever uses the wrapper object.

The proposed solution can be pseudo-implemented by writing out individual calls to the other type's initializer based on the number of elements in the input arrays, but at a major cost to clarity and maintainability, and it does not scale well.

As an example of the scaling problem, here is an example of wrapping `os_log()` in Swift, which still doesn't overcome type safety** concerns:

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

Given that the intended desire of Swift authors is to "splat" the elements of an array into a variadic format required by a method signature, it is proposed to add the `#variadic()` directive.

This directive will perform a one-way, zero-cost, compile-time reinterpretation of an `Array<T>` into a `T...`.

Example use of the directive:

```swift
func my_os_log(_ message: StaticString, log: OSLog = .default, type: OSLogType = .default, _ args: CVarArg...) {
    os_log(message, log: log, type: type, #variadic(args))
}
```

> _See the [**Alternatives Considered**](#alternatives-considered) section for a detailed discussion of why a compiler directive was chosen as the proposed syntax._

## Detailed design

This proposal introduces a single additional compiler directive with the name `#variadic`.

When the compiler encounters the `#variadic` directive, it forwards the array value provided to the directive to the function's implementation, rather than constructing an array from the individual arguments.

The directive has a simple syntax:
- It takes a single array argument as if it were a function, and has no directly usable return value
- It is syntactically valid only as a function call argument
- The result "value" can only exist as the sole argument to a variadic parameter of an identical type, it cannot be stored or captured in any way

Grammatically, `function-call-argument` is amended to include the following additional productions:

```
function-call-argument → splat-expression | identifier : splat-expression
splat-expression → #variadic(expression)
```

> _`splat-expression` is **NOT** added to any productions for prefix, binary, primary, or postfix expressions. It is syntactically invalid anywhere outside a `function-call-argument`._

A `splat-expression` is _semantically_ valid only when ALL of the following conditions are true:

* It is syntactically valid as per the above grammar productions.
* The complete `parameter-list` of the function includes exactly one `type-annotation` possessed of the `...` specifier identifying a variadic parameter.
  * Note: It is already a rule of the grammar that a function may have at most one variadic parameter
* The `splat-expression` appears in the same ordinal position as the variadic parameter, and with a matching external parameter name, if any.
* The `expression` interior to the `splat-expression` has a resolved type of `Swift.Array<Element>`, such that the resolved type of `Element` is either
  * identical to or
  * trivially expressible as the declared type of the variadic parameter.

In practical terms, a `splat-expression` "temporarily changes" a function's variadic parameter type to `Array<T>` rather than `T...`.

### Grammar exercise

As a demonstration of the various syntactical, grammatical, and semantic rules that apply to `splat-expression`s, consider the following code snippet:

```swift
typealias Value = /* any type */;

func f<Value>(input: [Value])  { g(param: #variadic(input.map { $0 })) }
func g<Value>(param: Value...) { /* function body */ }

func h<Value>(input: [Value])  { i(param: input.map { $0 }) }
func i<Value>(param: [Value])  { /* function body */ }
```

- At no time does any usage, or lack thereof, of the splat expression in any way augment, supplement, limit or otherwise alter the signature of the called function.
- It is ill-formed for the evaluated type of a `splat-expression` to be anything other than `Array<T>`
- It is ill-formed for the type of the array itself to be e.g. `Optional<Array<T>>`, even if `T` is itself optional
- It is ill-formed for `#variadic()` to appear in any lexical structure other than a function call argument
- If the type `T` of the function's variadic parameter is an optional, whether normal or implicitly unwrapped, the type of the provided array's elements must still match accordingly
- The compiler **MUST NOT** impose any form of syntactic or semantic difference between the received value of a variadic parameter (e.g. the variable `param` in the functions `g()` and `i()` above) and any other array, as in regards to a splat expression. (Note: It is believed that this is already true for the general case, but this proposal refrains from explicitly stating such a restriction outside the scope of the syntax with which it is concerned.)
- The body of function `f()` is semantically valid if, and only if, the body of function `h()` would also be semantically valid for any particular definition of `Value`
  - The snippet also demonstrates the property that the bodies of functions `f()` and `h()`, and of the functions `g()` and `i()`, are respectively semantically identical
- Conversely, the compiler **MUST NOT** accept the use of a `splat-expression` in the place of a parameter whose type is already an array
  - For example, this expression is ill-formed: `i(param: #variadic(input))`
- The compiler also **MUST NOT** accept a `splat-expression` if other arguments matching the variadic parameter in position are also present
  - For example, this expression is ill-formed: `g(param: input.first!, #variadic(input))`
- A `splat-expression`'s body is permitted to throw errors as the lexical context otherwise permits
  - Any required `try-operator` must appear as part of the inner `expression`, e.g.: `#variadic(try /* everything */)`
    - For example, this expression is ill-formed: `try #variadic(/* etc. */)`

## Source compatibility

This proposal is purely additive.

The `#` symbol is not valid for general use by Swift programs, and the identifier `#variadic` is not alredy in use by the language.

## Effect on ABI stability

This proposal has no effect on ABI stability.

Because a `splat-expression` does not in any way alter the signature or calling convention of functions receiving variadic parameters as arguments, there is no concern regarding use across module boundaries.

## Effect on API resilience

Likewise, the use of `#variadic()` has no impact on calling convention or signature, and thus offers no API resilience concerns.

## Alternatives considered

A number of alternatives were considered and rejected for this functionality:

### `Array<T>` to `T...` implicit conversions

This provides many of the same concerns as outlined in the `print()` example in the [**Motivation**](#motivation) section of ambiguity of use of the array's value.

### `#splat()`, `#spread()`, `#forward()` etc.

While the terms "splat" or "spread" for the operation in question are familiar to many, this is not universal. The author had never heard of the usage of "spread" for it until doing the research for this proposal. The term `forward` does not cover the full scope of the operation involved - it is valid for sending arrays to variadic parameters, not just forwarding inputs from other variadics.

In the end, "`variadic`" was settled on as:
- Simultaneously both specific enough and encompassing enough to address the feature's scope without exceeding it
- Consistent with Swift's usage of the same word to describe the very function parameters it affects
- Being just unique enough a word to give it a low false positive rate when searched for - in other words, easy for beginners to find in the docs

### Operator instead of a compiler directive

There are plenty of languages which provide the "splat" operation using an operator:

* PHP and JavaScript use the unary prefix `...` (though JavaScript calls it "spreading")
* Scala uses `_*`
* Perl, Ruby, and Python have `*` (and the latter two additionally use `**`)
* C# has the `params` keyword, which sidesteps the issue by accepting both argument lists and arrays transparently
* C has the ability to "forward" variadic parameters using `va_list`
  * however, since arrays are not first-class types in C, there isn't a real mechanism in place to perform a splat operation

However, C#'s `params` keyword would still have the overload ambiguity problem, and both Scala's `_*` and other language's use of `*` would introduce a learning barrier to a great majority of Swift users as the `*` operator has a long history in both C and Objective-C.

That would be in addition to adding an extremely narrow meaning to the symbol, which is already in use as a arithmetic operator for multiplication.

As a mental excercise, a number of other possible operators were considered, but none were sufficiently intuitive or unambiguous.

- `~array`
- `->array`
- `>>array`
- `array&`
- `^array`
- `array^`
- `\\array`
- `>array<`

In many of the pitch threads, as well threads on variadic generics, there was a strong argument in favor of reusing the `...` operator, as it provides a sense of consistency in the language. 

In considering that option, the following justifications for rejecting both the ellipsis operator, or any operator in general were expressed:

1. With existing language grammar, the ellipsis operator is unambiguous when it appears in a function declaration's `parameter-clause`

`...` as an operator may appear only in an `expression`, and a `default-argument-clause` is the only such production with a `parameter-clause`.

`...` may also appear as a suffix on a `type-annotation`.

However, in a `function-call-argument-list`, the ellipsis operator can appear almost anywhere. Even if the compiler experience no ambiguity (which has a high bar to guarantee as such), figuring out the actual effect of a `...` operator in any given call would likely be very difficult.

2. In present-day Swift, any non-restricted operator - including `...` - is subject to overloading by both the current module and any imported modules.

The use of `...` for closed and unbounded ranges is implemented this way, by providing overloads strategically in the standard library.

However, adding the "splat" operation - a compiler-time operation with a result that is inexpressible in the language itself - to `...` forces the compiler to either:

* disable the "splat" operation entirely whenever an overload is in scope (effectively making the feature's existence moot) or
* maintain additional semantics, machinery, and/or interfaces to resolve whether the user overload or the "splat" operation is invoked in any given context

3. `...` already has five use cases:
    * as a variadic function parameter
    * constructing `UnboundedRange`
    * constructing `ClosedRange` (infix)
    * constructing `PartialRangeUpTo` (prefix)
    * constructing `PartialRangeFrom` (postfix)

This is already too many meanings to assign any one operator.

It could be argued that `...` should not be used to define a variadic parameter in the first place, which leads to the next point.

4. `...` for variadic parameters dates back to before Swift was open source. It was effectively copied from (Objective-)C, at a time when the Core Team had different goals, no supporting community, and an inevitable lack of experience with how it would come to be used in the ecosystem. The fact that `...` is still used this way today is a historical artifact, one that is not only found in Swift.

Changing the language's use of `...` for defining variadic parameters would require a gargantuan effort, with little (if any) positive result.

The Swift community and maintainers should be mindful of this historical context, but that context provides little justification for ignoring the drawbacks for a mostly visual sense of language consistency.

### Another language construct

The primary problem is that the Swift language does not have many existing language constructs that can serve as a situation modifier of this nature:

- A standard library function would not only misrepresent the level at which "splat" occurs, but run afoul of the type system - which has no means of expressing "only valid where a variadic parameter goes" or "must be the only item passed to a variadic parameter" constraints
- The `@identifier` syntax is already used by property wrappers, and by built-in attributes. The splat operation is an active behavior of the compiler for call-sites; it does not describe any particular innate characteristic of the related code's definition
- `$` is already used by identifiers and property wrappers; reuse in this context would be ambiguous at best, directly conflicting at worst
- `\` also already has multiple meanings, and once again ambiguity would be quick to arise

On the other hand, each directive currently introduced by `#` describes an active, (usually) compile-time effect.

A few examples:
- OS and compilation condition detection
- Objective-C selector and keypath construction
- compile-time diagnostics

The splat operation in Swift is a decision at compile-time to avoid synthesizing an array in favor of using one already available, which seems a reasonable fit.

> \* _except in odd cases like `Self.Output == Any`_

> \*\* _The safety problems show up when functions like `os_log()` are imported from C._
>
> _When an `Array<T>` is passed to a variadic function which takes type `CVarArg...`, it transparently bridges over as a single argument. Thus, if a caller passes the `args` array to `os_log()`, all the function sees is a single C pointer, not an array of arguments._
>
> _The compiler will emit no warning, error, or otherwise, and the most likely result will be a crashed process._
>
> _By convention, most C functions using the `va_list` mechanism have a "`v`" variant - such as `vprintf()` or `NSLogv()` - which accepts the `CVaListPointer` available via `withVaList(args)`, but `os_log()` does not._
>
> _As a result, it is nearly impossible to use it safely from Swift without the ability to forward the variadic parameter._

_Note: It's probably just as well that `os_log()` and its ilk are left with larger concerns than the proverbial arrays of Hanoi. Without bigger problems to tackle, someone would probably try to make `gyb` an official part of the language or run Swift through `cpp(1)`._
