# Independent Generics

Up to this point, the design of independent generic parameters has ignored issues of `iso` and of
move types. This is because the default constraint is `aliasable` which does not include `iso` or
`temp iso`. However, with the introduction of hybrid types it seemed important to consider how one
would work with an array of structs (not struct references). That is, after all, exactly what most
games and high-performance code needs to do. Considering this has really brought to the forefront
that I don't know how it should work and perhaps don't have a proper mental model of independent
parameters.

Contents:

* [Defining Independent Parameters](#defining-independent-parameters)
* [The Problem](#the-problem)
* [Basics of Generics](#basics-of-generics)
  * [Reification and Variance](#reification-and-variance)
  * [Overloading and Self Capability Constraints](#overloading-and-self-capability-constraints)
  * [Combining Capabilities](#combining-capabilities)
* [Independent Parameters](#independent-parameters)
  * [Capabilities are Constrained by Variance](#capabilities-are-constrained-by-variance)
  * [Can't Reify Capability](#cant-reify-capability)
  * [Containers or Collections](#containers-or-collections)
    * [Container or Collection Operations](#container-or-collection-operations)
  * [Sharing Between Values](#sharing-between-values)
  * [Reified Ownership](#reified-ownership)
* [Questions](#questions)
  * [Can Independent Parameter Capabilities be Constrained?](#can-independent-parameter-capabilities-be-constrained)
  * [Do Independent Parameters Default to or Always Allow Move Types?](#do-independent-parameters-default-to-or-always-allow-move-types)
* [Ideas](#ideas)
* [Options](#options)
* [Decision](#decision)

## Defining Independent Parameters

What are independent parameters? Regular generic types are higher-order types. They can be thought
of as being a sort of template for declaring many different types (e.g. `class Factory[T]` is a
template for declaring many classes that are factories). When I say template, I do not mean a C++
style template. In Azoth, as in C#, all generic types must type check when they are declared. They
are not type checked after the generic arguments are substituted. In Azoth, the reference capability
is part of the type. Thus it is part of the generic argument (e.g. `Factory[iso Shape]` and
`Factory[const Shape]` are distinct types). Additionally, in Azoth reference capabilities are
deep/transitive. For example, any field accessed from a `const` instance will also be `const`
(except if it was `id` it will still be `id`). This does not mean that one can't have a `const
Factory[iso Shape]` since the factory could be constructing new instances of `Shape`. But it does
mean that such a factory couldn't be handing out references to some internal `Shape` field because
they would have the type `const Shape`. The reference capability being part of the type and
reference capabilities being transitive is very limiting for collection classes like `List[T]`. If
those rules applied to a list, it would mean that a read `List[mut Shape]` could not be used to
mutate the shapes. All shapes that one accessed from the list would be read only (since they are
effectively stored in fields in the list). It would also mean that one couldn't freeze the elements
of a list without freezing the list itself as well. Nor could one freeze the list without freezing
the elements. Independent parameters exist for cases like `List[T]` to provide that flexibility.

Consider a `class VarBox[T ind]` that acts as a box containing a single instance of `T`. Note that
the generic parameter `T` is independent. What the independence means is that the field of type `T`
in a `VarBox` instance is not transitively controlled by the `VarBox` instance. Instead, it is
almost as if the field was declared as a variable where the `VarBox` is referenced. The type `self
|> T` does not modify the capability of `T` and is always equal to just `T`. It means that the
capability on the generic argument can vary based on the code using the `VarBox` instance. It is
possible to have types like `const VarBox[mut Shape]` which would allow the shape in the box to be
mutated but prevent one from putting a different shape into the box.

Another way to think about what independence means is in terms of the object graph. Reference
capabilities control what can be done to the object subgraph reachable through a reference. An
independent parameter introduces a sort of layer or boundary in the object graph between then
generic type and the independent parameter. Even though the collection references the elements, they
are treated as if they weren't referenced by the collection. Instead, they are treated as if they
were referenced by a parallel reference existing alongside the reference to the collection. It is
this imaginary parallel reference which manages the capability applied to the collection elements
independently of the reference to the collection itself.

## The Problem

The problem was first noticed in the interface of the `Raw_Hybrid_Array[P, T]` class. However, to
avoid the complexity of thinking about the right way of handling the low-level details and unsafety
and to avoid some complexity about initialization we'll consider the design of a `Hybrid_List[P, T]`
class.

Imagine we will create an list of enemies for a game following an entity-component system (ECS)
approach. The `Enemy` struct may be a discriminated union of enemy types or it may be a single
struct with fields with variant meaning and empty slots in some cases. The contents of the `Enemy`
are not relevant to our example. Perhaps along with these enemies we want to store some counts of
how many enemies there are of certain types or with certain properties. Or perhaps we want to keep
an index to which one is currently attacking the player. This data can be stored in an
`EnemiesSummary` struct that is the prefix value for our hybrid list. In both cases, we wish to
store the actual struct in the list not a reference to the struct stored elsewhere.

Here is some code we might find in various places in the game codebase:

```azoth
public fn play_level(self, curren_level: Level)
{
    // We construct an initially empty list. The type parameters are `iso` to reflect that we are
    // expecting the list to contain the actual structs which if they were stored in variables would
    // be of type `iso`. The list itself is just `mut` because we won't need full isolation.
    let enemies: mut = Hybrid_List[iso EnemiesSummary, iso Enemy](EnemiesSummary(...));

    // We load up some enemies for the current level. We pass a reference to the `EnemiesSummary`
    // prefix so that it can be updated for the enemies we will add. If the `enemies.prefix` and the
    // `enemies` elements follow the normal rules of capabilities, they must be upcast to `mut`
    // before they can be used. So the effective type of `enemies` ought to already be
    // `mut List[mut EnemiesSummary, mut Enemy]` even thought it internally stores the origin for
    // those structs. Despite that being the list type, the `add_range` method must still take
    // ownership of any new enemies being added. Thus, it expects an `Iterator[iso Enemy]` argument
    // and that is what `load_enemies()` returns since it constructs the enemies from the level
    // configuration.
    enemies.add_range(load_enemies(current_level, enemies.prefix));

    // enter game loop
    while not level_complete
    {
        // To iterate over the list, the list itself must be temp const, but the elements can remain
        // mutable (i.e. they are independent).
        foreach enemy in enemies
            => draw(enemy, screen);

        foreach i in (0..<enemies.count).reverse()
        {
            // Even though the list was constructed with `iso Enemy` the return type of `at` must
            // have another capability that is consistent with the current capability of the list.
            // It must be at least `mut Enemy` but the list could be upcast to
            // `Hybrid_List[EnemiesSummary, Enemy]` and the return type would be `read Enemy`. Still
            // if the return type were `mut Enemy` then covariance could handle that.
            let enemy: mut = enemies.at(i);
            if not enemy.dead
            {
                if enemy.hit_points == 0 => kill(enemy); // mark enemy dead
            }
            else if now - enemy.died > 1 min
            {
                // The `remove_enemy` either discards an item from the list or takes it and then
                // destructs it. Either way, the isolation of the items must be recovered. So it
                // takes a `mut Hybrid_List[mut EnemiesSummary, iso Enemy]`. This is not a `move`
                // but a recovery is happening and it shouldn't be implicit since this is not a
                // `self` parameter.
                remove_enemy(enemies, i);
            }
        }
    }

    // `enemies` goes out of scope here. Despite the fact that `Hybrid_List[P, T]` is not a move
    // type it contains `EnemiesSummary` and may still contain `Enemy` instances. These are move
    // types and may have destructors. Isolation needs to be recovered for both the prefix and the
    // items and then they need to be destructed. Thus the hybrid list has become a move type and
    // needs to have a destructor which can destruct the prefix and items.
}
```

This code example and the comments demonstrate the unique challenges of managing a type with an
independent parameter.

If `Enemy` were a `move class` instead of a `move struct` much of the above code would be unchanged.
However, it would no still be necessary to recover isolation on the elements to `remove_enemy` or at
least isolation on the removed item. However, since remove_enemy is a separate method, the isolation
will have to be recovered before calling the method. Also, it would be fine to add `mut Enemy` into
the list as long as isolation can be recovered for them later when they go out of scope. For
example, one could add a `mut Enemy` to the list while retaining a reference to the enemy in order
to do some work. For example, to draw it to the screen and then release that reference allowing
isolation to be recovered again when the list goes out of scope. But assuming the list still owned
the enemies, it would still be necessary to recover isolation for them at the end and destruct them

At the same time, the `Hybrid_List[P, T]` member declarations must handle cases where the type isn't
a move type. Review the above code but instead consider what would be necessary if `Enemy` was a
non-move class. The type mechanics would be greatly simplified and it wouldn't be necessary to
recover isolation to remove items from the list or to destruct the list. However, there still may be
cases where the code wishes to recover isolation of the items for some operation or to freeze the
items. Once the items were frozen, then `at()` should return `const` instead of `mut`. That change
is not simple covariance on `mut`. Likewise once frozen, items added to the list must be `const`.

Let's try to declare the method signatures for the `Hybrid_List[P, T]` class and examine the
challenges. (I'll use the syntax as it existed before considering this problem at all.)

```azoth
published class Hybrid_List[any P ind readonly out, any T ind readonly out] <: // ...
{
    // No problem with `P` here as it will match the capability at the point of declaration
    published init(mut self, prefix: P) { ... }

    // Here `T` needs to variously be:
    // * `iso T` for owned `move struct`
    // * `mut T` for `move class`
    // * `const T` for frozen `T`
    published fn add_range(mut self, items: Iterator[T]) { ... }

    // Here `T` needs to have whatever the current capability of `T` is for the list. Though there
    // should probably also be a way to consume the items of the list. That would take
    // `temp iso self` and return `mut Iterator[iso T]`. However, if that method were called on a
    // a list whose items had been frozen, it would need to return `mut Iterator[const T]`.
    published fn iterate(temp const self) -> mut Iterator[T] { ... }

    // Here `T` should not be `iso`, but should reflect whatever the current capability of the list
    // items is. For example, if the items have been frozen then it ought to return `const T`. If
    // `T` is interpreted to have the current capability, then this could be accomplished with
    // `aliasable T`. However, that is inconsistent with the fact that capabilities are reified for
    // regular generics.
    published fn at(self, index: size) -> T { ... }

    // How does one support recovery of isolation for T or freezing T? Does that require methods?
    // If the list contains at least one item then `freeze list.at(0)` ought to freeze all items.
    // (Assuming freezing properly flow types all `read` references to now be `const`.) But that
    // requires the list to contain an element. It should definitely be possible to freeze the items
    // of a currently empty list so that all items added in the future must be `const`. Also, there
    // doesn't seem to be any equivalent for recovery of isolation. Do we need to add syntax like
    // `freeze list.T` and `recover list.T`?

    // Currently, it would be illegal to declare a destructor, but it seems declaring one is
    // necessary.
    published delete(lent mut self) { ... }
}
```

## Basics of Generics

There are a few things to keep in mind about how generics and capabilities work in Azoth as this
problem is solved.

### Reification and Variance

Regular generic parameters accept and capture not only a type, but a reference capability. That is,
given a class `Example[T]` not only a type, but also a reference capability must be passed as the
generic argument for `T`. For example, one might declare a variable of type `mut Example[mut User]`.
Even in cases where no reference capability is listed, a capability is specified, it is just the
default capability of `read` or `const`. Both the type and capability are *reified* into the
instance meaning that the instance is forever an instance for that type and capability. Reflection
hasn't been specified for Azoth yet, but when it has both the type and capability will be
inspectable. They are a property of that instance just as generic arguments are properties of
instances in .NET. So, unlike most other types, when a generic parameter type `T` is used, it does
not need to have a capability because the capability is "baked in" to the type. It is important to
remember that the generic type `T` isn't a `read` type, but is rather a type with some unknown, but
possibly restricted, capability.

Since Azoth has generic parameter variance, it may be thought that this reification can't be done or
that the capability changes on this instance. Neither of those is true. Consider a `Channel[sendable
T]` class which acts as a channel for sending values between threads (a sort of thread-safe queue).
Here the parameter `T` is invariant which is the default. This is because the channel both takes in
and gives out instances of `T`. Thus it would not be safe for it to be variant. However, it can
implement two traits `Channel[sendable T] <: Source[T], Sink[T]`. Each of these are variant. The
former is declared `Source[T out]`. This indicates that `T` is used only in output positions within
the trait and so `Source` is covariant with `T`. That is if `A <: B` then `Source[A] <: Source[B]`.
This is true not only of the named portion of the type, but of the reference capability. So, given a
`Source[iso Message]` it can be passed to a method expecting a `Source[Message]` since `iso <:
read`. However, the instance itself does not change type. The instance still produces `iso Message`.
The type reified into the instance is till `iso Message`. It is just that it is always safe to use a
`Source[iso Message]` where a `Source[Message]` could be used. The contravariant case is illustrated
by the declaration `Sink[T in]`. For a `Sink` if `A <: B` then `Sink[B] <: Sink[A]` (note the
reverse order). Thus a `Sink[Message]` can be used anywhere a `Sink[const Message]` is needed. In
other language covariant and contravariant are the only options. However, in Azoth reference
capabilities enable one more possibility which could actually be used on `Channel`. It can safely be
declared `Channel[sendable T readonly out]`. This indicates that while a mutable channel is
invariant on `T`, a read only channel is actually covariant with `T`. This is because all the
methods that take in a `T` are only available on mutable instances of channel. So given a read
`Channel[iso Message]` it is safe to use it as a read `Channel[Message]`. Again, regardless of the
variance, the type and capability are always reified into the instance. The code of the instance is
always running as if the original generic argument has been substituted into the generic class.

### Overloading and Self Capability Constraints

In Azoth, reference capabilities are flow types. This means that the capability of a variable may be
different on later lines that what it was declared with. For example, a `let s: mut Shape` could be
a `const Shape` later in the method if in between the shape was frozen (i.e. `freeze s`). In the
presence of loops flow types can become circular and a fixed-point must be calculated. Given that,
Azoth does not support overloading a method or function on reference capabilities. For example, it
wouldn't be possible to declare two overloads `fn example(s: mut Shape)` and `fn example(s: const
Shape)` even though no shape could ever be both mutable and constant at the same time. This would
give an error that it wasn't possible to overload `example` on `mut Shape` and `const Shape` since
they are both `Shape`.

The `self` parameter of a method is, in many ways, just another parameter. Like other parameters it
has a capability that it requires of the self argument. This is declared before the `self` keyword.
The `self` parameter is really just a parameter with the associated type `Self`. The type `Self` is
the true type of the current instance. Since it is an associated type, it does not have a reference
capability reified into it the way a generic parameter does. Thus the declaration `mut self` is
effectively shorthand for a parameter `self: mut Self`. Notice that the `self` parameters of the
methods of a class can each have different capabilities. Thus they require different capabilities to
be called on an instance. The capability of an instance is *not* reified into that instance. It
exists on the various references to that instance and as discussed earlier it can change over time.
A single instance can also have multiple references to it each with different reference capabilities
as long as those capabilities are compatible with the existence of the other references.

Like other parameters, methods cannot be overloaded on the capability of the `self` parameter. It is
illegal to declare two overloads `fn method(self)` and `fn method(mut self)`. Often a single
capability is enough given the subtyping relationship between them. For example, `fn method(mut
self)` can be called on an `iso` instance since `iso <: mut` and recovery makes it easy to recover
the `iso` later if needed. However, there are times where this isn't sufficient. To support these
cases, a capability set may be used on the self parameter. This effectively makes the method generic
over the capability of the self parameter. Other types can then be constrained/modified by the
capability the method was called on via a viewpoint type. For example, `get property(readable
self) -> self |> mut Shape` declares a property which can be called on a reference so long as it can
be read from (i.e. not `id`). The return type then varies with the capability the method was called
on. For example, when called on a `mut` reference, the return type with be `mut Shape` but when
called on a `const` reference the return type with be a `const Shape`. These features allow both
more flexibility in what methods can do and avoids needing to declare many overloads that would be
identical except for capability differences.

### Combining Capabilities

There are three ways to manipulate the capabilities of regular generic types. The first way is a
*capability viewpoint*. Consider a regular generic type `T` with a reified/captured capability. For
a capability *c*, the type `c |> T` is the type that would result from accessing a field of type `T`
from an instance with the capability of *c*. Capability viewpoints can't be applied to regular types
because they would simply evaluate to a different capability. For example, `const |> mut Shape` is
just the type `const Shape`.

The second, self viewpoint types, was discussed in the previous section. The type `self |> T` is the
type that would result from accessing a field of type `T` from an instance with the capability of
`self`. If `self` is known to have a specific capability *c*, then this is equivalent to `c |> T`.
However, the self parameter can also be constrained by a capability set. In that case, `self |> T`
will be whatever type would result from the capability of the reference the method was called on.
This possibility means that self viewpoints can be combined with regular types (e.g. `self |> mut
Shape`).

The third way is that a generic parameter type can be constrained to a capability set. The
capability will be upcast if necessary to make it fit in that set. For example, the type `sendable
T` is the type `T` with either the capability `const` or `id` depending on what its current
capability is. This cannot be applied to regular types because they would simply evaluate to a
different capability.

One interesting thing to note is that because `Self` is an associated type without a capability, it
cannot be used as a stand in for the full type of the self reference. Instead that type is
effectively the type `self |> iso Self`.

## Independent Parameters

Now let's consider what we know to be true about independent parameters and some of the
possibilities as well.

### Capabilities are Constrained by Variance

With a regular reference (e.g. to a `Shape`), multiple references can exist to the same shape
instance each with a different capability so long as all those capabilities are compatible with the
existence of the others. However, with an independent parameter, this possibility is constrained by
variance of the parameter. For example, it is valid to have `mut Shape` and `Shape` references to
the same instance. However, it is not valid to have `mut List[mut Shape]` and `mut List[Shape]`
references to the same list. This is because the list is `readonly out` over `T`. So as long as the
list is mutable, the capability of the list items is invariant. But once the list is readonly, the
capability can now vary. For example, it is valid to have `mut List[mut Shape]` and `List[Shape]`
references to the same list (note that the second list is not mutable). This is safe since it is not
possible to insert a non-mutable shape through the second reference.

So far, this is consistent with what we know of non-independent generic parameters.

### Can't Reify Capability

An independent parameter still reifies the plain type of the parameter, but does not reify the
(full) capability (e.g. `List[mut Shape]` reifies that the parameter is `Shape` without reference to
what capability it has). To see why this must be so, consider a list constructed as a `mut List[mut
Shape]`. This list can have many shapes added to it, then the list items can be frozen so that it is
now a `mut List[const Shape]`. This change in capability is completely at odds with the subtype
relationships between the capabilities. Indeed, if the parameter was treated as a normal generic
parameter with a reified capability, it would be possible to circumvent the capability safety.

Consider the class below:

```azoth
public class Illegal[T ind]
{
    public let value: T;
    let modifier: const (T) -> void;

    public init (.value, .modifier) { }

    public fn illegal(mut self) => .modifier(.value);
}
```

If that class were legal, one could do the following:

```azoth
public fn double_size(s: mut Square) => s.side_length *= 2;

let i: mut = Illegal[mut Square](Square(4), double_size);
freeze i.value; // freezes just the independent parameter

// Now, i: mut Illegal[const Square]
// But this call would mutate the Square!
i.illegal();
```

In this example, the body of the class treats the independent parameter `T` as if it were a regular
generic parameter with reified capability. In that case it would always be safe to call
`.modifier(.value)` because the parameter type would be compatible. The type `T` and its capability
do not change in the class regardless of what references exist to it. However, since this is an
independent parameter, this is not legal. Specifically, the field `modifier` is not allowed because
it does not allow the type `T` to vary its capability as needed. In the example, the moment the
`Square` is frozen, the `modifier` field is already invalid because it expects the parameter to
still be `mut Square`.

### Containers or Collections

The limitations of independent parameters mean that they are essentially useless outside of
collection and container types. The generic class declaring the independent parameter can never be
sure what the reference capability of the parameter is now and how it has changed. This means that
there is essentially no code that can be written in the generic class to work with the independent
parameter beyond taking one in, storing one in a field (or another collection/container), and giving
out one from a field. Any code to directly manipulate a value with the type would have to work on
all capabilities. Thus from the generic class's perspective, the independent parameter is accessed
with the capability `id` while not necessarily being a reference type and so not supporting
reference equality or identity hash. Yet, the creation of a value would require that it have the
`iso` capability since that is the only capability that is a subtype of all other capabilities.

One might be able to declare odd types that aren't collections but nevertheless use generic
parameters. While these types may be logically correctly, they may not actually be supported in
Azoth. These include:

* A `Factory[T ind]` that internally calls a method producing `iso T` to produce the instances. (It
  is unclear how one declares the initializer so that it can take a method producing `iso T`.)
* A `Computation[T ind] where T <: SomeTrait` that uses only `let const` fields to compute something
  on instances of `T` that it is given.

However, other than these odd types, the only real use of independent generic parameters seems to be
for collections/containers. That is, types which contain and in some way organize or manage zero to
many instances of the independent parameter. These can be categorized by how many instances they
manage. This categorization is useful when solving the problem of independent parameters because all
of these collection/container categories should be supported by the solution.

Categories of Collections/Containers:

* 0 to many: list, stack, queue, etc.
* 0 to 1: optional type
* 1: box
* 1 to many: rooted tree (a tree represented by its root node that always has a value), bodies in a
  solar system, words in a sentence, sides in a polygon.
* Fixed N: array

#### Container or Collection Operations

For the implementation of containers and collections with independent parameters, we should ensure
that all the basic operations are supported properly by the type system. Those operations are:

* add item
* take item (i.e. remove from collection but keep a reference outside of the collection)
* discard item (i.e. remove from collection and do *not* keep a reference out of the collection)
* replace item (replaced item either taken or discarded)
* alias an item (e.g. access an element of a list)
* recover isolation of the items
* freeze the items (for non-move types which can be frozen)

### Sharing Between Values

The way that the compiler enforces reference capabilities is by keeping track of sharing sets. A
sharing set tracks references whose object subgraphs may intersect. This could mean that it is
possible the subgraph reachable by one reference is reachable from another reference. But it could
also mean that it is possible that the same object is transitively reachable through both
references. Sharing sets control whether values can be recover isolation or be frozen. If all other
references in a sharing set are readonly, then it is safe to freeze a reference since no other
references could possibly be used to mutate a object reachable through that reference. However,
isolation is more strict. Isolation can only be recovered if all other references to the subgraph
are `const` or `id`. Indeed, `const` and `id` references are not even tracked in sharing sets
because they cannot break isolation nor be used to induce sharing between two other references. Thus
the rule is actually that isolation can be recovered if there is no other references in the sharing
set.

Now consider the case of independent generic parameters. For a box, there is only one element in the
box and so sharing sets work the same as a regular single reference. However, for a collection like
a list, there are many elements in the list. There is then not only the question of how the list
elements are related to other references by sharing sets, but how the list elements are related to
each other by sharing sets. For example, imagine a list of `Employee`s where each employee contained
a reference to the employee that is their manager (except for the CEO or founders who have no
manager). Then it may be possible to freeze the employee objects since they will all become constant
at the same time and so every reference to a manager will also be a reference to a constant object.
But if one needs to recover isolation in order to delete and employee, one can't do that since any
given employee may be referenced by other employees whom they manage. Thus the sharing set must
somehow represent the fact that there are effectively multiple references to employees in the same
sharing set and that if they are frozen, they are frozen together. This is new and distinct from
anything required of sharing sets by other language features.

### Reified Ownership

While independent generic parameters cannot reify the reference capability of the generic argument,
it seems they do need to reify the "ownership" of the elements. Ownership is not a concept
explicitly defined in Azoth to this point. However, it exists implicitly for move types. For
non-move types, there is not owner. Reference types have many references that all share the
instance. Value types are copied so that each instance is an independent copy. However, move types
require that isolation be recovered by the code that received the original isolated reference and
that be used to destruct the value. This can be avoided by moving the value to another place (i.e.
transferring ownership). It seems that independent generic parameters must reify whether they have
ownership for several reasons.

First, they must be able to determine if they are responsible for recovering isolation for an
element and calling its destructor.

Second, for hybrid types (i.e. structs) having ownership means that the value is stored
directly/inline rather than storing a reference. This means they the object layout actually changes
for generic types with ownership.

Third, the previous to requirements mean that the method declarations must be such that the
effective types of the methods are different for move types than non-move types. For example, a
method to clear a list must have isolation of the elements recovered before being called for a list
of move types. However, for a list of non-move types that same method does not need to have
isolation recovered.

## Questions

### Can Independent Parameter Capabilities be Constrained?

The current design treats the independence of the generic parameter as completely distinct from the
constraint of which capabilities can be used with the generic parameter. This is reflected in the
syntax which puts the capability constraint set before the generic parameter name and the
independence after. But maybe it doesn't make sense for a generic class to be able to constrain the
capabilities of a generic parameter. In all other aspects, the capability is managed by the code
that uses the generic class. Indeed one could imagine that as the number of capability constraint
set options grows that one of those would actually be incompatible with an independent parameter.
Say because it prevented upcasting the capability in a way that ought to be allowed. Indeed,
`readable` may already have this problem since it does not allow `id` which is the base capability.

Perhaps instead, independence should be treated as if it is an alternative to constraining the
capability. Syntactically it would be declared in place of the constraint. That would be consistent
with the fact that independent parameters remove the reification of capabilities but replace it with
the reification of ownership. Thus before the generic parameter either a capability constraint set
could be declared or independence could be declared.

### Do Independent Parameters Default to or Always Allow Move Types?

With regular generic parameters, they default to not allowing move types and to the `aliasable`
capability set. This is because the `iso` capability and `move` types impose so many restrictions on
how a type can be handled. The previous question considered whether the capability should be
constrained. This question asks how move types should be handled. Options:

* Independent parameters default to not allowing move types (this matches regular generic
  parameters, but will not be the most common use of independent generic parameters)
* Independent parameters default to allowing move types (fits with their use but requires additional
  syntax to disallow move types when desired)
* Independent parameters *always* allow move types (are there any use cases this prevents?)
* *All* generic parameters, including independent, default to allowing move types (this may make
  generic types difficult to write or require many declarations that move types are not supported.)

## Ideas

**TODO:** write up ideas that might be used in a solution

## Options

**TODO:** write up possible options to solve the problem.

## Decision

**TODO:** make a decision which option to go with.
