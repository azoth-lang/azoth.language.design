# Optional Types

While optional types may have special compiler support, they should be isomorphic to something that
could be written in Azoth. This is for two reasons. First, so that they do not have to be a special
case in the type system. If they can be expressed in Azoth then their types fit into the type
system. Second, so that library designers can create types with equivalent behavior.

There are a number of options for how they might work. All use a `closed value` type to represent
the cases of none and some. They vary in how the capabilities are handled.

Contents:

* [Variable Capability with `iso` Parameter Capability](#variable-capability-with-iso-parameter-capability)
* [Variable Capability with `mut` Parameter Capability](#variable-capability-with-mut-parameter-capability)
* [Variable Capability with Regular Parameter](#variable-capability-with-regular-parameter)
* [Const with Independent Parameter](#const-with-independent-parameter)
* [Const with Regular Parameter](#const-with-regular-parameter)
* [Special Pseudo Reference](#special-pseudo-reference)
* [Variable Capability with `mut` Parameter Capability and `self.T` Initializer](#variable-capability-with-mut-parameter-capability-and-selft-initializer)
* [Brainstorming](#brainstorming)
  * [Side Note](#side-note)
* [Decisions](#decisions)

## Variable Capability with `iso` Parameter Capability

In this approach, the `Optional` type has a flexible capability, but the type parameter is fixed to
a the `iso` capability.

```azoth
public closed drop? value Optional[iso T]
{
    case None;
    case Some(value: T);
}
```

Good:

* The capability would vary properly as it was copied since all the capability rules would be
  applied as if it were a reference type.

Problems:

* An optional struct would not work since `mut Struct?` would be encoded as `mut Optional[iso
  Struct]`. This would store the `Struct` value inline instead of storing a reference to a `Struct`.
* Constructing would not work properly since creating a `const Foo?` would still expect an `iso Foo`
  parameter to the initializer of `Some`.
* The type `own Struct?` would encode as `own Optional[iso Struct]` but that is incorrect because it
  applies `own` to a non-drop value type (assuming `Struct` is not a drop type).
* Currently, `iso` and `own` structs are not permitted as fields of a value. (That may have to be
  changed.)

## Variable Capability with `mut` Parameter Capability

In this approach, the `Optional` type has a flexible capability, but the type parameter is fixed to
a the `mut` capability.

```azoth
public closed value Optional[mut T]
{
    case None;
    case Some(value: T);
}
```

Good:

* The capability would vary properly as it was copied since all the capability rules would be
  applied as if it were a reference type.

Problems:

* An optional struct would not work since `iso Struct?` would be encoded as `iso Optional[mut
  Struct]`. This would store the `Struct` value as a reference instead of storing it inline.
* Constructing would not work properly since creating a `const Foo?` would still expect a `mut Foo`
  parameter to the initializer of `Some`.
* The type `own Struct?` would encode as `own Optional[mut Struct]` but that is incorrect because it
  applies `own` to a non-drop value type (assuming `Struct` is not a drop type).
* Currently, `iso` and `own` structs are not permitted as fields of a value. (That may have to be
  changed.)

## Variable Capability with Regular Parameter

In this approach, the `Optional` type has a flexible capability, and the type parameter does not
constrain the capability.

```azoth
public closed value Optional[any T]
{
    case None;
    case Some(value: T);
}
```

Problems:

* This allows too much flexibility. The capability parameter could be anything and may not match the capability of the optional.

## Const with Independent Parameter

In this approach, the `Optional` type is `const`, and the type parameter is independent.

```azoth
public closed const value Optional[independent T]
{
    case None;
    case Some(value: T);
}
```

Good:

* The capability applies to the type parameter and so rules like whether `own` is allowed and
  whether `id` has sharing tracked will probably apply correctly.

Problems:

* Optional types can only be used with generic parameters that accept `const`. For example, `class
 Example[mut T]` would not permit `Example[mut Foo?]` because the parameter is actually `const`.
* May have issues with nesting. For example, in the type `Factory[iso Foo?]` is the capability of
  `Foo` independent? Perhaps this is not a problem. Perhaps it is a general rule that nested
  independent parameters are no longer independent?
* Aliasing produces the wrong capabilities. Consider the type `Array[independent T]` which ought to
  have the same behavior. Given a `mut Array[own Struct]` then an alias to it ought to still be `mut
  Array[own Struct]`, not `mut Array[mut Struct]`. However, this approach would require the latter
  behavior. (This might not be right)

## Const with Regular Parameter

In this approach, the `Optional` type is `const`, and the type parameter does not constrain the
capability.

```azoth
public closed const value Optional[any T]
{
    case None;
    case Some(value: T);
}
```

Good:

* The capability applies to the type parameter and so rules like whether `own` is allowed and
  whether `id` has sharing tracked will probably apply correctly.

Problems:

* This blocks flow typing. For example given `let x: iso Foo?` it is never possible to freeze `x`.

## Special Pseudo Reference

In this approach, the optional type uses a special new syntax that is designed for the proper behavior of pseudo reference types. This syntax expresses the idea that the capability of the generic parameter is tied to the capability of the optional type.

```azoth
public closed value Optional[self T]
{
    case None;
    case Some(value: T);
}
```

Problems:

* Adds an additional complexity to capabilities
* Would need to adjust the behavior and allowed capabilities based on the type parameter it was
  applied to. For example an optional struct allows `own` and tracks sharing of `id` references.
* What are even the rules for this?

## Variable Capability with `mut` Parameter Capability and `self.T` Initializer

Set aside structs for a moment. If one had a class with a `mut T` parameter/field, then it would
have the correct behavior, but one would need a way to construct it with `T` values that were,
`read`, `const`, and `id`. Note that `iso`, and `mut` would already work fine. This implies that the
issue is with the initializer.

```azoth
public closed value Optional[mut T]
{
    case None;
    case Some(value: self.T);
}
```

## Brainstorming

classes and structs have a single "instance"

optional structs would need to follow the rules that `id` sharing is tracked. A regular value
wouldn't do that because `id Value[T]` doesn't track sharing. That would be fine for a class since
internally it has a non-id reference

Overload `Option[T]` on whether the parameter is a struct?  Use Value for non-struct and Struct for
struct?

* Disallow optional structs
* Disallow optional iso/own structs

What if internal references could only point into the heap, not the stack? Then either there
wouldn't be hybrid types or they could behave like any other type for capabilities. Then `own` would
exist only for drop types.

Alternatively, structs don't exist and `Var[T]` and `Ref[T]` use `unsafe` (but how does the runtime
understand interior references?)

* Disallow `id` on structs?
* Disallow `id` on structs that aren't known to be on the heap?

If you did either of those, how would you be able to use struct references as keys in a dictionary?

* Structs on the stack can only be lent? (Doesn't work because you are allowed to take an `id` of a
`lent` parameter) Structs on the stack can't be referenced doesn't work because references are how
you call methods on it. Bring back C# type `ref` and `in` to work with structs

**Good Idea:** Allow `id` to structs on the stack but disallow any access through them. That way,
they can point to invalid addresses without problem? What if that memory is later reused for a heap
object though? Just make it so `id H` never keeps an object alive! They still have to be updated if
an object moves but it is ok if they dangle. This also allows for leaving an `id` behind when moving
structs.

Since you can't freeze a `drop` type, maybe `own` isn't a supertype of `iso` but instead replaces
`iso`? Likewise, structs on the stack would only support `own` since they can't be `const`. Structs
on the heap would still use `iso` unless they were drop types. The issue with this is that they
still don't behave like other types for the purposes of generics. Maybe structs always use `own` but
are allowed outside of `drop` types? But how is that rule handled for generics? Or maybe structs are
just `drop` types and we go back to drop structs vs copy structs?

`own` can't just replace `iso` because passing a drop type to a function requires passing `iso` so
that it can safely recover isolation when it goes out of scope.

Re-reviewing the MS paper, they did not allow `iso` on generic arguments. Likewise, they did not
allow fields to lose isolation. Thus `iso` acted much more like a genuine affine type. One route
would be to treat it more like an affine type in all cases. There would however need to be some
special aspect for parameters to allow a parameter to explicitly receive isolation. Likewise `temp
iso` would have to be an affine type that couldn't be used as a generic parameter.

If `iso` was truly disallowed for generic arguments, would that prevent `Option[T]` from supporting
`iso`?

There is no way around it. To put a struct inside a value type while allowing `own` requires some
kind of special "field with the same capability as the value."

`iso` is sometimes needed as a generic type because you need it to use as a parameter or return type.

The older docs treated `iso` fields as having to be moved out of. The newer docs say that isolation
is just lost. I think the move out of is more proper. Now that I have the move assign idea `x =
move y = z;` that gives a syntax for swapping out of fields.

Normally, all fields of a value must be `let` and `aliasable` because of copying. Thus a field could
not be `iso` or `own`. `iso` could never be supported because it acts as an affine type in a field.
If the value was a drop type then it could support `own` drop type fields but it would need a way to
vary their capability with its own capability. Likewise for structs it would need to vary the
capability with its own and copying would need to apply the hybrid type rules to make a reference.

I feel like the affine rules mean that I ought to require assignments into an `iso` variable to
always be `iso` values even if the previous value has been aliased. But that doesn't work for
structs since the reference to the old value will be updated to reference the new value. Thus I
guess that later assignments should not have to be isolated? Well the value has to be moved, so the
value will be isolated, but the variable won't be isolated afterward, it will just be `own`. Yet,
the first assigning into an uninitialized `var` does need to be an `iso` variable. I think the value
does need to be `iso`. If you don't want that, don't use `iso`. You can still use `own`. The flow
typing rules exist mostly for parameters and generics. Speaking of generics, this is really annoying
for generics because one must declare locals `aliasable T` to avoid needed new value to be `iso` all
the time. I think that is the reason to not require `iso` on later assignments.

How do initializers with `self T` work?

Consider `move` on a generic parameter type that may or may not be `iso` or `own`. How does the
compiler handle that? It doesn't know if it needs to recover `iso` or indeed whether it even can.
Perhaps `move` is just moving the value and leaving `id` behind. The recover is implicit. Thus you
can use `move` on the generic parameter type and it will work whether you are moving `iso`, `own`,
or some other capability like `mut`. That does seem quite right. To move a struct, isolation must be
recovered so there can be no other local references. Rather it is like `move` must become a no-op
for generic types that don't need it.

For nested nullables, `?.` should go through multiple optional layers. But there needs to be a form
of `??` that goes through only one layer. This is for situations like poping a value from a stack of
optional values and handling empty stack differently than `none` on the stack. This could either be
that one must use pattern matching, or there could be additional operators. It could be that `??`
only evaluates one layer of optional while `???` or `??*` evaluates multiple layers.

### Side Note

Maybe `if` with `else` should only allow `bool` while `if` without `else` allows `bool?`? But maybe
it still doesn't allow `bool??` because should that still follow ternary logic?

## Decisions

1. Hybrid allows `id`. One can access nothing on a hybrid type with the capability `id` except for
   `identity_hash()` and `@==`. This restriction makes it safe to keep an `id` reference to
   something on the stack that has gone out of scope. The GC supports this because they are special
   references which do not keep the referenced object alive, however, they will be updated when
   objects are relocated.
2. Generics default `aliasable`. This is consistent with the Midori paper not allowing `iso` for generics. It also avoids any issues with structs or drop types since `own` is not aliasable. Thus a default generic parameter can be a struct reference but not an inline struct. It can be a drop type but not an owned drop type.
3. Conditional drop is `drop[T1, T2...]`, those can be stored in fields
4. Generics allow structs, values, and drop by default. This is usually hidden by the default `aliasable` constraint. If the `any` constraint is used then structs, values, and drop types can be used including `iso` and `own`. The compiler enforces that the implementations follow the correct safety limits to make that possible.
5. If desired, one can use `where T: not drop` or `not struct` or `not value` to exclude those from a generic type. This can reduce the restrictions on how a generic `any` type can be used.
6. `if` works as above
7. `iso` on fields is maintained, you must move out of them to access the value.
8. Move assign is supported
9. Moving out of a field without move assign leaves the default value
10. There should be a default value operator
11. setters return values to support move assign
12. Values support `self T` parameters which can be stored as fields (they must be `drop[T]` unless
    `T: not drop`)
13. Reassigning into an `iso` variable does not require an `iso` value
14. Need to spell out how `move` works on generic parameter types. It recovers isolation locally,
    but doesn't worry about whether the caller might have reference. If the type is `iso` then there
    will be no caller references. If the type isn't `iso` then the caller references will be ok.
