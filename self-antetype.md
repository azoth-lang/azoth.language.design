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

I am not sure what a consistent mental model of `Self` types is supposed to be. There seem to be
three independent aspects in play.

1. In protocols, it is an associated type referring to the class or struct that implements the
   protocol.
2. In classes and structs, it is an associated type referring to the true type of the instance.
   `Self` cannot be used in an input context (e.g. as a method parameter type).
3. A bridge rule requires that a method returning `Self` in a protocol must be implemented by a
   class method returning `Self`.

The consistency between `Self` in protocols and classes is that they are associated types
constrained to be a subtype of the current type that will be automatically further constrained or
set by subtypes. A major issue with the behavior of `Self` in protocols is that it makes a clear
behavioral difference between protocols and classes or structs. As in C#, it treats the protocols as
second class. It makes the first class or struct that implements the protocol special. I've
described this as treating the class or struct type as "real" and the protocol as a convention or
category.

Interesting Observation: Swift does not equate `Self` to the current type when the class is final.

```swift
import Foundation

class Base {
  public func test() -> Self {
    return self;
  }
}

final class Derived: Base {
  public override func test() -> Self {
    return Derived();
  }
}
```

After thinking about Swift's design longer, I think that the documentation describes it in a
misleading way. It is more proper to think of it as two things. When used in the return type, it is
a covariant type variable equal to the type of the current instance. When used in the parameters of
a protocol method, it is an invariant associated type that will be set to the class or struct
implementing the protocol. That is the proper way to think about Swift `Self` type. This eliminates
the need for the strange bridge between protocols and classes described above.

### Scala Self Types

In Scala there is a feature called "[self types](https://docs.scala-lang.org/tour/self-types.html)".
It isn't quite a self type as being discussed here, but it has interesting parallels and is the
context for what will be a true self type in Scala. So it is worth understanding.

Self types are used in traits and enforce a constraint that all implementers of the trait must also
implement another trait. This is very similar to, but distinct from, having a trait inherit from
another trait. The distinction is subtle and I won't try to understand the benefits right now.

As the [language designer states](https://github.com/scala/scala3/issues/7374), have a non-obvious
syntax. Within a trait, an identifier and another trait are given followed by `=>` and within the
scope after the arrow, the identifier is a reference to `this` with the given type (e.g. `self:
SomeTrait => ...`). In place of a regular identifier, `this` can be used to that `this` has a
changed type withing the scope (e.g. `this: SomeTrait => ...`). As the language designer states,
self types have two effects.

1. Restrict the type of `this` to a proper subtype of the current class
2. Introduce an alias name for this.

While Scala "self types" are not proper self types, they show an interesting and possibly desirable
feature. Namely, being able to further constraint the `Self` type with some other trait. The next
section further explores this.

### Scala Singleton Types and `this.type`

Scala has a feature called singleton types. A singleton type is a type which contains only a single
value (see [There are more types than
classes](https://typelevel.org/blog/2017/02/13/more-types-than-classes.html)). The syntax for
singleton types is `p.type` where `p` is basically a reference. The singleton type `this.type` can
be used as the return type of a method. However, it isn't a proper self type because "[there is only
a single value that you can return when the return type is `this.value` and that value is
`this`](https://users.scala-lang.org/t/guarantees-of-this-type/7822/3)". Actually, you can return
`null` too. The Scala FAQ addresses the question of "[How can a method in a superclass return a
value of the “current”
type?](https://docs.scala-lang.org/tutorials/FAQ/index.html#how-can-a-method-in-a-superclass-return-a-value-of-the-current-type)"
and clearly states that `this.type` isn't it. It also states there is no direct support for it.

The FAQ links to [Returning the "Current" Type in
Scala](https://tpolecat.github.io/2015/04/29/f-bounds.html) and [Advantages of F-bounded
polymorphism over typeclass for return-current-type
problem](https://stackoverflow.com/q/59813323/268898). The introduction of the former says that it
is a common problem and frequently asked (as evidenced by it being in the FAQ).

### Scala This Type Plans

The designer of Scala proposes replacing the "self type" feature described in the last section with
a [proper self type](https://github.com/scala/scala3/issues/7374) they are calling This type (Since
Scala uses `this` instead of `self`). He lays out a very clear plan for how they would work:

> * Every class or trait `C` has an implicit `This` [associated] type declaration.
>   * if `C` is final, the declaration is `type This = C`
>   * otherwise the declaration is `type This <: C`
>   * in an object o, the declaration is `type This = o.type`.
>
> * The type of `this` is `This`.
>
> * `This` types may also be declared explicitly.
>   * The only legal form of such a declaration is with an upper bound: `type This <: B`.
>   * The upper bound is joined with the implicit `This` type declaration. I.e. in class `C`, an
> explicit declaration `type This <: B` would yield `type This <: B & C` as the final type of `This`.
>
> Once `This` types are introduced, self type declarations are redundant and can be dropped. A
> definition like
>
> ```scala
> class C { this: T => ... }
> ```
>
> would be replaced by
>
> ```scala
> class C { type This <: T; ... }
> ```

And later on:

> There is one restriction we have to add: In a `new C`, the type of `This` is also implicitly set
> to `C`. So `new C` is treated the same as `new C { }` for the purposes of `This` checking. `new
> C{}` in turn expands to `new C{ type This = C }`. So if a base class has another idea of what
> `This` is this would give an error. That's how we prevent illegal extensions.

There is no discussion of how this would interact with using `This` as a parameter type or how one
could construct other values with the type `This`. This feature appears to have been ignored for a
long time now.

### Rust Self Types

Rust has a [`Self type`](https://doc.rust-lang.org/reference/paths.html#self-1) that in traits
refers to the struct type that implements the trait and in other contexts is just a synonym for the
current type. It seems to be a fairly simply thing in Rust and not a source of much complexity. It
was hard to even find discussion of it online. I believe that this is because Rust doesn't have
subtyping for structs. Thus, there is no ambiguity in what `Self` should refer to. Compare it to
Swift where in protocols `Self` refers to the class or struct that conforms to the protocol but in
classes and structs, it refers to the most concrete type of an instance. Those two behaviors
collapse to the same thing when there is no subtyping between structs. Also, in Rust, traits are
clearly categories of structs. That works better in Rust because of its low-level focus and
references to trait typed values are comparatively rare. Instead, everything is generic over structs
and uses traits to constrain that. Indeed, even having a reference to a value that is trait type
requires the use of the `dyn` keyword.

### ThisType for Object-Oriented Languages: From Theory to Practice

The paper *`ThisType` for Object-Oriented Languages: From Theory to Practice* by Sukyoung Ryu
tackles the challenges of a self type. It refers to them as `ThisType`s. The paper first describes
the problems and use cases and then proposes solutions at both the type theory and practical
languages levels. Here, I'll focus on the language level proposals and discussions.

In this formulation, self types are the type of the special `self` reference. However, if that were
simply equivalent to the type of the enclosing class of the method then the `Self` type would be
unsound. They note that previous researchers suggested two restrictions that prevent the
unsoundness.

> 1. Disable subtyping between a type and the type of its superclass when the superclass provides
methods whose types include [input] occurrences of a `This` type.
> 2. Disallow a binary method invocation when the compile-time type of the receiver is not an *exact
class type*, which is the declared type and not any of its strict subtypes.

The first option is far too restrictive and disallows subclassing anytime `Self` is used as a
parameter type. Basically limiting them to what `Rust` supports. However, since in most object
oriented languages, it is valid for one trait to implement another trait and for those to be used as
types, I think this would effectively outlaw the use of `Self` types in parameters to methods in
traits. While I don't fully understand the second, they argue and give examples (which if correct)
show that it is also to restrictive.

To address these limitations, to support more methods with `ThisType`d formal parameters, they
enhance the type system with *exact type capture*, *named wildcards*, and *exact type inference*,
and also introduce *runtime type matching*. To support more methods with `ThisType`d results, they
provide *virtual constructors*. Let us consider each of those.

In their terminology, exact object types are "record types" while inexact object types are
"existential record types". As best I can tel from a quick reading of the discussion of the theory,
they replace the standard types in a language which would normally be exact with inexact ones that
are existential. This mirrors how directly using a protocol type in Swift is an existential type.
The record types then become the exact types and may not have a subtyping relationship with each
other. Instead, they substitute a "specializes" relationship between the record types. The exact
record types are not subtypes of each other, but they are subtypes of the existential types. (Aside:
it is probably a coincidence but it is strange how strongly that mirrors Azoth idea that there is a
trait type introduced by each class but logically, there is also a true type for the class which is
apparent as the type of `self`. That is because `self` has access to private members that no other
reference has access to. Does that mean that you could access the private members of another
instance if you could determine that it had the same exact type?)

With that background, they introduce *exact class types*. For a class `C` the exact class type is
written `#C`. Note that this and other types will in fact be allowed in code. Exact class types can
be used in Java instance of tests and for casting. For these:

* An object creation expression `new C(...)` has type `#C`.
* `#C <: C`, but *not* vice versa.
* `#C <: #C` and `null <: #C` but `#C` is a supertype of no other types.

They introduce a special type variable `This` that can be regarded as an implicitly declared type
variable in the class. A `This` type appearing in the definition of a class (or interface) `C`
denotes the runtime type of the special variable `this`.

* `This <: C`, but *not* vice versa
* `This <: This` and `null <: This` but `This` is a supertype of no other types. It is not a
  supertype of `C` (or `#C`). To see why, note that `this` can be bound to an object of a subclass
  of `C` at runtime.

Finally, they introduce *exact type parameterization* so that `C</X/>` denotes the type of an object
*that has a declared class type `C` and a runtime exact type `X`*. A type `C</X/>` is well formed if
`X` represents an exact type `#D` where `D` is `C` or a subclass of `C`, and a well-formed type
`C</X/>` is equivalent to `X`. They then treat all class declarations as having implicitly declared
type parameters as `class C</This/> extends B</This/> {...}`. This is very much like the CRTP, but
while `C</This/> <: This`, `C<X>` is not a subtype of `X` as it is with the CRTP. They then assume
that every inexact class type `C` is implicitly `C</~/>` where `~` is a wildcard for some exact type
`#D` that makes `C</#D/>` well form. Also, `C</X/> <: C</~/>` for any `X` in their type system. They
liken this to Java wildcard types and point out that `C</~/>` is conceptually equivalent to an
existential type.

With all that infrastructure set up, *exact type capture* is the introduction of "fresh type
variables to represent the runtime types of variables declared as a class type." This means that
when type checking method calls on a class `C`, they introduce a fresh type variable `X` and replace
`This` in the signature with `C</X/>`. I believe this just explains how to type check method calls
without having to deal directly with the existential type.

Next they introduce *named wildcards*. Named wildcards let you provide a name for the exact type
wildcards to allow the programmer to specify the type relationships between parameters. They give
the example of a compare method declaration `int compare(Ordered</X/> a, Ordered</X/> b)`. Here the
programmer is able to indicate that the two parameters should have the same exact type. Another
example given is `Object</X/> id2(Object</X/> o) { return o; }`. This clarifies the relationship
between the the parameter and return types. It is unclear to me why it is necessary to introduce
this complexity. Couldn't the same effect be achieved by `int compare<T>(T a, T b) where T: Ordered`
and `T id2<T>(T o) { return o; }` respectively? Would that not have the same effect of assigning the
same wildcards to the appropriate types? In Azoth, there would be a difference that these methods
would become generic over struct types, but in Java it seems like these would be identical. Though I
suppose it depends on how you define wildcards to work.

The final thing they introduce to improve parameters is *exact type inference*. This is a system
that infers exact types in a flow sensitive manner. That reduces the need for the developer to use
exact type annotations. Basically, this just means that within a method, local variable exact type
wildcards are inferred. For example, if you declare a variable `BN p1 = x.getPrev();` when the
parameter was `BN</X/> x` then the type of p1 is inferred to be `BN</X/>` (assuming a proper
definition of `getPrev()`). This is only necessary when the method parameters use named wildcards.
(They use JastAdd in their implementation!).

Methods returning `This` have an issue that the only value a method has of type `This` is `this`. To
provide a way around this, they introduce *virtual constructors* that act as a factory method. "An
invocation of a virtual constructor is dynamically dispatched based on the runtime type of the
receiver and generates an object of the same runtime type as the receiver." They allow a class to
declare only one virtual constructor. A subclass may then inherit, override or hide (i.e. C# `new`)
the virtual constructor. If the subclass wishes to change the parameters of the virtual constructor
it must hide the superclass virtual constructor. When a subclass hides the superclass virtual
constructor, it must redefine all methods returning `Self`. Virtual constructors are invoked with
`new This(...)` and declared with `This(...) { /* body */ }`. (I believe there is no syntax for
override vs hide, it is determined by whether the parameters match.)

They give an example where all the above isn't sufficient and so they introduce a *runtime type
match*. This allows the programmer to test whether two objects are of the same exact type at
runtime. The syntax is `classesmatch (x,y) { /* then-block */ ... } else { /* else-block */ ... }`
and is akin to matching on type in C# or Java with `is`. Flow typing is used so that within the then
block of the class match, the two variables are known to have the same exact type and so methods
taking `This` parameters can be called on them. They use this to code an example where a function
counts the number of pairs of equal points in a list of points that may not all be the same exact
type.

They describe in detail how their language features are compiled into plain Java bytecode through a
process they call *inexactization*, which is very similar to type erasure. They also detail the
interaction with other Java features like overload resolution and arrays. There testing showed that
existing Java programs compiled with only minor additional type annotations.

This paper presents lots of interesting ideas and describes a language that truly solves the self
type problem. They describe lots of previous academic work and it seems none of those actually
solved the problem. Instead, there were cases when runtime errors could occur or there were too
sever of limitations. However, introducing to programmers syntax for exact class types (e.g. `#C`)
and exact type parameterization (e.g. `C</X/>`) and requiring them to sometimes use named wildcards
introduces too much complexity. That is made worse by the fact that most Java code does not require
these features. So they become obscure language features that developers are less likely to
understand and remember. It seems to me that depending on the proper behavior of existential types
or wildcard types it should be possible to avoid many of those issues by using more type parameters
on methods. Alternatively, if existential types are directly supported, that might be sufficient.
The example uses of named wildcards might become `int compare(X a, X b) forSome X : Ordered` and `X
id2(X o) forSome X { return o; }`. I think something like that would not have been allowed in Scala
2 because existential types were part of the individual type declarations. Anther place where named
wildcards might be critical is in field types. If you had to fields and wanted them both to be
`C</X/>` how could you do that without introducing an unwanted type parameter to the class?

Their idea of having only a single virtual constructor so that a subclass can hide it and override
all methods returning `Self` is interesting but seems to cause problems. First, imposes the sever
limitation of only allowing one virtual constructor. What if you wanted two virtual constructors
with different parameter types? Second, the requirement to override all `Self` returning methods is
too much. Many `Self` returning methods might be returning `self` yet one is forced to redefine
them. Note this is more onerous than simple overriding since one is not allowed to call the base
implementation in case it actually returns a base type. It would also interact poorly with `final`
methods (i.e. `sealed` in C#). Can one not hide a virtual constructor if the superclass has a
`final` method returning `This`?

## Framing

At the risk of prematurely selecting an approach, I think it is worth laying out some ideas about
how to think about the `Self` type and the range of designs that are possible.

### Bounded Associated Type

**NOTE:** This section is incorrect, but is being left in for reference.

First, it seems intuitively correct that the `Self` type is essentially an associated type generated
by the compiler that is automatically constrained by the compiler. Consistent with the are:

* The proposed design for Scala 3 `This` type
* Swift existential types for protocols using `Self`
* It is definitely seems to act like a type variable
* It shouldn't be passed like it is a type parameter (though of course, associated types can be
  transformed to type parameters).

The following rules seem to apply to the `Self` type:

* In a `struct` or `sealed` class `C`, `Self` = `C`
* In any class or trait `T`, the constraint `Self <: T` applies
* As in Scala 3 self types, it probably makes sense to allow an additional bound to be placed on the
  `Self` type. So the users statement `associated type Self <: B` for some type `B` is equivalent to
  `Self <: T & B`.

### Implicit Parameter

If a self type were supported in C#, it should be the case within a type that `typeof(This) ==
this.GetType()`. Consider a concrete class `C` that is also the base class of several other classes
`S1`... `S`*N*. For an instance of `C` constructed with `new C(...)` the `Self` type is `C`. But for
an instance of `S`*N* constructed by `new S`*N*`(...)` its `Self` type is `S`*N*. If `Self` were an
associated type then it would need to be constrained in `C` to `Self <: C`. But in order to be able
to instantiate it, it would need to be declared `Self = C` in `C`. However, that would prevent
subclasses from changing it. Thus, it cannot be an associated type. It must be some sort of implicit
type parameter. (For example, Scala only allows [abstract type
members](https://docs.scala-lang.org/tour/abstract-type-members.html) in abstract classes and
traits. Swift only allows [associated
types](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/#Associated-Types)
in protocols.)

Thus, self types must be a constrained implicit type parameter that is passed the type itself
whenever constructing them. Thus a constructor of type `T` always returns `T[Self = T]` (assuming
the implicit parameter is made explicit as the type parameter named `Self`). This still leaves a
need for existential types `T[Self = *]` and some sort of types match construct to enable calling
binary methods. It also begs the question what the `Self` type of the inner `T` is in the type
`T[Self = T]`.

### Flavors

In the review of the prior art, it is clear that there are actually multiple flavors of self type
and it isn't immediately obvious which one will be both implementable with Azoth and powerful enough
to enable the desired use cases. The solution may not map cleanly to one of these flavors. It may be
a mix of two of them. But the flavors will help clarify the problem. Let's explore and name each of
the flavors so we can refer to them when looking at the use cases.

#### `Class` Type

Swift is the clearest example of this, though Rust is sort of an example too. In Swift protocols the
`Self` type is mostly a type variable referring to the class or struct that conforms to the
protocol. Due to lack of a good term for this, I am shorting it and calling this the `Class` type.
Note that "concrete" would not be correct because if an abstract class implements the trait, then it
would still be the `Class` type the trait refers to.

A `Class` type does not then carry down to subclasses. It is only really meaningful within the trait
and the implementing class can just use its own class name when overriding members of the trait that
used `Class`. If `Class` is allowed to be used within a class or struct declaration, then it is a
simple synonym for the name of the current class. Of course, it could be given a different meaning
as how `Self` works in Swift class method return types, but for the sake of clarity, I will try to
separate those concepts. That is, `Class` may be treated as a simple synonym for the containing
class of a declaration with no special effect on subclasses. This is seen in Swift for parameters of
method that conform to a protocol taking a parameter of type `Self`.

Within a trait, `self <: Class`.

#### `Derived` Type

One major issue with the `Class` type is that it makes a clear distinction between traits and all
other types. It treats traits as second-class citizens. In Azoth, all classes implicitly create a
trait and there should be far less of a distinction between them. For example, changing an abstract
class with no state into a trait should not cause a change in the behavior of the system. But if
that abstract class implemented a trait that used the `Class` type, then it could. The `Class` type
would not refer to a different class. The subclass of the removed abstract class. That is a real
issue for intended design of Azoth.

Trying to formulate a type like `Class` that is agnostic to the distinction between classes and
traits, one might imagine a `Derived` type. This is a type variable that refers to the immediate
subtype of this trait or class. There are several strange things about this. One is that at each
level of the type hierarchy the `Derived` type variable refers to a different type. In this way, it
would have to be more like an implicit type parameter that is implicitly passed in the declaration
of the subtype. If so, it might be desirable that there was a way to "forward" on the `Derived` type
between levels. That is, to indicate that a particular type should pass its own `Derived` type
variable on as the implicitly passed type. Any class directly instantiated would have its `Derived`
type variable set to itself.

The impact on subtyping on having members taking parameters of type `Derived` or returning `Derived`
is very confusing. At each level the type parameter is independent. Yet, there is will be subtype
relationship where it will go strictly down the hierarchy. This would be fine for return types but
for parameters types would require some way for the compiler to determine that the types were
compatible.

It is always the case that `self <: Derived`.

#### `Root` Type

For completeness, one can imagine another variation where the type variable refers to the root class
or struct in the type hierarchy. This has the same problem as the `Class` type of treating trait
types as second-class. It does however have the benefit of giving a consistent meaning to the `Root`
type variable throughout the hierarchy so that there is no issue with subtyping. In languages like
C# or Java, this would make `Root` a synonym for `Object` but in Azoth, `Any` is a trait and there
is not root class.

It is always the case that `self <: Root`.

#### `Instance` Type

The instance type is a type variable that will always refer to the true type of the current
instance. This is what I think of as a true `Self` type. This is also the definition being used by
the proposed Scala 3 `This` type and the paper on `ThisType`.

Using `Instance` as a return type is relatively straightforward as shown in Swift classes. Using it
as a parameter type poses many problems as described in the `ThisType` paper. In particular, it
effectively makes every bare type that internally uses `Instance` into an existential type.

It is most correct to say that `self: Instance`. That is that the `Instance` type is the true type
of `self` not just a supertype.

While the type value of the `Instance` type parameter is not technically an exact type, the fact
that it is almost always an unknown type means that it is very difficult to construct a scenario
where it doesn't act like an exact type. Types do need to be invariant over their `Instance` type.
Otherwise, incorrect binary method calls could be allowed. Furthermore, the `Instance` type needs to
be treated has having the `Instance` associated type equal to itself. For example, consider a method
that checks `x is Instance xInstance`. If this evaluated to true even though the true type of `x`
was a subtype of `Instance`, it would allow binary methods to be called when they shouldn't be.
However, since if the true type of an instance is `T` then `Instance` is effectively `T<Instance =
T>` only other instances with the same exact type will match the type.

Actually maybe it is actually an exact type. If type parameters are allowed to be fully self
referential, then for some class `C` let `T = C[Instance = T]`. Then `T` really is an exact type.

The above is incorrect, types must be covariant on the self type. This is necessary to use some type
`T[Instance = *]` where `*` indicates an existential type. If the true type is `C` then it must be
the case that `C[Instance = C] <: T[Instance = C]`. Otherwise the only type that would match
`T[Instance = *]` would be `T[Instance = T]`.

It is interesting to note that `T[Instance = C] <: C`. In the `ThisType` paper, they use equality
instead because they make `Instance` a parameter over exact types.

#### `Same` Type

Another possible flavor of a self type is something that is not a true type variable. Instead the
`Same` type is a synonym for the declaring type that requires that the `Same` type also be used when
overriding. As such it violates subtyping to use it in input position. When used in return position,
it does not force the method to be overridden in subtypes. Since it cannot be used for parameters it
is very similar to just using covariant return types to narrow the return type of the method, just
with an added requirement that it *must* be narrowed. That may make it not worth the complexity of
adding.

## Ideas

Here I will lay out various ideas that exist in the toolkit for tackling the challenges of self types.

### `required` Methods

A method marked `required` would have an implementation but be required to be overridden in a
subtype. The overriding method would not have to be `required`. This could help with methods
returning self types. The exact rules would be:

* A `required` method cannot be `abstract`
* In a subclass a `required` method must be overridden. The override does not itself need to be
  `required`.
* A `sealed` class cannot have `required` methods.
* A `required` method implementation returning a self type, treats the return type as a synonym for
  the containing type name. Note though that for the `Instance` type it makes the associated type
  also equal to the containing type rather than an existential type. Thus it effectively restricts
  the return type to be exactly the containing type.
* When calling a base required method from the overriding method, the return type of the base method
  is treated as the base type.

The final two rules are what makes them useful as factory methods for self types. Those rules are
safe because the required method will return a value that is of the type of the class declaring the
required method. That will satisfy any supertype methods being overridden. Any subtype will be
forced to override the method and in so doing forced to return a value compatible with the self type
return.

This is not a great name for this. However, it is unclear what would be a better one. The `required`
keyword is taken from the keyword used on Swift constructors which is similar but not quite the same
situation. Saito and Garashi in *Matching `ThisType` to Subtyping* 2009 use `nonheritable`. Note
that `sealed` with `Self` return type does not work because it could be sealed and returning `self`
so it can't be taken as requiring the method be overridden.

### Virtual Constructors

The `ThisType` paper and others have proposed virtual constructors and Swift has them as well. As
noted before, the variation of virtual constructors proposed in the `ThisType` paper has issues. So
here we propose a variation equivalent to Swift's approach.

A constructor is marked virtual with the `required` attribute. This indicates that each subtype is
required to have a constructor with matching parameters. The rules for virtual constructors are:

* Virtual constructors return either the self type in question or the declaring type whichever is
  more specific. (We are here leaving open any of the flavors discussed above.)
* The `required` keyword allows for the fact that the constructor body actually "returns" the
  declaring type. (See meaning of `required` on methods returning self.)
* Any subtype must declare a compatible constructor with the `override` keyword to override the
  required constructors. (Optionally, constructors could be "inherited" in some situations as done
  in Swift. I think implicitly implemented would be the more correct term.)
* Overriding constructors will have to be `required` themselves unless the type is `sealed`. This is
  necessary since it will have a self return type.
* Required constructors can then be called from within the type using `new Self(...)` and a virtual
  call will be made. The return value will have the self type.

An alternative design could have the `required` attribute be necessary only at the levels where `new
Self(...)` is called. Though a required constructor would still have to be overridden in every
subtype. Thereby making it clear what is really required and so if a supertype removed a required
constructor it would be clear that the subtype constructors are no longer required.

### Same Type Test

This introduces a pattern match like syntax that allows one to type test that two values have the
same self type. In the `ThisType` paper they use a flow sensitive type control flow construct.
However, Azoth only uses flow sensitive typing for reference capabilities. When doing type checking,
it requires pattern matching to assign a new name. Following that approach the straw man syntax
`#(x, y) is let (x1, y2) and Self == Self` allows one to create new variable that the type system
knows have the same exact type. (This really needs a better syntax.)

### Exact Types

I am not including exact types, exact type parameters etc. in the list of ideas. I already believe
these are too complex to add to the surface syntax of Azoth. As such they are not worth exploring.
I do want to point out though that if both supertype and subtype constraints are available then
something like exact types can be simulated. Naming a type parameter or existential type variable
`T` can be made to have an exact type by bounding it both above and below. If `C` is a class type
the `T where T <: C and C <: T` makes `T` exactly `C`.

It should be noted though that exact and inexact types may still be necessary for type checking with
self types and reasoning about them.

## Unorganized Thoughts

**NOTE:** at about this point it became clear that I was still confused about certain things. So I
began to just make notes and have thoughts to grope toward a solution. This section records that.

***

`Self` is a type parameters where in any type `T`, `Self <: T`. To override a method, that uses
`Self`, one must also use `Self` because that is what you know to be of type `Self`. **`self:
Self`**. Required methods and virtual constructors let on instantiate instances of type `Self`.

***

The challenge is to invoke a binary method. The same instance check creates a type parameter `I <:
T` where `x: I` and `y: I` but that doesn't seem sufficient because two normal instances that where
both `I` would not be safe to call a binary method on. This is where you need existential types and
type parameters.

***

`Self` is not an associated type because of how it works in a concrete base class. Generally, an
associated type is either abstract or once set cannot be changed in subtypes. Note how Scala
associated types are only allowed in traits and abstract classes. However, the `Self` type will have
the class type in constructed instances of a concrete base type at the same time that it will have a
different value in subclasses.

***

Three kinds of types `T[*]` (existential on `Self`), `T[*X]` (named existential on `Self`),
`T[*Self]` (need a better notation) aka `#T` aka `T[$]` (exact type where `Self` is recursively
defined i.e. `T = T.Self`).

***

The same type test operation creates a type variable `X[$] <: C` for the constraint `C`. And both
variables have that type, so you can call binary methods.

***

If `T` derives from `B` then `T[*] <: B[*]` but it is not the case that `T[$] <: B[$]` (sometimes
that will hold, but it cannot be generally assumed or taken as true). So it is actually invariant in
the type parameter, but the existentials allow subtyping. The recursive formula for self types
amount to exact types but are more complex because they would allow types like `T[C[D[*]]]`.

***

What is instead of exact types you really treat it as an invarient type parameter that you can pass
by name? `int Compare<T>(Ordered<Self=T> a, Ordered<Self=T> b)`. That is essentially what you have
to write in .NET now. `int Compare(Ordered<*T> a, Ordered<*T> b)`. Maybe it is an optional first
type parameter? Must be explicit when using as method parameter? `Ordered<Self=*>` or
`Ordered<Self>`.

***

Once can use `Self` in return position all the time.

The `Self` parameter is an implicit first parameter. Prefix a parameter with `:` to pass the self
parameter. `int Compare<T>(T a, T b) where T: Comparable<:T>`.

But, `where T: Comparable` already implies `T: Comparable<:T>` right?

For `Comparable`, you want it to act a little more like an associated type `trait Comparable<in T =
Self>`? The `in` here implies that it isn't an invariant `Self` parameter.

***

```azoth
trait Comparable
{
    associated type T (default Self);
}
```

***

There are two valid models of associated types.

1. Associated type that is covariant.

    Thus can be overridden in subtypes. Associated types are invariant by default. So once assigned
    in a type, they cannot be reassigned in subtypes. However, they can be marked `in` or `out` and
    that allows them to be reassigned in a subtype in a way that was compatible with the variance.
    Thus `Self` is an implicit `out` associated type that gets implicitly reassigned in each subtype
    to the current type.

2. Invariant Type Parameter

    All uses of types are implicitly existential types where the implicit `Self` type parameter is
    existential. Note, this is not an associated type because since it is invariant, one wouldn't be
    allowed to reassign it in a subclass.

***

I'm not convinced that the limitation that you can only implement a trait with an associated type
once makes sense. It might be a limitation of treating traits as second class citizens. For example,
imagine a class implements multiple traits representing that you exist in multiple categorization
schemes. Each of those categorization schemes may have a separate implementation of comparable. So
your class would be comparable in multiple ways. But that doesn't mean that it doesn't make sense
for `Comparable` to use an associated type.

***

The associated type for `Equatable` or `Comparable` ought to sort of default to the type of the
current class. However, it could be set at any level in the hierarchy as long as it is being set to
the current type. It can be bounded by self `type T where Self <: T where T <: Equatable { T = T }`
(Where `T` means two different things in the two uses inside of curly braces). But that still isn't
sufficient because you shouldn't be allowed to set it as being equatable to a subtype or supertype
that is still in the inheritance chain. For example, if a class is halfway down the chain, it should
not be allowed to declare that it is equatable to the type above it in the chain. That doesn't make
any sense. Likewise, equatable to the type below it in the chain doesn't make sense. So there must
be some additional constraint that requires that it be set to the current type when it is being set.

***

I've been thinking that the `Self` type maybe shouldn't include the reference capability. Likewise,
I've thought it shouldn't be included when dealing with the type parameter of equatable and
comparable. Now that I am thinking that equatable and comparable should use associated types, maybe
the answer is that associated types don't have reference capabilities but type parameters do. That
actually makes sense because when you declare a type outside of a class, you are not saying it has a
specific capability. So when you put the type declaration inside the class and it becomes virtual,
you still aren't talking about a type with a reference capability. You are talking about a bare
type. This really seems like it might be the right thing.

***

Maybe the self referential type constraint (i.e. `type T where Self <: T where T <: Equatable { T = T }`) does work for the associated domain type of
`Equatable` and `Comparable`. `Comparable` has an associated type `Domain` where the `Domain`
implements `Comparable` and `Domain.Domain = Domain`. That means that whatever type ever gets
assigned to `Domain` has to be `Self` I think?

***

The constraint on the `Domain` of `Equatable` described above is not sufficient because it would let
you name a different type that already implemented `Equatable` on itself, but that constraint plus
the supertype of the self type might be sufficient. (I am no longer sure what I meant by that.)

***

If you can implement a trait multiple times then it isn't sufficient, because you could name a
supertype that implements the trait with a different associated type.

***

If you can implement a trait multiple times there is a case where those constraints would not be
sufficient. Imagine `C <: B <: A`. Both `B` and `C` implement `Comparable`. `C` does so properly.
`B` tries to set the associated domain of comparison as `C`. I think that satisfies both the
constraints. No, no, it isn't going to satisfy the self type constraint. So maybe there is no
problem because if `B` implements it correctly and `C` names `B` as the domain, then it has just
restated a true fact that is already there. The two implementations of the trait are identical.

## Reformulation of Self Types

Having thought through what makes sense for self types more, I think it is clear there are only a
few possibilities and those are what we should evaluate with the use cases. These possibilities are
composed of some core functionality along with advanced functionality that may not be necessary.

### `Self` Basics

The `Self` type is an implicit type variable. It is always constrained by the declared type it is
used in. When used inside of a type declaration `A`, `Self <: A`. The `Self` type is always the type
of the current instance. It acts like an implicitly declared and set covariant associated type. That
is, each type declaration sets the `Self` type to itself, but subtypes set it to themselves. This is
possible because it is covariant and can be assigned new values as long as those values are more
specific.

The `self` instance variable is of the `Self` type (i.e. `self: Self`).

In `struct`s and `sealed` classes, the `Self` type is known to be equal to the declaring type.

### Output `Self`

Since `Self` is covariant it can be used safely in output position and for local variables. Within a
method, the only instance known to be of the `Self` type is `self`. In order to be able to return
any other instances, from a method returning `Self` that is not in a `struct` or `sealed` class,
required methods or required virtual constructors or both are needed.

#### Required Methods

A `required` method is one that has an implementation, but must be overridden in any subtype. Within
a required method returning `Self`, the return type is instead treated as being equal to the
declaring type. This allows the method to return new instances of the declaring type. This is safe
because any subtype will be forced to override the method and return an instance of type `Self`.
Thus, the method implementation will be called only when the instance is of the declaring type or a
call to a base method is being made. In the case that a base call is made to a required method, the
return type is again treated as being the declaring type of `base`. Also, due to this, `required`
methods must inherently be `open` in Azoth. It is undecided whether that means the `open` keyword is
required or whether it is implied as it is for `abstract`. Since `required` could be applied to
methods that do not return `Self`, perhaps `open` should be required only when a `required` method
returns `Self`.

Required methods are thus able to act as factory methods for instances of `Self`.

#### Required Virtual Constructors

Another option is to have `required` virtual constructors. A `required` virtual constructor must be
overridden in subtypes and each overriding constructor must also be marked `required`. These
constructors can then be invoked with `new Self(...)` and have a return type of `Self`. In this way,
they can also act as factories for values of type `Self`. The Swift language already has virtual
constructors with designated and convenience constructors. That is a lot of infrastructure and
complexity. If it is decided that they are necessary to Azoth, then `required` constructors are an
obvious extension. However, if not, then it probably makes more sense to stick with only required
methods.

### Input `Self`

Because `Self` is covariant, it can't be used in input position in methods. This can be an issue for
binary methods. That is methods that one would want to take a parameter of `Self` in addition to the
`self` parameter that is already of type `Self`. If binary methods need to be supported then there
must be a way to first allow subtyping even though having `Self` in input position violates
subtyping and also, to establish the proper type relationships to allow binary methods to be called.

To enable this, we first must introduce the concept of exact types and existential types. An exact
type for the type `T` is written `#T` and is the type of all instances of type `T`. However, it does
not include any subtypes. So if `S <: T` it is not the case that `S <: #T` or `#S <: #T`. However,
it is always the case that `#T <: T`. (Note: `#T` is not valid Azoth syntax. It is merely notation
for discussing exact types.)

Now let us introduce existential types. For each type, there is an associated `Self` type that is
implicit. By default it is implicitly given an existential type. That is "there exists" some type
that satisfies it. This is like Java wildcard types. We allow the implicit `Self` type parameter to
be listed first among the parameter list but distinguish it by prefixing with a `:`. This when using
some type `T`, what we want to do is instead use `T[:*]` where `*` is the wildcard type. Note that
if `S` was previously a subtype of `T`, then `S[:*] <: T[:*]`. One important point is that the
existential wildcards must range over *exact* types only. (Note: At his point, we are not proposing
that `T[:*]` is valid Azoth syntax. It is merely notation for discussing exact types.)

Now, with those two concepts in place, we change the meaning of using some type name `Foo`. If `Foo`
is a struct or sealed class than `Foo` is refers to the exact type `#Foo`. Otherwise, `Foo` refers
to the existential type `Foo[:*]`.

#### Named Wildcards

It may be necessary to relate existential types to each other. To do that, one could introduce a
syntax of naming existential type parameters. The syntax would be something like `fn example(a:
Foo[:*T], b: Foo[:*T])`. This declares a function `example` taking two parameters implementing the
`Foo` trait that are known to have the same `Self` type. This would then allow binary methods to be
called on them.

#### `Self` Match

This is a new pattern match or control flow structure that conditions on two variables being of a
given type *and also* having the same `Self` type. This way we can take to variables that are not
known to have the same `Self` type and recover a case where they are known to have the same `Self`
type and binary methods can be called on them.

I am not sure what a good syntax for this would be. One could treat it like pattern matching on a
tuple. Options for that include:

* `#(a, b) is let x, y and Self == Self`
* `#(a, b) is let x: T[:*X], y: T[:*X]`

Alternatively, one could have a special match construct like `a, b are_same x, y : T`.

Or one could imagine an actual control structure that induces flow typing. `self_same a, b { ... }`.

### Associated Type Bounded by `Self`

There is a case not covered by the above. This case is an associated type bounded by `Self`. It
seems there might be a common pattern of an associated type `T` in trait `Trait` with a bound like
`where Self <: T where T <: Trait and T.T == T`. Or in an alternative notations `where Self <: T
where T <: Trait { type T = T}` or `where Self <: T where T <: Trait[.T = T]`. This constraint is
tight enough to almost ensure that the associated type can only be populated with the declaring type
in whatever type sets it. However, it is complex and there is one case it doesn't quite prevent. So
perhaps it would make sense to create a special syntax for it. One idea is `where T <:
Trait[:Self]`. But it is unclear whether that is correct since it would have a specific meaning. It
is also dependent on the syntax introduced for named wildcards. If named wildcards are needed and
aren't added, this syntax would be confusing. It is quite difficult to come up with a good syntax.
Perhaps `self type T`?

## Use Cases

This problem is very complex and has few obvious guiding principles. As such, a careful review of
actual use cases will be a better guide to the correct design than an abstract understanding the a
"correct" design.

### Clone

Consider the problem of a trait and method that produces a clone of an object. The return value is
expected to be of the same instance type as the instance being cloned. The cloneable interface has
been criticized in both C# and Java but we will use it here as a stand-in for a more specific clone
or copy trait that does not have the same issues. In C# the `ICloneable` interface is used along
with the `object.MemberwiseClone()` method. While in Java there is the `Cloneable` interface and the
`object.clone()` method which performs a memberwise clone but throws an exception if the current
class doesn't implement `Cloneable`. Both return `object` to the great frustration of everyone who
tries to use them.

What are the requirements for the cloneable use case? Each subclass should override the clone method
so that it returns a new instance that has the same type as the current instance. Some edge cases
are interesting. If you call `clone()` on a variable of type `Cloneable` you expect to receive a
`Cloneable`. As noted in the sections about proxies, a proxy class may want to return a value that
is not of the proxy type. For example, a proxy for an object across the network may wish to return a
local clone that is not proxied. One can theorize other cases where one might want to return an
unwrapped or simplified instance that is not of the same type. Consider the declaration below.

```azoth
public trait Cloneable
{
    public fn clone(self) -> Self;
}
```

To implement this trait, one probably needs either `required` on the `clone()` method or needs to
call a required virtual constructor. Presuming that some copy constructors exist it seems most
straightforward to use `required` on the `clone()` method. Otherwise, one would be stuck trying to
create a `required` constructor that can somehow properly clone.

However, this version of `Cloneable` seems to be stricter than what some versions of clone want. A
proxy could not return a non-proxy. The unwrapping or simplifying case would likely also be blocked.
To loosen the requirements, we could switch to an associated type bounded by `Self`.

```azoth
public trait Cloneable
{
    public out type Clone
        where Self <: Clone
        where Clone <: Cloneable[.Clone = Clone]

    public fn clone(self) -> Same;
}
```

This trait allows each subtype to further refine the `Clone` result type. Implementing it is
straightforward, but it is also possible to implement it incorrectly by not overriding it when one
should. The covariant `out` does seem to work correctly with the advanced type constraint.

Features Needed:

Implementing `Cloneable` properly requires either:

* Output `Self`
* `Self` Match

Or

* Associated Type Bounded by `Self`

### Copy Constructors

In order to support copyable structs, special copy constructors may be needed. A copy constructor is
essentially a required constructor. If a copy constructor is declared, then each subtype must
provide its own copy constructor. The difference is that they are virtually dispatched on the
parameter type. It does however seem that copy constructors may introduce more complexity than just
have a special `copy` method and possibly a `Copyable` trait. In that case, `Self` would be used as
an output type and required methods would be needed.

Features Needed:

* *Maybe* Required Constructors
* *Maybe* Output `Self`
* *Maybe* Required Methods

### Equatable

There are at least two approaches to supporting `Equatable`.

The advanced approach is built off the insight that while it is slightly odd, it is reasonable to
compare any two equatables for equality. Thus the most basic implementation might be:

```azoth
public trait Equatable
{
    public fn Equals(self, other: Equatable) -> bool;
}
```

But that requires type checks or something like the Scala `canEqual` method to properly implement
equality. If one expects that for two things to be equal, they must always always be the same type,
then one could extend the `Equatable` type to enforce that and simplify writing the equality check
as:

```azoth
public trait Equatable
{
    protected fn equals(self, other: Self) -> bool;

    public fn equals(self, other: Equatable) -> bool
    {
        if(#(self, other) is let (x, y) and Self == Self)
            return x.equals(y); // Calls other overload

        return false;
    }
}
```

However, another approach is to think that `Equatable` establishes a domain of equality. Within that
domain, everything must be equatable to everything else. Types may not have to be identical for two
values to be considered equal. To support this, the associated type bounded by `Self` pattern makes
more sense.

```azoth
public trait Equatable
{
    public type Domain
        where Self <: Domain
        where Domain <: Equatable[.Domain = Domain]

    public fn equals(self, other: Domain) -> bool;
}
```

Note that here `Domain` is invariant because even though it is used in input position, it doesn't
really make sense to expand the domain that can be checked for equality.

We'll see in the next section that the latter approach is more consistent with comparison.

Features Needed:

Implementing `Equatable` properly requires either:

* Output `Self`
* `required` methods

Or

* Associated Type Bounded by `Self`

### Comparable

While it is somewhat reasonable to imagine checking any two `Equatable`s for equality, it is not the
case that any two `Comparable`s can be compared. Instead, `Comparable` must establish a domain
within which items can be compared to each other. I am surprised that the Scala `Ordered` trait
doesn't talk about using `canEqual` or something similar to check whether to values can be compared.
The only reasonable way to implement `Comparable` is with a bounded associated type.

```azoth
public trait Comparable <: Equatable
{
    // The type is already declared, but we want to further constraint it
    public override type Domain
        where Domain <: Comparable[.Domain = Domain]

    public fn compare_to(self, other: Domain) -> Ordering;
}
```

Features Needed:

* Associated Type Bounded by `Self`

### Abstract Domains with Multiple Implementations?

**TODO:** This example is given in the ThisType paper, but I don't fully understand it yet.

### Fluent API

In a fluent API, one might wish to make sure that specific steps returned `self`. This is a simple
use of the `Self` return type.

Features Needed:

* Output `Self`

### Proxies

As partially already covered in other use cases, a proxy may wish to behave like another type. This
is an argument against true binary methods. A `Self` bounded associated type allows the proxy to
keep the associated type as a supertype and thereby unproxy or compare to an unproxied instance.

Features Needed:

* Associated Type Bounded by `Self`

### Doubly Linked List

Some sources give the example of a doubly linked list implementation that inherits from a singly
linked implementation. Personally, I think that is strange and probably a bad idea. But it does
potentially allow single linked list traversals to operate over the doubly linked list.

```azoth
public class Linked_Node[E]
{
    public protected var elem: E;
    public protected var next: Self;

    public fn insert(self, node: Self)
    {
        Self tmp = self.next;
        self.next = node;
        node.next = tmp;
    }

    public fn insert_element(self, e: E)
    {
        Self newNode = self.make_node();
        newNode.elem = e;
        this.insert(newNode);
    }

    public required fn makeNode() -> Self
        => new Linked_Node[E]();
}

public class Doubly_Linked_Node[E]: Linked_Node[E]
{
    public var previous: Self;

    public fn insert(self, node: Self)
    {
        super.insert(node);
        node.previous = this;
        node.next.previous = node;
    }
    public required fn makeNode() -> Self
        => new Doubly_Linked_Node[E]();
}
```

They then give the code below as an example of client code that could cause problems. However, the
other paper explains how exact type capture can allow the compiler to properly type such code.

```azoth
let head: LinkedNode[Integer] = ...;
head.next.insert(head); // possibly ill-typed
```

### Sorting Ordered List

Consider a function that wants to sort a list of items that implement `Comparable`. For that, we
need not only that the list element type is `Comparable`, but that it is within the comparable
domain. Or put another way, the `Domain` must not be abstract. I am not sure the best way to handle
that. It may be the case that the Domain not being abstract is already a requirement. Just like you
can't use a type as a type parameter when it has abstract static members.

```azoth
public fn sort[T](list: List[T])
    where T <: Comparable, T.Domain
{

}
```

## Conclusion

There is no proven need for input `Self` and it introduces a lot of complexity. It seems best to
leave it out for now. However, it seems like output `Self` and associated types bounded by `Self`
are both important to have. To support that, it seems required methods should also be supported.
Required constructors should be supported only if virtual constructors are added for other reasons.
