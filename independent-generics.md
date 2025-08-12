# Independent Generics

Up to this point, the design of independent generic parameters has ignored issues of `iso` and of
move types. This is because the default constraint is `aliasable` which does not include `iso` or
`temp iso`. However, with the introduction of hybrid types it seemed important to consider how one
would work with an array of structs (not struct references). That is, after all, exactly what most
games and high-performance code needs to do. Considering this has really brought to the forefront
that I don't know how it should work and perhaps don't have a proper mental model of independent
parameters.

## The Problem

The problem was first noticed in the interface of the `Raw_Hybrid_Array[P, T]` class. However, to
avoid the complexity of thinking about the right way of handling the low-level details and unsafety
and to avoid some complexity about initialization we'll consider the design of a `Hybrid_List[P, T]`
class.

Imagine we will create an array of enemies for a game following an entity-component system (ECS)
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
        // To iterator on the list, the list itself must be temp const, but the elements can remain
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
However, it would no longer be necessary to recover isolation on the elements to `remove_enemy`.
Also, it would be fine to add `mut Enemy` into the list as long as isolation can be recovered for
them later when they go out of scope. For example, one could add a `mut Enemy` to the list while
retaining a reference to the enemy in order to do some work. For example, to draw it to the screen
and then release that reference allowing isolation to be recovered again when the list goes out of
scope. But assuming the list still owned the enemies, it would still be necessary to recover
isolation for them at the end and destruct them

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
    //
    published fn at(self, index: size) -> T { ... }

    // How does one support recovery if isolation for T or freezing T? Does that require methods?
    // If the list contains at least one item then `freeze list.at(0)` ought to freeze all items.
    // (Assuming freezing properly flow types all `read` references to now be `const`.) But that
    // requires the list to contain an element. It should definitely be possible to freeze the items
    // of a currently empty list so that all items added in the future must be `const`. Also, there
    // doesn't seem to be any equivalent for recovery of isolation. Do we need to add syntax like
    // `freeze list.T` and `recover list.T`?

    // Currently, it would be illegal to declare a destructor, but it seems declaring one is
    // necessary.
    published delete(iso self) { ... }
}
```

## Basics of Generics

Use `Channel[T]` `Source[T]` and `Sink[T]` as examples

* Generic arguments including capabilities are reified
* If the capability "changes" this is because of covariance and contravariance
* Can't overload on capabilities
* Capability of `self` is fully outside of the instance and not in any way reified.
* Currently available mechanisms
  * Constraining `self` and self viewpoint (i.e. `self |> T`). But typically self viewpoint has had
    not effect on independent parameters.
  * Constraining a generic parameter to be cast up to a capability set.
  * Specific capability viewpoint (e.g. `read |> T`)
* `Self` has no capability. One has to combine (e.g. `self |> iso Self`)

## Independent Parameters as Containers

* Independent parameters are so restricted that they can really only be used as containers (confirm
  even for the case where a trait is implemented).
* A container can have 0 to many, but the possible cases may matter (list them)
* If there are many items in a container, what happens if the items become shared with each other?
* The basic operations one can perform are:
  * add item
  * take item
  * discard item
  * replace item (replaced item either taken or discarded)
  * alias an item
  * recover isolation of the items
  * freeze the items (for non-move types which can be frozen)
* It seems like the issue with whether the container "own" the generic type
* Maybe independence replaces the capability restriction? But leaves a reification of whether the
  item is owned. This is consitent with the fact that the `self` capability is not reified and
  cannot limited (except `const` types).
* Maybe it is like the basic capability on `self`
* Imagine specifying it like the capability of self (e.g. `mut self[aliasable]`)
