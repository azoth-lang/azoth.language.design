# Classes and Structs

## Value Types

The `struct` keyword is used to define value types. Originally, Azoth was not going to have value
types, but only reference types declared with `class`. This ran into several issues.

First, it was unclear how to interop types with C without complicated marshalling since all types
could have a vtable and might be subject to slicing of subclasses. There were also issues about
subtyping since any type could be subclassed. How would one handle a subclass of `int` being passed
to an external function? Still it would have been possible to apply the `external` keyword to class
declarations and restrict external classes to have no base class and no subclasses.

Second, there were problems around combining move and copy semantics into reference types. For
example, consider a reference to a `var` containing a class with move semantics. What if one were to
assign a different instance into it that was actually a subclass. Since they would be different
sizes, that could never be implemented unless types with move semantics were still stored as
references to the heap. Of course, the compiler could probably optimize that away in many cases, but
it seemed foolish to rely on that so much.

## New Operator

The syntax for constructing values evolved over the history of the language design. Originally, a
`new` operator was required for constructing both classes and structs. Then it evolved to only be
required for classes. Now, there is no `new` operator for either structs or classes.

### `new` Everywhere

Any value with mutable state should be created with the `new` operator. Rust doesn't have a new
operator. However, it was decided that for Azoth the explicitness of having a new operator for
allocating memory was good. It then might seem to make sense to not use new for structs. Using new
for structs in C# has always felt a little weird. However, the optimizer of Azoth is expected to
frequently convert reference types into stack allocations so they might end up equivalent. Also, it
makes structs consistent. And it would seem weird to declare struct constructors using new but then
call them without new. One can't just have static functions build structs because of safety around
definite assignment.

### `new` for Classes

The weirdness of using `new` on structs continued to grate on me. With the introduction of
initializers declared with `init` it became possible to let `struct`s have initializers and classes
have constructors. This seemed to make sense since from the callers perspective, initializing a
struct was equivalent to calling a static function that returned the struct. However, to allow
structs to masquerade as classes, constructors were still allowed on structs. For example, a
`String` or `Array[T]` might technically be a struct, but creating one allocated memory on the heap.
They were effectively thin wrappers around a reference so it seemed to make sense that they ought to
be constructed with `new`.

### No `new`

It seemed for a long time that using `new` for classes only would be the final design. However,
during the design of `closed` types for enums and discriminated unions, the `object` keyword was
introduced for other uses and this opened up the possibility of dropping `new` since it would no
longer be needed for anonymous classes/objects. Also, there was concern about the language
complexity that was being introduced with `closed`, `object`, `value`, and `case` (and other designs
were even more complex). Removing `new` became a way to reduce complexity in the language in other
places.

Pros of `new`:

* Clearly marks heap allocation
* Provides obvious syntax for handling allocation failure (i.e. `new? Foo()` would would return
  `none` if there wasn't enough memory)

Cons of `new`:

* Heap allocation can happen inside of any function, especially factory functions. So marking heap
  allocation was a bit of a false label. It made an unnecessary distinction. For example, it didn't
  mean that a struct initializer didn't do any heap allocation.
* Complexity of having both constructors and initializers
* Complexity of supporting constructors on structs so they can act as reference wrappers

Weighing the pros and cons, it was decided to remove `new` from the language and use initializers
for both classes and structs instead. Both Swift and Kotlin have paved the way here and it seems to
have worked well. This simplifies the language. The handling of allocation failure was more
important in the predecessor to Azoth, called Adamant, because it was a lower level language.
Adamant didn't have a garbage collector so the developer had much more control over memory
consumption. Supporting that in Azoth doesn't seem important given that it has a garbage collector.
No other mainstream high-level garbage-collected language has something like that. Furthermore, if
desired, a syntax for it could be added. For example perhaps `Foo?()` would be a workable syntax.

## All Methods Must Have a Unique Implementation

Given a method `M` in interface `A` and interfaces `B` and `C` that implement `A` and override `M`,
a class `D` that implements `B` and `C` must override `M` to provide a unique implementation of `M`
to be called. This applies even if `B` and `C` rename `M`. At first glance this seems a strange
restriction to place. However, since Azoth is trying to not lock in an implementation, this
restriction ensures certain implementations are possible, and can be safely removed in the future if
desired. Specifically, it enables a vtable implementation where every method across all classes is
assigned a unique slot and there is only one vtable pointer in an object. If this restriction were
not in place, it would then be ambiguous which method pointer to put in the `M` vtable slot for
class `D`.

## Private

Private members are private to the instance not shared between instances of the class. This
difference from the other OO languages is because one can subtype a class. If a class could access
other objects' private fields, it could attempt to access a private field on a type that is a subtype
but doesn't have that field.

## Override Fields

Non-private fields can be overridden with other fields or properties. This is necessary because a
class can subtype another class and must have a way of providing behavior for the accessible fields.
This also has the advantage of allowing direct use of fields rather than making everything a
property with "`get`" and "`set`" functions.

## Against Convenience Initializers and Initializer Inheritance

Swift has convenience initializers and allows them to be inherited to subclasses when all designated
initializers are overridden. This seems to add lots of complexity to the language and add relatively
little value. As such, I think it is a design mistake and something like it shouldn't be added to
Azoth.

Having initializers that call other initializers instead of the base class initializer makes sense.
However, C# demonstrates that can be achieved without the complexity introduced in Swift. If you try
to read the docs on
[Initialization](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/)
in Swift, esp. the sections [Initializer Delegation for Class
Types](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/#Initializer-Delegation-for-Class-Types),
[Initializer Inheritance and
Overriding](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/#Initializer-Inheritance-and-Overriding),
and [Automatic Initializer
Inheritance](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/#Automatic-Initializer-Inheritance)
you'll see the amount of confusing complexity these features add. That seems complex and difficult
to keep straight in one's head.

It is generally accepted now that using too much inheritance or having deep inheritance hierarchies
is a bad idea. It is better to use protocols/interfaces/traits. Furthermore, Swift really encourages
the use of structs over classes. So there shouldn't be too many classes that inherit from another
class. Among those that do, initializer inheritance only kicks in when the subclass implements all
designated initializers and there are convenience initializers to inherit. That ought it be a small
percentage of all types then. So in that small percentage of cases, you have avoided the need to
redeclare a few constructors on the subclass? Sure, that is nice, but not high-value. Not something
you can't live without.

The only answer I've found so far is that Objective-C had a similar feature of initializer
inheritance.
