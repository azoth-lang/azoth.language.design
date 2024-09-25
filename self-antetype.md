# Self Antetype Design Notes

Supporting a "Self Type" or "This Type" is an important part of the plan for the Azoth language.
However, it is now clear that there isn't a ready made answer to how this should work. It seems an
obvious feature to add given the [curiously recurring template pattern
(CRTP)](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern), Rust's self type, and
that Swift has some self type support. However, there seems to be no worked out answer for how to
handle self types in a language with subtyping.

This document is to review, explore the design space and hopefully come up with a design for self
antetypes in Azoth. The term "antetype" is used within Azoth to refer to a type without a reference
capability attached. There are questions about whether self types should have reference capabilities
inherent to them the way generic parameter types do or not. However, trying to deal with that at the
same time as trying to figure out the basics of self types seems very difficult. To hopefully break
down and simplify the problem, self antetypes will first be tackled. That is, how should something
like self types work an a version of Azoth that doesn't have reference capabilities and there for
has a type system much more akin to the type system of C#, Java, Swift, Kotlin, or even Scala? It is
hoped that once that is worked out, it will be easier to determine how that feature should work with
reference capabilities.

This document assumes a reasonable familiarly with what the concept of a self type is and how type
systems work. However, even without that if you read all the sections, it should become clear.

## Initial Issues

A self type is something like a type parameter or alias that refers to the concrete type of the
current object. The `self` reference is of that type. When used as a parameter or return type, this
requires subtypes to use a more specific `Self` type because they know that the `Self` type or more
narrowly constrained.

What are the issues that occur upon trying to introduce and use such a thing in a language with
subtyping?

1. *Constructing a Value of the Type*: When a method returns `Self`, how does one implement the body
   of the method? One can return `self`, but otherwise one must somehow get another value that is
   known to be of that type. One way to do that would be to construct or factory another instance of
   the `Self` type. How is this done?
2. *Using `Self` as a Parameter Type Seems in Tension with Subtyping*: If a method has a parameter
   of type `Self`, then a subtype needs that overloads the method seemingly expects the method to
   take a more specific type as a parameter. But that is in conflict with the fact that subtype
   method parameters must be contravariant (i.e. have less specific parameter types.) Hint: maybe we
   need to be stricter about how these methods are called and know at the call site that the
   parameter types are exactly equal, not just both subtypes of the same supertype.
3. *Is a More Specific Type Actually Desired in Subtypes?*: In use cases like equality and
   operators, it isn't even clear that having subclasses have a more specific parameter type is what
   is desired.

## Prior Art

Here we review existing examples of self types in languages or academic research.

### Swift Self Type

The Swift language has a `Self` type. The official documentation has a [surprisingly short section
on the self
type](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/types/#Self-Type).
It starts by saying "The `Self` type isn’t a specific type, but rather lets you conveniently refer
to the current type without repeating or knowing that type’s name." So it is seemingly describing
some kind of type variable or parameter. Yet it makes it sound like maybe it is just some convenient
shorthand? We'll see later it is a true type parameter. The behavior of `Self` in Swift depends on
where it is used.

In a protocol declaration or a protocol member declaration, the `Self` type refers to the "eventual
type that conforms to the protocol." Other sources confirm this means the class that implements the
protocol, not subclasses of that class. Testing in swiftfiddle.com confirms that. This shows that
when used as a parameter type, the `Self` type isn't a true `Self` type that carries down to the
most concrete class.

```swift
import Foundation

protocol ExampleProtocol {
    func test(of value: Self)
}

protocol AnotherProtocol: ExampleProtocol{}

class Base: AnotherProtocol {
    // ERROR on `Self`: "covariant 'Self' or 'Self?' can only appear as the type of a property,
    // subscript or method result; did you mean 'Base'?"
    // func test(value: Self) {}
    func test(of value: Base) {
      print("Base.test")
    }
}

class Derived: Base {
    // Compile errors for trying to use `Self` or `Derived` as the parameter type
    override func test(of value: Base) {
      print("Derived.test")
    }
}

var a = Base()
var b = Derived()
a.test(of: a)
b.test(of: b)
b.test(of: a)
```

Note how even though the `Self` type is used in ExampleProtocol it doesn't take on the type of
`AnotherProtocol` but instead of `Base`. This makes the class the "real" thing and the protocols are
second-class.

Testing shows that unlike when `Self` is used for a parameter type, when it is used as a return type
in a protocol, it requires that the conforming class use `Self` as the return type of the method.
This is seemingly in contradiction to the documentation's statement that it refers to the type that
conforms to the protocol.

```swift
import Foundation

protocol ExampleProtocol {
    func test() -> Self
}

class Base: ExampleProtocol {
    // ERROR: method 'test()' in non-final class 'Base' must return 'Self' to conform to protocol 'ExampleProtocol'
    // func test(value: Base) {}
    func test() -> Self {
      print("Base.test")
      return self;
    }
}
```

The behavior in classes is different. In a class the `Self` type "refers to the type introduced by
the declaration." That is again overly vague. The documentation further states that it is restricted
to appear in a return position or inside the body of the member. When overriding a method returning
`Self`, the compiler requires that the return type of the overriding method be `Self`.

```swift
import Foundation

class Base {
    func test() -> Self {
      print("Base.test")
      return self;
    }
}

class Derived: Base {
    // ERROR: "cannot override a Self return type with a non-Self return type"
    // override func test() -> Base {
    override func test() -> Self {
      print("Derived.test")
      return self;
    }
}
```

Furthermore, `Self` is not simply a synonym for the current type because in the code above adding
the line `var t: Self = Base();` to the base class `test()` method gives the error "`cannot convert
value of type 'Base' to the specified type 'Self'`". It seems that within classes, the `Self` type
acts as something like a an associated type that is constrained by the current type (i.e. `Self <:
Base` in Base). Because it is covariant, it is disallowed in parameter position.

Further evidence that the `Self` type is essentially an associated type is provided by the
[existential any](https://www.hackingwithswift.com/swift/5.6/existential-any) feature that was added
to Swift. Swift protocols act as existential types by default because their associated types are
unknown. To reflect that, they added the requirement that they be prefixed with the keyword `any` to
indicate that an existential type is involved. As of Swift 5.10 any protocol using `Self` must be
prefixed with `any` when used as a type. (Testing shows that for unknown reasons, this is not
necessary when `Self` is only used in return position.) Although the plan is that in Swift 6, all
protocols will have to be. The requirement for `any` makes it clear that Swift is using an
existential type to fulfill the associated `Self` type of a protocol.

```swift
import Foundation

protocol Test {}

protocol SelfTest{
    func test(x: Self)
}

class Base: Test, SelfTest {
    required init() { }
    func test(x: Base) { }
}

var a: Test = Base();
// ERROR: use of 'SelfTest' as a type must be written 'any SelfTest'
var b: SelfTest = Base();
```

There is an additional feature of Swift that is relevant to self types even though it is generally
not documented as such. When writing a method returning `Self` it is necessary to have a value that
is of the associated type `Self`. The `self` reference is. But no other reference is. Suppose one
wished to return a new instance instead. To do so, one might call another method returning `Self` to
construct that method. But it would have the same problem in its implementation. In Swift,
initializers can be essentially virtual methods on the metatype object. This is used to allow for
example, the calling of initializers on generic parameter types. But it can also be used to
construct a value of type `Self`. By default, initializers do not have to be overridden. However, an
initializer can be marked `required` and it must then be overridden in each subclass. The compiler
can thus guarantee that the initializer can be called for any subclass and produce a value of type
`Self`. So a method returning `Self` can call required initializers with `Self.init(...)` to create
new values of type `Self`.

### Scala Self Types

### Scala This Type Plans

### Rust Self Types

### "ThisType for Object-Oriented Languages: From Theory to Practice"

By Sukyoung Ryu

## Framing

### Bounded Associated Type

### Return Position Flavors

### Parameter Position Flavors

## Ideas

### Virtual Constructors

### Same Type Test

### Implemented "Abstract"

A method with a body that still must be overridden in subclasses

## Use Cases

### Clone

### Copy Constructors

### Equality and Comparison

### Abstract Domains with Multiple Implementations?

### Proxies

### Doubly Linked List

A doubly linked list that inherits from a singly linked list since a doubly linked one can be traversed like a doubly linked one (Example from *On Binary Methods*)
