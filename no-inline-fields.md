# No Inline Fields or Private Inheritance

During the writing of the standard library there were multiple times when it seemed like it would be
valuable to have a mechanism like field inlining or private inheritance to avoid extra layers of
references. One place it seems like it ought to be used is in wrapping `Unsafe_Hybrid_Array` to
create `Hybrid_Array`. For regular arrays, the length and reference to `Var[T]` make it more
reasonable that they would be a value type. Additionally, it makes sense for arrays to not have
identity since two arrays could refer to the same start element but have different lengths. However,
for `Hybrid_Array` those don't apply. One would wish that `Hyrbid_Array` could be a reference type
but not introduce an extra layer of references beyond what already exists for `Unsafe_Hybrid_Array`.

## Inline Fields

One option that was considered was allowing the `#inline` attribute to be applied to a field. This
avoids introducing new syntax. However, it is a little odd in that it creates a complex condition
that can cause a compile error. But there are other attributes like `#obsolete` or
`#attribute_usage(...)` that have complex interactions with the compiler. There were two ways that
inlining a field my be accomplished.

The first option would be to take the fields of the type being referenced and inline them into the
containing type. This poses several issues. First, the actual field type must not vary from instance
to instance nor over time. Perhaps that could be enforced by requiring that it be a `let` binding of
a sealed type. More critically, when taking a reference to the field, one is actually creating
interior references that appear to be normal references. This would be bad for garbage collection
performance. It might be thought this could be avoided by restricting the use of references to the
field. That could be done some. For example, sharing could be used to prevent a permanent reference
to the field from being created. But one cannot avoid the possibility that calling a method on the
field will cause objects to be created which reference that field value. Given these issues, it
seems like another approach is needed.

The second option is therefore to only allow inlining when the class contains a single `let` bound
reference type field. In that circumstance, the field reference can replace the containing type
reference. The containing type then becomes a different vtable operating  on the object layout of
the field type. Given that Azoth will likely implement method lookup with fat pointers instead of a
pointer to the vtable in the header, that seems quiet doable. This would still require that the
field be of a single concrete type. If that were not the case, then the object layout would not have
a fixed size and other classes would not be able to inherit from it. Still this may be the most
viable option.

## Private Inheritance

The second approach to inlining is very close to C++'s private inheritance. Indeed it imposes odd
limitations like that the containing type must be a reference type that directly inherits from `Any`
and that the field must be of a non-varying concrete type. These limitations are naturally expressed
by private inheritance. With private inheritance, the base class is the class you are "inlining".
There can be no other base class and by naming the base class you are naming a single concrete class
to inline. In many ways, private inheritance is the natural form of what we are trying to express.
It also has the merit of having a clear implementation rather than the magic of a `#inline`
attribute.

The downside of private inheritance is the language complexity. One must add a syntax for private
inheritance (likely `class Example : private Base`), but that is only the beginning. One must then
define all the lookup rules of methods. When accessed from the outside the base class is not
visible. However, when accessed via `base.` the base class does exist. All base methods are
essentially private. Are the base methods actually inherited to the current class as `private`? If
so, how does that interact with implementing a trait that defines a method with the same name? It
seems very complex for a very obscure language feature.

There is an additional issue with private inheritance. The `Unsafe_Hybrid_Array` type should not
support inheriting from it normally. It would seem very odd that one can privately inherit from it.
But if one was allowed to, it would require that the derived class be declared `sealed`. This is
just a very odd situation for one of the cases that one most wants this feature for.

## Decision

Given that the `#inline` attribute has many awkward and random limitations and issues it seems not
worth implementing it now. Likewise private inheritance would add more complexity to the language
than seems worth it for the feature. Even if it were added, there would be issues for the important
use case of `Unsafe_Hybrid_Array`. For now, the decision has been made to simply not support this
feature. `Hybrid_Array` will be implemented as a value type. This will allow for safe expansion
later since value types are more restricted that other types. If private inheritance or field
inlining is added, then it will be possible to change `Hybrid_Array` to a class and recompile.
