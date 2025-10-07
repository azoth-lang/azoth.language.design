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
  independent parameters are not longer independent?
* Aliasing produces the wrong capabilities. Consider the type `Array[independent T]` which ought to
  have the same behavior. Given a `mut Array[own Struct]` then an alias to it ought to still be `mut
  Array[own Struct]`, not `mut Array[mut Struct]`. However, this approach would require the latter
  behavior.

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
