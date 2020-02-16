# Property wrapper requirements in protocols

* Proposal: [SE-NNNN](NNNN-property-wrapper-protocol-requirements.md)
* Authors: [Gwynne Raskind](https://github.com/gwynne), [Tanner Nelson](https://github.com/tanner0101)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: **Awaiting implementation**

## Introduction

It is sometimes desirable for a protocol to require that a conforming type use a property wrapper in the declaration of a required property. This allows consumers of the protocol to access the wrapper's projected value and properties, not just the wrapped value.

## Motivation

Currently the only way to generically access the projected value of a property wrapper is to use `Mirror` to reflect the containing object and cast the appropriately-labeled descendant to the property wrapper's type:

```swift
extension Model {
    var _$id: ID<IDValue> {
        self.anyID as! ID<IDValue>
    }

    var anyID: AnyID {
        guard let id = Mirror(reflecting: self).descendant("_id") as? AnyID else {
            fatalError("id property must be declared using @ID")
        }
        return id
    }
}
```

This approach turns out to be extremely slow at runtime, and requires a runtime check and use of `fatalError()` to verify a requirement the compiler could otherwise have enforced.

## Proposed solution

We propose allowing a subset of property wrapper syntax to appear in protocol declarations:

```swift
protocol Model {
    associatedtype IDValue: Codable, Hashable

    @ID var id: IDValue? { get set }
}
```

Only property wrapper names may appear in a protocol declaration, and initializer parameters are not permitted. A property wrapper attached to a protocol requirement has two effects:

1. Conforming types must use the specified property wrapper on their declaration of the property.
1. If the property wrapper has a `projectedValue`, the protocol provides access to it via the usual `$` prefix.

This has the effect of making the projected value of the wrapper available via the protocol, without affecting how backing storage works. For this proposal, we do not consider specification of nested property wrappers in a protocol

## Detailed design

Given the above declarations, the effect of the `@ID` wrapper on the protocol requirement is roughly semantically equivalent to:

```swift
@propertyWrapper
class ID<T: Codable & Hashable> {
    var wrappedValue: T
    var projectedValue: Self { self }
}

protocol Model {
    associatedtype IDValue: Codable, Hashable

    var _id: ID<IDValue?> { /* private */ }
    var id: IDValue? { get set }
    var $id: ID<IDValue?> { get }
}
```

The implicit backing store property does not exist as an actual protocol requirement; it serves the semantic purpose in this design of forcing the use of the property wrapper by conforming types even if the wrapper offers no projected value. In the case that the wrapper does have a projected value, the implicit `$`-prefixed property _is_ a strict requirment of the protocol. Conforming types may not provide an actual declaration of the projection property; it is provided by the property wrapper; its presence as a requirement of the protocol is for the specific purpose of exposing the projection via the protocol. The presence or absence of a setter on the implicit requirement is determined entirely by the property wrapper's declaration and can not be influenced by the declaration in the protocol (e.g. a `get`-only property may still have a settable projected value and vice versa).

## Source compatibility

No source compatibility issue is raised by this proposal; the proposed syntax is purely additive and was previously a compile error.

## Effect on ABI stability

The authors are aware of no ABI stability concerns raised by this proposal.

## Effect on API resilience

The authors are aware of no API resilience concerns raised by this proposal.

## Alternatives considered

No alternative designs for this functionality have yet been considered or proposed at the time of this writing.
