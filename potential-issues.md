# Potential Issues

There are things that have been identified as potential or unresolved issues in the language design.
This section documents them and when possible tries to give possible resolutions.

## Trait Functions on Structs

If a struct implements a trait that defines a trait function, how does the `self` parameter work
when it is invoked on that struct? The `self` parameter must be some kind of `ref` or `iref`. This
corresponds to the issue of how an override of the trait function should be declared in the struct.
It seems it ought to be `ref self`. However, a trait function could take `self` and pass it to
something expecting the type of the trait. This is a problem because that method is expecting a
reference type not a `ref` type.

Possible Resolutions

* Restrict Trait Functions: A trait function is restricted what it can do with the `self` reference
  to things that would be safe regardless of whether `Self` is a class or struct unless a constraint
  `where Self : struct` or `where Self : class` is applied. This implies that traits are truly
  distinct from classes. While they can be used as reference types, when they are applied to structs
  or used as generic constraints they are not working that way.
* Use `iref` and Allow Conversion: A trait function can only be overridden by an `iref self`
  function. That way the value exists on the heap. But then when the trait function passes `self` to
  another object an implicit conversion is occurring from `iref Self` to the trait type `T` which is
  a reference type. That would imply all references could be internal references. Note that one
  still has to have the notion of `iref` because there is no other way to express `iref S` where `S`
  is a struct type. It is too bad `iref` couldn't be removed because that would simplify the
  language. Allowing any reference to be internal has been avoided because that would make the
  garbage collector much more complex. But perhaps this could be handled by tagging the references
  that are internal? It would allow more class references to be inlined since it would then be
  possible to hand out references to them. A serious problem with this option is that it is not
  possible to call trait methods on structs on the heap.
* Require Overriding: All trait methods must be overridden by the struct. They are overridden with
  `ref self`.

## `iref` Conversion to Trait Reference

Can you convert an `iref S` to a trait type `T` that `S` implements? If so, that introduces
additional overhead/complexity to the garbage collector since any reference could now be an internal
reference.

## Generics Over Value Types Without Implicit Boxing

Without implicit boxing, a generic type that could be a value type cannot be passed as a trait
reference. This greatly limits what can be done in generic types that don't constrain their generic
parameters.

## Closed Value/Struct Tearing

When assigning a closed value or struct variable a new value, the new value may have a different
discriminator. The challenge is that the discriminator can change the layout of the data. Since
writing values and structs is not atomic, the garbage collector could read the value in a state
where the discriminator doesn't match the values in memory and could therefor follow a value that is
an invalid reference.

Possible (Partial) Resolutions:

* Use volatile writes and reads on the discriminator along with a special discriminator that tells
  the GC not to follow any references in the struct.
* Zero out the struct values before setting a different discriminator. (May still require volatile
  reads or writes.)
* Do not allow `ref` or `iref` to closed structs to cross threads. At least then it is only one
  thread and the GC thread that have to coordinate.
* Do not allow assignment into closed types that could possibly change the discriminator. (Not as
  crazy as it sounds given `let`.)
* **Recover isolation on the previous value before assigning into it to prevent broken references.**

## Cancellation

Passing cancellation tokens as contextual parameters still requires that everything calling a
cancellable operation pass along the cancellation token. This makes them viral similar to the issues
with function color.

## Exclusive Mutability

Should an exclusive mutability reference capability be included (e.g. `xmut`)? It may be needed for
properly working with iterator invalidation. However, it is complex and makes it difficult for
developers to decide when they ought to use it.

## Dictionary Keys and `independent(shareable)`

## Capability Set Including `own`

If `iso` maintains isolation on fields, then a capability set that includes `own` but not `iso` is
needed so that generics can allow `own` but not `iso`.
