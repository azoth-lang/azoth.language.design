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
  * [Do Independent Parameters Default-to or Always-Allow Move Types?](#do-independent-parameters-default-to-or-always-allow-move-types)
* [Ideas](#ideas)
  * [Thoughts](#thoughts)
  * [Reified Ownership Produces Cases](#reified-ownership-produces-cases)
  * [More thoughts](#more-thoughts)
* [Options](#options)
* [Decision](#decision)
  * [Meaning of `iso`](#meaning-of-iso)
  * [Ownership of Structs](#ownership-of-structs)
  * [Resource Types](#resource-types)
  * [Ownership is Distinct from Isolation](#ownership-is-distinct-from-isolation)
  * [Owned as an Additional Capability](#owned-as-an-additional-capability)
  * [Independent Parameters with Owned](#independent-parameters-with-owned)
    * [Evaluating this model](#evaluating-this-model)
  * [Issue: Struct Ownership when Freezing](#issue-struct-ownership-when-freezing)
  * [Result of Moving a Struct](#result-of-moving-a-struct)
  * [`temp iso` Doesn't Have Ownership](#temp-iso-doesnt-have-ownership)
  * [Other Decisions](#other-decisions)
  * [Remaining Question](#remaining-question)

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

### Do Independent Parameters Default-to or Always-Allow Move Types?

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

### Thoughts

* Imagine specifying it like the capability of self (e.g. `mut self[aliasable]`)
  * Do you then use self viewpoint `self |> T` to describe the type? It seems like that shouldn't be
  needed since generic parameters normally contain the capability.
  * But if you say that it doesn't contain the capability, then what is the capability of a field of
  `T`? Does one have to constantly write `self |> iso T` to just get the current capability of `T`?
  * How does this help work with varying types based on whether they are move or not?
* Maybe you describe it as if you had to move them and that is weakened for non-move types. For
  example, maybe you use `own T` when you want to talk about a place you use `T` that should have
  ownership if it is a move type. You have to use `move` on these things. But `T` refers to `T` with
  the current capability.
* Alternatively, `T` refers to an owned `T` so that a field is just `T` and you have to use
  something like `self |> T` to have a T that is restricted. What's good about that is it makes
  sense of the fact that you don't have `self |> T` for fields.
* However it works, you need to think about how it fits with having a field that is another
  collection. You need to pass along the type.
* Even if an independent parameter didn't allow move types, how does it deal with `iso`? What
  capabilities do fields then have?
* You can't just say use `iso T` as the owned type because how does one get an alias to a value?
* Consider list clear, the signature must somehow convey that if the list has ownership, then
  isolation of the elements needs to be recovered to call it, but if the list doesn't have ownership
  then it can be called on any mutable list. The self parameter is somehow like `mut self[own
  implies iso T]`. Basically the constraint needs to either be `iso` or `any`.
* If owned/unowned has been reified into `T` then perhaps what is needed is to use a viewpoint of
  it. So maybe `T |> iso` produces wither `iso` or `any` depending on whether it was reified with
  owned or unowned? But how do you express `T` with either `iso` or current? * Is `mut |> iso` still
  iso since the field has isolation even through a mutable reference?

### Reified Ownership Produces Cases

If we think that owned/unowned must be reified. Then the container/collection operations produce
various cases. Consider a collection with generic parameter `T` reified with `owned X` or `unowned
X` and current capability `c`. Then the cases are:

* add item parameter, take item return type (tied to self parameter), replace item parameter, field
  type
  * owned: `iso X`
  * unowned: `c X`
* take item self parameter, discard item self parameter, replace item self parameter, clear self
  parameter
  * owned: `mut self[iso]`
  * unowned: `mut self[any]`
* alias an item return type
  * owned/unowned: `c X`

Those are the cases for the type syntax. You still need syntax for:

* recovering isolation of independent parameter
* freezing independent parameter
* destructor for collection

### More thoughts

* The capability/sharing system acts as if there is a reference to the items carried along with the
  reference to the collection. What if we give that reference a name and treat it like an additional
  parameter. It seems to give a good way to constrain that, but mostly constraining it isn't what is
  needed. It also doesn't give an obvious syntax for the type cases needed for collections
  * `class List[independent t: T]` names the additional reference `t`
  * `public fn at(self, any t) -> t |> T` OR `public fn at(self[any t]) -> t |> T`
  * This gives a syntax for freeze of `freeze list.t`. One would need a recover keyword to do
    `recover list.t`
* What if we give a variable for the capability of an independent parameter?
  * `class List[independent c T]` names the current capability `c`, `T` doesn't have a capability
  * `public fn at(self) -> c T` works, but how do you get the `iso` or `c` and `iso` or `any`
    needed?
  * Is a field `let v: c T`? That d
* Any combining of `self |> T` feels wrong because the capability of `self` doesn't affect `T`
* What is needed is a capability set that is either `any` or `iso` depending on whether the type has
  ownership. Call that `x` for now. Let `T` represent the type with the current capability. Then
  according to regular capability constraint rules `x T` is either `iso X` or `c X` as needed for
  the first case and `mut self[x]` is either `mut self[iso]` or `mut self[any]` as needed for the
  second case and `T` is `c X` as needed for the third case. (No! that isn't quite right because if
  `c` is not `iso` it can't be constrained to `iso`.)
  * That being wrong is also why you can't do `iso T` with a regular generic parameter. **You can
    only constrain a generic parameter to a capability set that contains `id`. That ensures there is
    always a compatible capability.**
* While the last idea isn't quite right, a keyword that behaves that way is basically what is
  needed.
* Recovering isolation on list elements isn't quite a move but is similar in that it should really
  only happen as part of passing or returning the list. In some ways it is like a move because
  isolation is available in that scope. But when the function returns you still have a way to access
  the list elements. If they were moved, then you shouldn't be allowed to. Maybe `move` passes
  ownership via an isolated reference while `recover` just recovers isolation without passing
  ownership? It still isn't clear how one makes clear which aspect of a list you are passing. Given
  a list `l`, how do you express each of:
  * move the list, share the elements
  * share the list, share the elements
  * share the list, move the elements
  * share the list, recover isolation of the elements
* If you have a list and pass ownership of the items to the list, when the method returns, do you
  now have an empty list that you can start adding items to? Perhaps moving ownership leaves behind
  a `never` type so the list becomes `List[never]` and you can no longer access the elements or add
  them, but out could still check the `count`.
* Maybe `own` is a separate stronger capability than `iso`. (This seems to make sense and calls back
  to ideas from Adamant when the capabilities controlled memory. That makes sense since here the
  capability controls the resource lifetime.)
  * In this design a `List[iso Shape]` doesn't have ownership. Instead a `List[own Shape]` does.
  * But aren't `own` and `iso` mutually exclusive? I think they have to be. If you've truly lost any
    other references to a move type, then the one remaining `iso` reference must have ownership. So
    you can't have an `iso` reference to a move type. That argues that there is not a distinct
    capability. Instead `iso` behaves slightly differently for move types (as original thought). It
    must just be that independent parameters interact with `iso` in interesting ways.
* Maybe what is needed is a different keyword than `independent`. One that names it as a collection
  element. Then the missing `x` keyword could follow the naming scheme of collection elements.
  * `member`
  * `contained`
  * `boxed` - confusing with C# boxing
* What if we use `ind` as that missing keyword
  * `class List[ind T readonly out]`
  * `let items: mut Bounded_List[ind T]`
  * `fn push(mut self, value: ind T)`
  * `fn pop(mut self[ind]) -> ind T` the use with `self` seems wrong
  * `fn at(self) -> T`
* Maybe we can outlaw having fields be `iso` or `temp iso`. No, that conflicts with the fact that
  a move type can have ownership through fields of other move types.
* What if ownership was a sort of property that is independent of other things? You could have an
  `own mut X` reference. Then you could restrict `iso` to not be on fields and have a tag that tells
  you something has ownership and will need to call the destructor. An `iso X` reference would
  implicitly have ownership for move types since this is the only reference. Transferring ownership
  would require recovering isolation because one couldn't guarantee that the new owner wouldn't
  destruct it while you still had a reference. But it still isn't really clear that this adds value
  over just having `iso` imply ownership on move types. I guess it makes the reified property of
  ownership explicit.
* PROBLEM:
  * If all structs are `move`, then you can't use a struct field in a class unless that class is
    `move`. That is a problem for performance optimizations that want to avoid a layer of
    indirection by using a struct. Maybe we need structs that aren't `move` but still all struct
    types would enforce that there is an owning binding and other references don't outlive it.
    * maybe `move` types are renamed `resource` types and structs don't have to be them but structs
      still enforce the recovery of isolation without destruction to avoid references outliving the
      value.
    * Thus non-resource structs could be kept as `iso` fields in classes.
    * Would you then have "struct traits" that acted like structs even though they were traits?
* What if we use `own` as that missing keyword. That fits with the fact that it is related to
  ownership and might have to be moved.
  * `class List[ind T readonly out]`
  * `let items: mut Bounded_List[own T]`
  * `fn push(mut self, value: own T)`
  * `fn pop(mut self[own]) -> own T` the use with `self` seems wrong
  * `fn at(self) -> T`
* It seems like a modification on `iso` would be better. Something that means possibly isolated or
  isolated if owned.
  * `iso?`
  * `iso/T` and `iso/any`
  * `iso/? T` and `iso/any`
  * `iso|? T` and `iso|any`
  * `iso|cur T` and `iso|any` but then do you have to use `cur T` for the other case?
  * `iso|* T` and `iso|any` and `* T`
* Should an independent parameter `T` be modeled more like an associated type that doesn't have a
  capability? It could then have two special capabilities. One that means `iso` if owned and another
  that means current capability. But would that let you say `mut T` since it take a capability? This
  is very different than how it is handled today where it is modeled as having a capability that is
  unknown.
* It really is like a second reference to the elements is being passed in. The elements are being
  accessed through that reference.
  * What if the basic capability is written `* T` meaning unknown or wildcard (or maybe `_ T`)
  * `fn at(self[any]) -> t |> * T` or `t |> _ T`
  * A syntax for `iso|any` is still needed (just use the underscore again?)
  * The full set with underscore is then:
    * `class List[independent t: T readonly out]`
    * `let items: mut Bounded_List[_ T]`
    * `fn push(mut self, value: _ T)`
    * `fn pop(mut self[_]) -> _ T` the use with `self` seems wrong
    * `fn at(self[any]) -> t |> _ T` possibly any could be the default so that it could be ignored
    * However, the `_` is almost totally redundant. If it were the default, then one would only need
      a new syntax for `mut self[_]`.
* It may not make sense to pass ownership of the items of a list without passing ownership of the
  list itself. What is good about that is that you still need to `move` the whole list. But there is
  still the question of how to pass an isolated list without isolated elements.
  * If you pass a list with isolated elements, the elements may not be isolated afterward and you
    need to recover isolation again.
* How does a class declare that it owns a struct, but then allow many references to that struct? If
  the field is `iso S` then won't the rules require that isolation of the field be recovered at the
  end of each method? This may be a killer argument for ownership being partially independent of
  `iso`. When passing ownership you need `iso`, but once a field or local can have ownership without
  being isolated. Maybe owned is a weaker capability that isolated. So isolated implied owned but
  not vice versa? `iso <: own <: mut`. Note that `own` is not a subtype of `const` because having
  ownership doesn't mean there aren't mutable references to it.
  * `own` would be a capability that only applied to `struct` or things that could be `struct`
  * This would enable the rule that `iso` and `temp iso` can't be used for fields.
  * `temp const` may be like `iso` in that it can never be directly used, it is always cast up
    before being accessed.
  * It seems like you could just use `own` everywhere and `iso` would be a sort of ephemeral type.
    But then you could just rename that to `iso`.
  * What if `iso` isn't a reference capability at all? Joe Duffy admits it is different. **What if
    it had a distinct syntax saying that you needed isolation to assign into a variable or field but
    was never a part of the type?** Then `own <: mut` that exists only for structs but requires
    taking isolation to assign into it. Or maybe `iso` is a reference capability, but it is
    ephemeral. It can be used on return types and it is the result of `move` expressions but cannot
    be the type of a field or variable because accessing the field or variable would break
    isolation. For local variables one can simply `move` into them to express that isolation is
    recovered. However for parameters, one needs a different way to declare that one requires a move
    expression to pass the argument. Essentially, the parameter type is `iso`, but the variable in
    the method is not. Put another way, it seems valid for the public signature to have an `iso`
    parameter, but not for the local variable to.
    * It seems like `lent iso` may be nonsensical. If you pass an `iso` reference then you no longer
      have any reference to it. In what sense can you then lend it and not be concerned about
      whether it is mixed with other things. So maybe, parameters can be either `lent` or `move`. A
      `move` parameter means that it actually expects `iso`. The question is how does that interact
      with generic independent parameters? Can they be `iso`? Perhaps they can't anymore since `iso`
      is an ephemeral capability. When considering the method type, `move` would disappear and the
      parameter would become `iso`.
    * Would regular generic parameters limit themselves to non-ephemeral types? There are many cases
      where a generic parameter might be used as a parameter or return type, but if you wanted to
      store the type in a field then you couldn't allow `iso` or would need a constrained version of
      it. Generics might be a reason to not use `move` parameters since how does one say they take a
      generic parameter that is `iso`? Maybe instead one just allows `iso` in parameter and return
      types but not variables and fields?
* Isolation of list elements is strange. It is sort of like `temp iso` because a reference to the
  main list can create new references to the items. But it is stronger than `temp iso` because
  sequestered aliases can't be allowed since they would break if an item were removed from the list.
  * This seems to argue against a mental model where there is just an extra reference to the items.
    Instead, it really is that they are managed by the list, but more flexibility is allowed.
  * Returning to Joe Duffy's writeup of Midori, I notice that he describes list as an example of an
    object with "layers". He says that they made it complex for a while but then collapsed it back
    to the simple core. It makes it sound like there is a basic version of it that works without
    being too complex.
* Clarify what `temp iso` means. It doesn't just mean that as long as the reference exists there are
  no other references except sequestered references. Aliases can be created from the `temp iso` and
  as long as they exist, the other references must be sequestered. Like `iso` it is a sort of
  ephemeral property, but part of that property continues to exist. Also, `temp iso` doesn't have
  ownership.
* It should be possible to prevent the freezing of structs even if there are trait aliases because
  there will always be one variable in the sharing set that has ownership of the struct and that
  will have the struct type. So that binding in the sharing set can prevent freezing. But it is odd
  that you can't have const structs.
  * Alternatively, `const` references to structs could participate in sharing.

## Options

**TODO:** write up possible options to solve the problem.

## Decision

**TODO:** make a decision which option to go with.

### Meaning of `iso`

`iso` is a sort of almost ephemeral capability. A variable binding or field marked `iso` does not
maintain isolation. The `iso` capability instead acts as `mut` or some other capability. Putting
`iso` on a type causes three things:

1. Isolation must be recovered to assign into that value. (What about reassigning to `var`?)
2. That binding owns the the value. This only matters if it is a struct in which case this binding
   stores the value inline until it is moved out.
3. After assignment, isolation is not maintained.

The last point implies that an `iso` field in a class does not have to maintain isolation. That is
counter intuitive enough that it may make sense to change the keyword to something like `own` which
is a statement about the binding not about the ephemeral property.

### Ownership of Structs

An owned struct will recover isolation when it goes out of scope to ensure that there are not
references to the struct. For fields, their lifetime is managed by the garbage collector, so they
never actually go out of scope. Thus a field does not trigger recovery of isolation.

Questions:

1. Can you freeze a non-resource struct? It seems like you should be able to, but then `const`
   references to the struct would not be tracked and could outlive the struct.
2. What about the problem of `id` references to a struct being tracked?
3. Can up upcast struct references to trait references? If so how are rules about `const` and `id`
   enforced? Does the language need `struct trait`s that can be upcast to and enforce those rules?

### Resource Types

The previous concept of move types is replaced with resource types. There can be resource classes,
structs, and traits. Not all structs are resource structs. The resource attribute is inherited and
must be repeated on the subtype. Resource types can have destructors. A resource type forces the
recovery of isolation when the owner goes out of scope and then calls the destructor. This needs to
happen even for fields. To support that, a resource type must either be on the stack or a field in
another resource type. When the containing resource type is destructed is when the field will
recover isolation and be destructed.

### Ownership is Distinct from Isolation

Thinking carefully about ownership one realizes that ownership is a property that is distinct from
and still exists even after isolation is lost. An isolated reference has ownership since it is the
only reference to the object graph, but once aliases exist, ownership can still be retained. Two
examples demonstrate this. The examples use `File` and `Output_Stream` as examples of a resource
classes.

1. Consider a `mut List[iso File]`. As soon as one of the file elements is accessed, this becomes a
   `mut List[mut File]` since there is now an alias to one of the files. They are no longer
   isolated. That is consistent with how a variable containing a file would work. But now, consider
   passing this `mut List[mut File]` to a method. That method could add another file reference to
   the list but would not know that it needed to pass ownership. It could add an alias to a `File`
   owned by something else and then the list could destruct the `File` before it ought to be. Thus,
   we see that the list passed to the method must somehow encode in its type that while the files
   aren't isolated, they still are owned.
2. Consider a method or constructor which wraps an `Output_Stream` in a `Text_Writer` that owns the
   output stream. This needs ownership of the `Output_Stream` but it is fine if other references
   exist to it. It is just that those other references will have to go away before the `Text_Writer`
   goes out of scope. That will be ensured since they will be in the same sharing set as it. The
   method not taking an isolated reference means that it cannot destruct the `Output_Stream` since
   it couldn't recover isolation on it. But it can wrap and return it.

### Owned as an Additional Capability

Ownership being distinct from isolation implies that it is a separate capability `own` where `iso <:
own <: mut`. Resource types and structs have the `own` capability. Other types don't have it as a
distinct capability. It sort of collapses into `mut` for other types.

When an `iso` variable is accessed it must be upcast and its type changes via flow typing. I think
there may be an issue right now where it will change the variable to `read` if it is first accessed
as `read`. Instead, it seems that the capability should also be moved along the path toward `mut`.
For example, consider an `iso` variable that is first aliased by a `read` reference and then aliased
by a `mut` reference. For the second alias to be taken, the variable must still be `mut`. It is just
that variable references have flow typing so that if one freezes a reference, the variable can
change to `const` and does not block freezing the way a `mut` reference in some other sharing set
would. For structs and resource types, `own` is the next capability up and is what `iso` becomes
when a variable is aliased. That ensures ownership is retained and tracked. The example of a list of
files is then handled since a `mut List[iso File]` becomes `mut List[own File]` when accessed. It
cannot be upcast to `mut List[read File]` because list is `readonly out`. It can only become a list
of read only files when the list itself is read only (i.e. `read List[read File]`).

### Independent Parameters with Owned

In the model where `own` is its own capability, independent parameters do not reify the capability
or ownership. Instead, an independent parameter `T` conceptually has the current capability of the
parameter at all times. This capability is unknown within the class.

Additionally, capability constraints must be applicable even to independent parameters. This is how
the class can upcast the capability of the independent parameter to fit different situations.

#### Evaluating this model

Let's consider again the `Hybrid_List` from the problem statement. Under this model, nothing about
the capabilities of the independent parameters are reified. Only the bare type is reified.

```azoth
published class Hybrid_List[any P ind readonly out, any T ind readonly out] <: // ...
{
    // `P` has whatever capability the class was declared with
    published init(mut self, prefix: P) { ... }

    // `T` has whatever capability the parameter currently has. If `T` is an owned type, this will
    // be `own T`.
    published fn add_range(mut self, items: Iterator[T]) { ... }

    // Returns an alias of the current capability by upcasting with `aliasable T`
    published fn iterate(temp const self) -> mut Iterator[aliasable T] { ... }

    // Returns an alias of the current capability by upcasting with `aliasable T`
    published fn at(self, index: size) -> aliasable T { ... }

    // The destructor declaration needs some way to convey that it exists only for `own P` and `own T`
    published delete(lent mut self) { ... }
}
```

There is still a question how one recovers isolation on the elements or freezes them.

Now consider the collection methods and the cases that result:

* add item parameter, take item return (safe for `own` because the collection must still be `mut`),
  replace item parameter, field type
  * `T` with the current capability
* discard item self parameter (call `take()` and get ownership and the destructor will be called)
  * `mut self`
* alias an item return
  * `aliasable T`
* recover isolation of the items
  * ???
* freeze the items
  * ???

For recovery and isolation, it must be doable for the list, prefix, and items separately

### Issue: Struct Ownership when Freezing

Resource types can't be frozen because it doesn't make sense to destruct a constant value. Indeed the destructor

**Decision:** struct types cannot be frozen and as such cannot be `const`. Note this also means that
you cannot declare a `const struct`.

### Result of Moving a Struct

A binding with ownership has space allocated for the full struct. Thus after the value is moved, a
reference cannot be left behind. Symmetry with reference and value types might make one think that
some special `id` with ownership value should be left. The problem with that is that aliases could
be made to that and then would have to be tracked and when the value went out of scope it would need
to recover isolation. But it couldn't recover isolation because it is no longer safe to use the
value at all. Instead, a variable owning a struct must simply be inaccessible after the value is
moved out. This is like Rust which simply reports a compiler error, "use of moved value."

**Decision:** struct types cannot be `id`. No value is left behind when a struct is moved. The
variable simply cannot be used.

### `temp iso` Doesn't Have Ownership

It must be the case that `temp iso` doesn't have ownership and thus it isn't a subtype of `own`. So
does it need a different name?

### Other Decisions

**Decision:** all independent parameters allow any capability and allow move types. This simplifies
the language. Thus the syntax moves before (i.e. `class List[independent T readonly out]`).

### Remaining Question

* How does move/recover and freeze work with independent parameters
* Should we switch to using `iso`, `own`, and `const` to recover and freeze? If not, how does one
  express recovering ownership?
* How does one declare and write destructors for classes with independent parameters?
* How can a class conditionally have ownership without full independence? For example, a
  `Text_Writer` may or may not have ownership of the contained `Output_Stream` but it isn't a type
  parameter at all and it always needs to be able to mutate it. Thus it wants it to be either `own`
  or `mut`.
* How do trait methods deal with the fact that `Self` may be a struct or resource type? Does that
  impose too great of a restriction on what can be done with `self`?
  * Should a `const trait` assume that `Self` cannot be a struct or resource type or does that still
    require an explicit constraint on `Self` (e.g. `where Self: class, not resource`).
* How does one constrain to resource types or not? Options:
  * Possibly a resource type `where resource T` or `where T: resource?`
  * Not a resource type: `where T: not resource`
  * Not a struct type: `where T: not struct`
  * Definitely a resource type `where T: resource`
  * A resource or struct: ???
* What are the proper defaults for type parameters with regard to allowing resource types and
  structs? Is `aliasable` still the correct default? If it is used, then a type parameter doesn't
  have to worry about structs and resource types.
* Constraining to a capability set seems to only work if that set includes `id`. Is that correct?
* What about `shareable ind`?
* Does the `readonly` capability include `id`?
  * Answer: yes because `id List[X]` is still `out` in `X`
* Is `<constraint> T` (e.g. `aliasable T`) the correct syntax or should some kind of operator apply
  since it is really an upcast of the capability? Perhaps it should be `T as aliasable`?
* Since `readable` accepts `iso` and `own`, that means a method on a struct or resource type
  accepting `readable self` might have to accept ownership and also might have to call the
  destructor.
* For a `mut List[own S]`, the items must be isolated in order to resize the list. How is that
  handled?
* Can you assign overtop of a struct while there are references to it in order to change its value?
  What about cases of a closed struct?
