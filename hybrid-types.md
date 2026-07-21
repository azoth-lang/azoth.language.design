# Hybrid Types

In the previous version of Azoth, there were classes which were reference types allocated on the
heap. There were also structs which were allocated on the stack or inline in objects. Structs could
be either `copy struct`s or `move struct`s. A copy struct was passed by value (i.e. copied). A move
struct applied the rules of a `move` type which meant it had to be moved and would leave behind a
value with an `id` capability. A move struct would then not be very useful without the ability to
refer to it without `ref` and `iref` types.

Variable references were formed with `ref`, `iref` and `var`. A `ref` could refer either to a
binding on the stack or to a field on the heap. The compiler would enforce rules that ensured
variable references couldn't outlive the binding they referenced. An interior reference (i.e.
`iref`) referenced a field binding. No rules were necessary to restrict them since they would keep
the object they referenced into alive. Interior references were critical to the implementation of
array slices. An array slice was internally represented as an `iref` to the first element of the
slice and a length. Indexing would be performed by an unsafe operation of converting the `iref` to a
pointer, indexing the pointer and converting the result back to an `iref`. Both `ref` and `iref`
could be combined with `var` (i.e. `ref var` or `iref var`) to reference a `var` binding and allow
assignment into a var. These types could of course be further combined so that one might have a `ref
var iref int` for example. Additionally, they interacted with reference capabilities. Accessing a
`ref var` through a read only object would both change it to a `ref` and make the referenced object
read only.

Additionally, `ref` required `ref struct`s which permitted fields to have `ref` types but could then
only be stored on the stack. The design for `ref` and `ref struct` mirrored that found in C# except
that they were true types rather than modifiers on parameters, variables, etc.

This system was complex. It was confusing and complex how reference capabilities interacted with
`ref` and `iref` types. Additionally, the rules for `ref` were distinct from but similar to the
limitations provided by `lent`. Finally, since `ref` and `iref` formed true types, they could be
used as generic arguments. The details of how that should work had not been fully worked out.
However, if a type declared it supported that, it would have imposed complex and confusing
limitation on the use of the generic parameter.

In trying to find a way to simplify this complexity, I somehow thought of the idea of "hybrid
types". I think I was thinking about how regular references are simplified in languages like C# by
having reference types which by the way they are declared are passed by reference. This is an
example of the tradeoff between the simplicity but limitedness of declaration site vs the power but
complexity of use site in language design. I wondered if you could combine the allocation on the
stack/inline with the pass by reference semantics of classes to eliminate the need for `ref` and
`iref` at use sites. Upon reflection I realized that Azoth already provided the mechanisms to make
that possible. Types can already be marked `move` to require that a unique owner be tracked. This
owner (i.e. `iso`) could then be the place where the type was allocated. All other reference
capabilities would then imply that the value was a reference to the original. Since `move` types
automatically recover isolation before going out of scope, that would provide a mechanism to enforce
that all references are restricted in a way that they can cease to exist before the value goes out
of scope. This can be achieved by the regular reference capability rules and `lent`. That unifies
the mechanism of `lent` with the system that controls references to items on the stack. This idea
also seems to be a natural extension of the idea of declaring heap allocated reference types and
stack allocated value types. It is just that other languages lack the mechanisms to make it safe.

The more I've thought about this idea the more I like it. It greatly simplifies the language by
eliminating `ref` and `iref`. It seems to be an elegant extension of reference capabilities. It
avoids complexity with generics. Indeed, I think that the mechanism for handling `move` classes as
generic arguments will apply to these hybrid types since the rules are essentially the same and
`iso` determines where the memory is allocation. I think `move` types should already not allow
freezing since calling a destructor after freezing doesn't make sense. So that eliminates one
potential issue.

## `id` for Hybrid Types

There is one issue with hybrid types and that is the `id` capability. For reference types, one can
always take an `id` to them and the sharing of `id` references is not tracked. One can recover
isolation even though there are `id` references. This is a problem because one could hold an `id`
reference to a hybrid type and then have a reference into the stack after that value was
deallocated. There seem to be three options:

* Disallow `id` reference capabilities on hybrid types.
* Enforce sharing rules on `id` types to some extent on hybrid types (not always since if the type
  is on the heap then the `id` reference would be safe and keep the containing object alive.)
* Remove `id` reference capabilities from the language.

I am not sure which makes sense. Removing `id` all together is tempting since it further simplifies
the language, but I am not sure if it will be needed and its absence will be frustrating. `id` is
also important for reference types because when moving a value what is left behind is an `id`. That
brings up the issue of what is left behind when moving a hybrid type if `id` is not allowed on them.
One thought was that `id` could be allowed but when moving a hybrid type, the `id` would change.
That is, old `id` references would continue to point to the previous value location and would not be
equal to `id` references of the new value location. It seems that would be necessary if `id`
references to hybrid types are allowed at all. Another option would be to enforce Rust style rules
that a variable of a hybrid type simply can't be used after moving the value out. If `id` was
removed altogether, that could also be applied when moving reference types. Currently, the only
place `id` is used in the standard library is in `Identity_Equivalence` and it could be easily
removed there if `id` was removed from the language.

From the Pony documentation it seems `id` might be most useful when communicating between threads.
However, their `tag` allows calling methods on an actor which may not be allowed in Azoth.

## Terminology and Keywords

It has been a challenge to figure out what to call these hybrid types and what keywords to use to
declare them. At first I referred to them as simply `copy struct`s and `move struct`s. But that
seems too confusing since they have very different semantics. I think calling the three kinds of
types reference types, value types and hybrid types works well. Declaring them seems like it needs
three different keywords. Reference types are declared with `class` (consistent with C# and Java).
However hybrid types are declared, they should have the `move` keyword since they share those
semantics and can implement `move` traits. Since hybrid types are a new concept, there is no
existing keyword or term for them. There are however terms for value types. Previously, a `value`
declaration was going to serve to declare a constant value for a `closed struct` similar to how an
`object` declaration declares a singleton object. However, the term `value` is the best I could come
up with for declaring value types. Further, it seems like a separate declaration akin to `object`
isn't needed. The reason it is needed for reference types is to indicate that an object is
allocated. For value type and hybrid types, no memory needs to be allocated. All that is being
indicated is an optimization to not copy the instance data around. That optimization can simply be
performed when it is appropriate. That leaves only what to use for hybrid type. Here `struct` seems
most appropriate because they are still allocated on the stack/inline and will still be the types
that can be referred to with pointers. Thus the three kinds of types and their declarations are:

| Type      | Declaration   |
| --------- | ------------- |
| Reference | `class`       |
| Value     | `value`       |
| Hybrid    | `move struct` |

Note that `value` declarations do not allow `var` fields (only `let` fields). This ensure they are
safe to copy regardless of what reference capability they have.

### Terminology Update 1

There are no longer `move` types. In their place are `drop` types. It was realized not all hybrid
types are drop types. Instead, hybrid types have the special `own` capability regardless of whether
they are drop types.

It is still unclear what the proper terminology and keywords to use are. Currently `class`,
`struct`, and `value` are used. However, that leads to an awkward problem of how to name the
equivalent of `object` declarations for value types. Currently they are `unit value`. The `unit
value` keywords are confusing because they sound like they should be a value used with unit of
measures. Indeed, if something like F# style generic arguments are needed to support units of
measure, there could literally be unit values.

Another option would be to use `record` for value types. Then `value` could be used for `unit
value`. The problem with that is that "record types" currently refers to the shorthand syntax for
types whose properties are declared inline with the class/struct/value declaration. So a `record`
keyword could be confusing giving the term "record" two different meanings. Alternatively, `value
type` could be used for value types and `value` for `unit value`. What is bad about that is the
declaration isn't just of the type, but is a "template" for creating the instances.

Given that hybrid types are the new concept, they should really get the new keyword. That suggests
`class`, `hybrid struct`, and `struct`. Leaving `value` for unit values. That also preserves
`struct` for a copy-by-value type like it is in other languages.

### Terminology Update 2

The use of `value` for value types and `unit value` for the equivalent of `object` has been an
ongoing annoyance. In the C++/C# linage, a `struct` should be a pass-by-value value type. The
keywords are just confusing. The new hybrid type ought to be the thing with a unique name. So new
names have been decided.

Another reason `record` would be a bad keyword to use is that they are generally used for types that
are more immutable or more raw data. That is not what a hybrid type is.

I searched for a new keyword to use for hybrid types. There doesn't seem to be one. The best I could
find was `dimorph`. But even that doesn't make sense. When we think of a `struct` or `class`
declaration, we are declaring the data structure and only implicitly that it is a reference type or
value types. A keyword like `dimorph` would be explicitly talking about both the data structure and
the way it was referenced. That seems inconsistent.

This leads back to a modifier on the `struct` keyword. You really are declaring a value type. It is
just that the value type is affine. Indeed, in Rust, this is called a `struct`. (Rust being the
major language with affine types. Though Swift now had `~Copyable` structs.) The best possibilities
for modifiers were: `identity struct`, `move struct`, and `affine struct`. The `identity struct`
keyword comes from the idea that this is a value type with an identity. That has the advantage of
naming what it is rather than what you can do with it. It is consistent with how Java is introducing
value types as classes without identity. But it isn't immediately obvious while being familiar
enough that people might expect they know what it means. It doesn't signify how different these are.
The `move struct` name is good in that it reuses an existing keyword in a way that makes sense. It
is a struct that must be moved. Note that this no longer conflicts with "move types" as those have
been replaced with `drop` types. But again, it isn't immediately obvious while being familiar enough
that people might expect they know what it means. It doesn't signify how different these are. That
is how we come to `affine struct`. This name is precise in the sense that this is the proper term
for the type. It is different enough that most developers will be struck that this is something
unique and different they will need to learn and understand. One might think `linear struct` would
be better since these are colloquy referred to as "linear types" even though those are technically
types that must be used exactly once. But that could easily be confusing since it sounds like the
struct itself is somehow linear (e.g. maybe it has a linear layout).

Thus the new terminology will be:

| Type      | Declaration   | Singleton |
| --------- | ------------- | --------- |
| Reference | `class`       | `object`  |
| Value     | `struct`      | `value`   |
| Hybrid    | `move struct` | N/A       |

## Variable References

It might be thought that this new design has lost the important functionality of having `ref var`
and `iref var` allow assignment into a variable. Interestingly, something very similar can be
achieved with a `move struct Var[T]`. This struct contains a `var T` field. To use it, declare the
variable to be mutated as `let x: Var[T]` instead of `var x: T`. Then one can pass around `Var[T]`
references to that struct wherever it is allocated. This does impose some restrictions on the
caller. However, given that this won't be used as much as it is used in C# and will be primarily for
high-performance coding and special cases like array slices, it seems more than adequate.
