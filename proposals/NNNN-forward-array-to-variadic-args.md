# Support passing `Array<T>` to parameters of type `T...`

* Proposal: [SE-NNNN](NNNN-forward-array-to-variadic-args.md)
* Authors: [Gwynne Raskind](https://github.com/gwynne)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-128 Pass array to variadic function](https://bugs.swift.org/browse/SR-128)
* Previous Revision: None

## Introduction

A variadic function parameter of type `T...` is available in the function body as a value of type `[T]`, but callers of such functions may not pass a value that is already an array. There are many situations where alternatives to "splatting" an array in this fashion are either slow, convoluted, or entirely unavailable. This proposal provides syntax for performing the "splat" operation in a manner consistent with existing behavior.

Swift-evolution thread: [TBD: Discussion thread topic](https://forums.swift.org/)

## Motivation

Consider `print()`'s three-argument form:

```swift
public func print(_ items: Any..., separator: String = " ", terminator: String = "\n")
```

A developer with an array of items they wish to output through this function signature can not simply pass the array as an input:

```swift
  1> print(1, 2, 3, 4, separator: "|")
1|2|3|4
  2> print([1, 2, 3, 4], separator: "|")
[1, 2, 3, 4]
  3>
```

To get the desired result, the developer must do more work manually:

```swift
  3> print([1, 2, 3, 4].map { String($0) }.joined(separator: "|"))
1|2|3|4
```

In `print()`'s case, this limitation is annoying and perhaps a little inefficient to get around, but that's all. But in other cases, it becomes more serious. `Combine` deals with the inability to pass an array to a variadic parameter by declaring both forms of some methods:

```swift
extension Publisher {
    public func append(_ elements: Self.Output...) ->
        Publishers.Concatenate<Self, Publishers.Sequence<[Self.Output], Self.Failure>>

    public func append<S>(_ elements: S) ->
        Publishers.Concatenate<Self, Publishers.Sequence<S, Self.Failure>>
        where S : Sequence, Self.Output == S.Element
}
```

While the `Self.Output == S.Element` constraint on the second method makes it relatively safe to provide both forms of input in this case (except in odd cases like `Self.Output == Any`), the duplication remains a maintenance burden and potential source of confusion. And sometimes, generic constraints can't provide even partial safety - consider `Foundation.NSArray`:

```swift
extension NSArray {
    public convenience init(objects elements: Any...)
    @nonobjc public convenience init(array anArray: NSArray)
}
```

The legacy design of `NSArray`'s Objective-C API accidentally avoided what would otherwise be an obvious problem when creating an array of arrays, thanks to the `array` label on the second initializer, but lacking that, the use of `Any` as the only possible element type would be an accident waiting to happen. An extra burden in maintenance and API bloat is once again imposed (if only theoretically in this particular instance).

There are even cases where functionality is limited or even completely unavailable - for example, it's currently impossible to "forward" conformance to the `ExpressibleByArrayLiteral` and `ExpressibleByDictionaryLiteral` protocols, as seen here:

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

It can be sort-of faked by writing out individual calls to the other type's initializer based on the number of elements in the input arrays, but at a major cost to clarity and maintainability, and it scales _very_ poorly. Another option is to require separate `init(forwardedArrayLiteral: [Self.ArrayLiteralElement])` and `init(forwardedDictionaryLiteral: [(Self.Key, Self.Value)])` methods in a protocol the target object must conform to, but that just pushes the problem out onto anything that ever uses the wrapper object.

And finally, there is this truly unfortunate example, which stands as the only working - but still not safe! - option for wrapping `os_log()` in Swift at the time of this writing:

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

This is unsafe because when functions like `os_log()` are imported from C, there are larger concerns than the "arrays of Hanoi": `Array` bridges transparently as a `CVarArg`. If a caller tries to pass the `args` array to `os_log()`, it would be interpreted as a single argument and bridged as a C pointer. The compiler will emit no complaint or even warning, and the most likely result is a crashed process. By convention, most C functions using the `va_list` mechanism have a "`v`" variant - such as `vprintf()` or `NSLogv()` - which accepts the `CVaListPointer` available via `withVaList(args)`, but `os_log()` does not. As a result, it's nearly impossible to use it safely from Swift without the ability to forward the variadic parameter.

## Proposed solution

As there is clear value to the ability to forward variadic argument lists to other functions with compatibly-typed variadic parameters, and as this is not meaningfully distinct from the ability to instruct the compiler to distribute a compatible array into a variadic parameter list, this proposal proposes the "splat" operation, with the following syntax:

```swift
func my_os_log(_ message: StaticString, log: OSLog = .default, type: OSLogType = .default, _ args: CVarArg...) {
    os_log(message, log: log, type: type, #variadic(args))
}
```

The `#variadic()` directive performs a one-way, zero-cost, compile-time reinterpretation of an `Array<T>` into a `T...`. This value can exist only as the sole argument to a variadic parameter of identical type; it has no concrete type of its own and can't be stored or captured. In essence, the compiler is told to just use the given array as the "real" parameter instead of faking or initializing one from the argument list.

See the Alternatives Considered section for a detailed discussion of why a compiler directive was chosen as the proposed syntax. tl;dr version: Everything else was either confusing, already overused, or inappropriate to the task.

## Detailed design

This proposal introduces a single additional compiler directive with the name `#variadic`. The directive has a simple syntax: It takes a single array argument as if it were a function, and has no directly usable return value. It is syntactically valid only as a function call argument. Grammatically, `function-call-argument` is amended to include the following additional productions:

```
function-call-argument → splat-expression | identifier : splat-expression
splat-expression → #variadic(expression)
````

Please note that `splat-expression` is **NOT** added to any productions for prefix, binary, primary, or postfix expressions. It is syntactically invalid anywhere outside a `function-call-argument`; in this respect the name of the production is somewhat misleading thanks to it semantic value. (For additional confusion, the `expression` _inside_ the `splat-expression` production **IS** a normal expression, and can do most things any other expression can do.)

A `splat-expression` is _semantically_ valid only when ALL of the following conditions are true:

- It is syntactically valid as per the above grammar productions.
- The complete `parameter-list` of the function includes exactly one `type-annotation` possessed of the `...` specifier identifying a variadic parameter. (Note: It is already a rule of the grammar that a function may have at most one variadic parameter.)
- The `splat-expression` appears in the same ordinal position as the variadic parameter, and with a matching external parameter name, if any.
- The `expression` interior to the `splat-expression` has a resolved type of `Swift.Array<Element>`, such that the resolved type of `Element` is either a) identical to or b) trivially expressible as the declared type of the variadic parameter.

In practical terms, a splat expression provides a narrowly-defined method of invoking the otherwise hidden "real" signature of a variadic function - e.g. the function is treated for the purposes of the splat expression (and _only_ the splat expression) as if the type of its variadic parameter was `Array<T>` rather than `T...`, passing the provided array as the variadic argument rather than invoking an array initializer over the individual arguments. The splat expression can even be defined in terms of simply substituting a preexisting array for the implicit array initialization that the compiler would otherwise synthesize for a variadic parameter.

The following various syntactical, grammatical, and semantic rules apply to splat expressions, and make reference to the following code snippet:

```swift
typealias Value = /* any type */;

func f<Value>(input: [Value])  { g(param: #variadic(input.map { $0 })) }
func g<Value>(param: Value...) { /* function body */ }

func h<Value>(input: [Value])  { i(param: input.map { $0 }) }
func i<Value>(param: [Value])  { /* function body */ }
```

- The body of function `f()` is semantically valid if and only if the body of function `h()` would also be semantically valid for any particular definition of `Value`. The snippet also demonstrates the property that the bodies of functions `f()` and `h()`, and of the functions `g()` and `i()`, are respectively semantically identical.
- Conversely, the compiler **MUST NOT** accept the use of a `splat-expression` in the place of parameter whose type is already an array. Given the above snippet, the function call expression `i(param: #variadic(input))` would be ill-formed.
- The compiler also **MUST NOT** accept a `splat-expression` if other arguments matching the variadic parameter in position are also present. For example, the call `g(param: input.first!, #variadic(input))` would also be ill-formed.
- If the type `T` of the function's variadic parameter is an optional, whether normal or implicitly unwrapped, the type of the provided array's elements must still match accordingly.
- A splat expression's body is permitted to throw errors as the lexical context otherwise permits. Any required `try-operator` must appear as part of the inner `expression`, e.g.: `#variadic(try /* everything */)`.
- It is ill-formed for the evaluated type of a `splat-expression` to be anything other than `Array<T>`.
- It is ill-formed for the type of the array itself to be e.g. `Optional<Array<T>>`, even if `T` is itself optional.
- It is ill-formed for `#variadic()` to appear in any lexical structure other than a function call argument.
- The construction `try #variadic(/* etc. */)` is ill-formed.
- At no time does any usage, or lack thereof, of the splat expression in any way augment, supplement, limit or otherwise alter the signature of the called function.
- The compiler **MUST NOT** impose any form of syntactic or semantic difference between the received value of a variadic parameter (e.g. the variable `param` in the functions `g()` and `i()` above) and any other array, as in regards to a splat expression. (Note: It is believed that this is already true for the general case, but this proposal refrains from explicitly stating such a restriction outside the scope of the syntax with which it is concerned.)

## Source compatibility

This proposal is purely additive. The `#` symbol is not valid for general use by Swift programs, and the identifier `#variadic` is not alredy in use by the language. Thus, no source compatibility issues exist.

## Effect on ABI stability

This proposal has no effect on ABI stability. Because the `splat-expression` does not in any way alter the signature or calling convention of functions receiving variadic parameters as arguments, there is no concern regarding use across module boundaries.

## Effect on API resilience

Likewise, the use of `#variadic()` has no impact on calling convention or signature, and thus offers no API resilience concerns.

## Alternatives considered

A number of alternatives were considered and rejected for this functionality:

### Ask library authors to provide additional overloads which take an explicit array in place of a variadic parameter

At first blush this was not only a sensible-seeming solution but also a viable workaround without needing a language feature. Unfortunately, it falls apart almost immediately due to overload resolution ambiguity:

```swift
func multizip<T>(arrays: T...) -> [T] where T: Collection { multizip(arrays: arrays) }
func multizip<T>(arrays: [T]) -> [T] where T: Collection { /* body */ }

// Which version is invoked? Is this "zip two [String]" or "zip one [[String]]"?
multizip([["a"], ["b"]])
```

Even if the ambiguity could be resolved, such as by removing the variadic variant entirely (which would negate the value of the existing language feature altogether) or constraining the type of `T.Element`, it is often not an option to even try. As was seen eariler, `os_log()` is provided by the OS and can not be augmented or avoided by user code. Similar issues will be encountered by any code importing `CVarArg`-using functions from C/Objective-C unless they have `v` variants. `print()` is yet another clear example - the parameter of `print()` and `debugPrint()` is, by design, `Any`; constraining it leaves it effectively useless..

### Allow `Array<T>` to implicitly convert to `T...`

This is just another version of explicitly providing an overload, and suffers from exactly the same ambigutity problems.

### Provide N overloads taking 0...N parameters each

This is the approach taken by the `os_log()` example back in the Motivation section. This obviously quickly becomes miserable to maintain even with such tools as `gyb`, and has no hope of scaling past a couple dozen elements at best. For good measure, it can also - depending on the types of the values involved - suffer from ambiguity issues, though less often than the array-taking overload.

### Use <some language's> operator for the splat operation instead of a compiler directive

There are plenty of languages which provide the "splat" operation using an operator. PHP and JavaScript use unary prefix `...`, though JavaScript calls it "spreading". Scala uses `_*`. Perl, Ruby, and Python have `*` (and the latter two additionally use `**`).

For good measure, C# has the `params` keyword which sidesteps the issue by accepting both arguments lists and arrays transparently, and C has an exceedingly awkward ability to "forward" variadic parameters using `va_list` (though since arrays are not first-class types in C, there's doesn't really exist anything on which to perform a splat operation anyhow).

- The `...` operator already has multiple meanings in Swift - in particular for specifying `ClosedRange`s and `UnboundedRange`s, not to mention of course variadic parameters themselves. It would not be practical to assign it yet another meaning.
- The `*` operator has a long, sordid history in C and Objective-C, and would doubtlessly be confusing at best to many Swift developers. There is also a potential ambiguity with vector operations and of course the multiplication operator. Nor does it necessarily suggest the splat operation at first or even second glance.
- Scala's `_*` operator suffers from the same confusion issue.
- C#'s keyword would have the overload ambiguity problem mentioned above

The author considered a number of other possible operators as well, none of which could be considered sufficiently intuitive or unambiguous. Here's just a few of the discarded options, several of them more in jest than anything else:

- `~array`
- `->array`
- `>>array`
- `array&`
- `^array`
- `array^`
- `\\array`
- `>array<`

### Call the directive `#splat()`, `#spread()`, `#forward()` etc.

While the terms "splat" or "spread" for the operation in question are familiar to many, this is not universal. The author had never heard of the usage of "spread" for it until doing the research for this proposal. The term `forward` does not cover the full scope of the operation involved - it is valid for sending arrays to variadic parameters, not just forwarding inputs from other variadics.

In the end, "`variadic`" was settled on as being simultaneously both specific enough and encompassing enough to address the feature's scope without exceeding it. It is consistent with the language's own usage of "variadic" to describe such parameters. Also considered was that it provides an unambiguous term for those unfamiliar with the concept to look up, and stand a better than even chance of being accurately directed to relevant documentation.

### Use <some other construct> than a compiler directive

There are not many other language constructs which would be suitable for a situational modifier of this nature.

- A standard library function would not tend to correctly convey the level at which the consideration is applied, and the type system has no means to express the "only valid where a variadic parameter goes" or "must be the only item passed to a variadic parameter" constraints.
- The `@identiifer` syntax is already used by property wrappers, and by built-in attributes. The splat operation is an active behavior of the compiler; it does not describe any particular innate characteristic of the related code.
- `$` is already used by identifiers and property wrappers; reuse in this context would be ambiguous at best, directly conflicting at worst.
- `\` also already has multiple meanings, and once again ambiguity would be quick to arise.

On the other hand, each directive currently introduced by `#` describes an active, (usually) compile-time effect - OS and compilation condition detection, Objective-C selector and keypath construction, and compile-time diagnostics, to name a few. The splat operation in Swift is a decision at compile-time to avoid synthesizing an array in favor of using one already available, which seems a reasonable fit.

