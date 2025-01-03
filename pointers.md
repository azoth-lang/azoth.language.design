# Pointers

## Syntax

While C/C++ have well established pointer syntax and Azoth is in that linage, it uses pointers much
more rarely. Given that, it made sense to change the pointer syntax for purposes of clarity and
freeing up symbols. In terms of clarity, being able to combine the dereference operator in a postfix
position with the member access operator seemed useful. In terms of freeing up symbols, the unary
star operator and unary ampersand operators aren't actually that useful for anything else, but not
overloading their meaning reduces cognitive load.

A number of possible syntaxes were considered. They were:

| Syntax          | Address Of  | Dereference | Member   | Mutable Pointer | Immutable Pointer |
| --------------- | ----------- | ----------- | -------- | --------------- | ----------------- |
| C/C++           | `&x`        | `*x`        | `x->y`   | `*mut T`        | `*T`              |
| `@` Address†    | `@x`        | `^x`        | `x^.y`   | `@mut T`        | `@T`              |
| `@` At          | `&x`        | `@x`        | `(@x).y` | `&mut T`        | `&T`              |
| ptr             | `ptr.to(x)` | ?           | `x.y`    | `ptr<mut T>`    | `ptr<T>`          |

† Dereference matching type declaration. Thus type is pointer to `T` and matches expression pointer
to `x`. Or use "address of" instead of "pointer to". This matches an example from JAI.

The `@` meaning "at" syntax is confusing to programmers familiar with C++ reference syntax. Also, it
doesn't provide a clean member access unless the dot operator auto-dereferences which I've never
liked.

The `ptr` type seemed a good idea because it demotes pointers from a first class language feature.
However, it has some problems. The `ptr` type overloads the dot operator. This is consistent with
the behavior of the dot with variable references. There doesn't seem to be a good way to handle
dereference here though. Perhaps there is some operator that could be allowed to be overloaded. That
operator maybe should be allowed on variable reference types to get the underlying reference.

In the initial designs, pointers were allowed to be null, like default pointers in other languages.
However, this led to needing some other inconsistent syntax for non-null pointers. Furthermore, if
making reference types non-null by default makes sense, why wouldn't it make sense to make pointer
types non-null by default? That is why pointers are now non-null and must be made nullable using
optional types.

## Valid Pointer Types

At first it was assumed it would be reasonable to have pointers to reference types. Eventually it
was realized there was serious issues with this. As a result, only pointers to struct types are
allowed. This follows the precedent set by the C# language. On C#, pointer may only be made to
"unmanaged types". That is, primitive types and structs with unmanaged type. Problems that would be
caused by allowing pointers to reference types include:

* References are "fat" they contain both a pointer to the data and a pointer to the vtable. To
  support calling methods on pointers to reference types, some pointers would have to be fat while
  other's weren't this would lead to confusion and issues with interop.
* Pointer arithmetic doesn't make sense on pointers to reference types. Pointer arithmetic only
  makes sense when there is a contiguous sequence of values of the same exact type. They can't be
  different sizes or different subclasses that need different vtables.
* How to declare a pointer to a reference? If `@Foo` is a pointer to a `Foo`, what is a pointer to a
  reference to `Foo`? Something like `@ ref Foo` could work but seems awkward and confusing.
* Constructing pointer types from generics leads to confusion. Given a generic type `T`, if `T` is a
  struct then `@T` is a pointer to it, but if `T` is a class then `@T` is now a pointer to the data
  of the class, not a pointer to the reference. This is inconsistent and breaks the idea that
  references are effectively structs that refer to an object.

## Class Value Types

At one point it was thought that it would make sense to have a way of referring to the value type
implied by a reference type. That is essentially the value that is being referenced by a reference
type. These were called class value types since they were values types associated with classes. The
proposed syntax for the class value type of a reference type `T` was `^T`. The intended reading
being the dereferenced value of `T`. By making these types distinct, one could avoid the problem
described in the valid pointer types section of needing fat pointers. Instead, the class value type
would guarantee that the value was of exactly type `T` and not some subclass.

The problem with class value types is how to handle invoking methods on such a type. The methods of
some class `C` expect to take their self parameter as a reference to `C`. When invoking a method,
all that is available is a pointer to `^C` which cannot be converted to a reference. Attempting to
make the conversion opens up the possibility that some method could capture a reference to `self` in
another object and produce an invalid reference to something that is actually not a garbage
collected object.

## Void Pointers

At one point it was thought that the void pointer syntax should be `@Any`. That made some sense
where it was possible to have pointers to reference types or later when it was still possible to
have pointers to class value types. However, once those were removed it became confusing. While it
certainly reads well, it seems to imply that you could have a pointer to a class since the `Any`
type is the base type for all reference types. There then didn't seem to be any good option. It
doesn't make sense to introduce an `@any` type with unique syntax just for this. Besides, the `@any`
type wouldn't be a base type for value types. Given that, it made the most sense to fall back on the
traditional convention of using actual void pointers i.e. `@void`. This actually makes somewhat more
sense in Azoth since the `void` type is a true type that acts somewhat like a unit type.
