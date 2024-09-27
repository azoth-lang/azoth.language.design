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

#### `Same` Type

Another possible flavor of a self type is something that is not a true type variable. Instead the
`Same` type is a synonym for the declaring type that requires that the `Same` type also be used when
overriding. As such it violates subtyping to use it in input position.

## Ideas

Here I will lay out various ideas that exist in the toolkit for tackling the challenges of self types.

"To address this problem, Bruce and Foster [2004] proposed factory method patterns and Saito and
Igarashi [2009] proposed nonheritable methods"

### `required` Methods

A method marked `required` would have an implementation but be required to be overridden in a
subtype. The overriding method would not have to be `required`. This could help with methods
returning self types. The exact rules would be:

* A `required` method cannot be `abstract`
* In a subclass a `required` method must be overridden. The override does not itself need to be
  `required`.
* A `sealed` class cannot have `required` methods.
* A `required` method implementation returning a self type, treats the return type as a simple
  synonym for the containing type name.
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
these are too complex to added to the surface syntax of Azoth. As such they are not worth exploring.
I do want to point out though that if both supertype and subtype constraints are available then
something like exact types can be simulated. Naming a type parameter or existential type variable
`T` can be made to have an exact type by bounding it both above and below. If `C` is a class type
the `T where T <: C and C <: T` makes `T` exactly `C`.

It should be noted though that exact and inexact types may still be necessary for type checking with
self types and reasoning about them.

## Use Cases

This problem is very complex and has few obvious guiding principles. As such, a careful review of
actual use cases will be a better guide to the correct design than an abstract understanding the a
"correct" design.

### Clone

Consider the problem of a trait and method that produces a clone of an object. The return value is
expected to be of the same instance as the instance being cloned. The cloneable interface has been
criticized in both C# and Java but we will use it here as a stand-in for a more specific clone or
copy trait that does not have the same issues. In C# the `ICloneable` interface is used along with
the `object.MemberwiseClone()` method. While in Java there is the `Cloneable` interface and the
`object.clone()` method which performs a memberwise clone but throws an exception of the current
class doesn't implement `Cloneable`. Both return `object` to the great frustration of everyone who
tries to use them.

What are the requirements for the cloneable use case? Each subclass should override the clone method
so that it returns a new instance that has the same type as the current instance. Some edge cases
are interesting. If you call `clone()` on a variable of type `Cloneable` you expect to receive a
`Cloneable`. However, as noted in the sections about proxies, a proxy class may not want to return a
value that is not of the proxy type. For example, a proxy for an object across the network may wish
to return a local clone. One can theorize other cases where one might want to return an unwrapped or
simplified instance that is not of the same type. Consider the declaration below

```azoth
public trait Cloneable
{
    public fn clone() -> Instance;
}
```

This seems to be stricter than what many versions of clone fundamentally demand. Even if some
special exemption were made for proxy types, the unwrapping or simplifying case would likely be
blocked. It seems what we want is that for all `T <: Cloneable`, `T.clone() -> T` where `T` is a
standard inexact type. This would then be the `Same` type. Though others would argue that is too
loose.

```azoth
public trait Cloneable
{
    public fn clone() -> Same;
}
```

Review of flavors

* `Class`: Incorrect, does not vary with subclasses.
* `Derived`: Incorrect, even if it requires the use of `Derived` in subtypes, if over constrains the return value.
* `Root`: Incorrect, does not vary with subclasses.
* `Instance`: Possible, it is too constricting in some cases but matches many other cases definitions.
* `Same`:

Does clone require exact type?

### Copy Constructors



### Equatable

### Comparable

Shouldn't be allowed to compare any comparables

### Abstract Domains with Multiple Implementations?

### Fluent API



### Proxies

Clone may unproxy.


### Doubly Linked List

A doubly linked list that inherits from a singly linked list since a doubly linked one can be traversed like a doubly linked one (Example from *On Binary Methods*)

Does `set_next()` need an exact type for its parameter? Some papers think so. It does because `LinkedList` is a supertype of `BinaryLinkedList`
