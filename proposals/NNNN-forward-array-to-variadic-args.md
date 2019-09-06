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

Or even this unfortunate example, which would be absurd in the context of `print()` but is the only option for correctly and safely wrapping `os_log()` in Swift at the time of this writing:

```swift
  4> // Note: This example doesn't add anything to os_log() that would not be
  5> // possible by calling the original imported C function directly.
  6> func my_os_log(_ message: StaticString, log: OSLog = .default, type: OSLogType = .default, _ args: CVarArg...) {
  7.     switch args.count {
  8.     case 0: os_log(message, log: log, type: type)
  9.     case 1: os_log(message, log: log, type: type, args[0])
 10.     case 2: os_log(message, log: log, type: type, args[0], args[1])
 11.     case 3: os_log(message, log: log, type: type, args[0], args[1], args[2])
 12.     case 4: os_log(message, log: log, type: type, args[0], args[1], args[2], args[3])
 13.     case 5: os_log(message, log: log, type: type, args[0], args[1], args[2], args[3], args[4])
 14.     default: fatalError("Can only handle five format specifiers.")
 15.     }
 16. }
 17>
```

Even more problematically for this particular usage, `Array` bridges transparently as a `CVarArg`. If a caller _does_ attempt to pass an `Array<CVarArg>` as the `args` parameter, it would be interpreted as a single argument bridged as a C pointer. The compiler will emit no complaint or even warning, and the most likely result is a crashed process.

## Proposed solution

As there is clear value to the ability to forward variadic argument lists to other functions with compatibly-typed variadic parameters, and as this is not meaningfully distinct from the ability to instruct the compiler to distribute a compatible array into a variadic parameter list, this proposal proposes the "splat" operation, with the following syntax:

```swift
func my_os_log(_ message: StaticString, log: OSLog = .default, type: OSLogType = .default, _ args: CVarArg...) {
    os_log(message, log: log, type: type, #variadic(args))
}
```

The `#variadic()` directive performs a one-way transformation of an `Array<T>` into a `T...`. This value can only exist as the sole argument to a variadic parameter of identical type. It has no concrete type of its own and can not be stored or captured. See the Alternatives Considered section for a discussion of why a compiler directive was chosen as the proposed syntax.

## Detailed design

This proposal introduces a single additional compiler directive with the name `#variadic`. The directive has the same syntax as `#selector()`: a function-like invocation taking a single parameter. `#variadic` does not return a value, and is syntactically valid only as a function call argument. Grammatically, it is described simply as follows:

```
splat-expression → #variadic(expression)
````

The grammar of `function-call-argument` is additionally ammended to include the following production:

```
function-call-argument → splat-expression | identifier : splat-expression
```

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

Even if the ambiguity could be resolved, such as by removing the variadic variant entirely (which would negate the value of the existing language feature altogether), this is often not possible. `os_log()`, for example, is provided by the OS and can not be augmented or avoided by user code. Similar issues will be encountered for any code using `CVarArg` (generally things imported from C/Objective-C). `print()` is yet another immediate example.

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
