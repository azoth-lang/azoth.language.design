# Equality

Equality has turned out to be a much thornier problem than anticipated. It doesn't seem to play well
with type variance and has strangeness with inheritance too.

First a little background. There seem to be several schools of equality in languages out there. In
the Haskell and Rust school, equality is a trait/type class that types implement. So a type T is
always equatable in only one way to another type. If you want another way of comparing equality (for
example, if you want to compare strings ignoring case) then you use the new type pattern to make a
type with a different equality. The the C# and Java school, there are both interfaces for a type to
support equality (e.g. `IEquatable<T>`) and independent equivalence relations (e.g.
`IEqualityComparer<T>`). They also make the mistake of giving every object an `Equals(object?)` and
`GetHashCode()` method. Scala ran into problems with this and had to introduce "[Multiversal
Equality](https://docs.scala-lang.org/scala3/reference/contextual/multiversal-equality.html)" to
deal with some of the issues.

## Problems

### Contains Method

**TODO:** Example of how the contains method breaks variance and is difficult to implement. If it is
on a set, then there is an issue with the equivalence relation used by the set.

### Equality of Constant Lists

Consider an inherently immutable list. Such a list ought to be `out T` (e.g. `IFixedList<out T>`).
Indeed, doing so has been very helpful within the Azoth compiler. It also makes sense that if `T` is
equatable then it should be possible to compare two such lists of `T` for equality. Unfortunately,
C# does not support such conditional implementation of an interface. However, since it has
"universal equality" we can just assume values can be compared for equality and get some flavor of
equality.

So, we would like to implement `IEquatable<IFixedList<T>>`. The first problem we run into is that
this is not allowed because of variance. This makes sense since it effectively adds a method `bool
Equals(IFixedList<T>? other)`. Such a method would normally violate the variance since a list of
`Cat` could be upcast to a list of `Animal` and then have its equals method called with another list
of animals which would not be safe (for some arbitrary method taking a list). This is the same
problem as the `Contains` method.

While we can't implement `IEquatable<IFixedList<T>>`, we still have the `bool Equals(object?)`
method. Perhaps we can just properly implement that? Here we run into a second problem. The typical
way to implement such a method would be to check that the parameter has the same type as us and cast
it (e.g. either check `GetType() == obj.GetType()` or `obj is IFixedList<T>`). However, objects in
C# retain their types from construction even if they have been upcasted. So we can easily run into a
situation where we are trying to compare an `IFixedList<Cat>` to an `IFixedList<Animal>` and have
them report not equal because the second is not a list of `Cat` even though it may happen to contain
only `Cat`s. There is also an issue with making equality commutative in this case since the reverse
comparison may succeed.

It seems in C#, we are forced to check only that the other object is `IFixedList<object>` and then
compare the values using `bool Equals(object?)`. This is unfortunate since due to unboxing and type
checks, it is the least efficient way to compare values for equality.

### Implementing Equality with Inheritance

**TODO:** Good example of why type checks are necessary when implementing equality.

## Dictionary Key Equality

In C#, an equality comparer can be provided when constructing the dictionary that is used to
determine if the keys are equal. In some languages like Rust, that is determined statically at
compile time by the type. One argument for the type based approach is it eliminates the question of
how to compare two dictionaries that are using different equality comparers for their keys. A
similar argument applies for sets.

## Mutation vs Key Comparers

One would like to be able to have mutable keys for dictionaries that are compared by reference. For
sets, that makes even more sense. Yet, comparisons or orders can be used to compare values in ways
that aren't stable under mutation. That is, you can compare and sort values based on their current
state. Yet such a comparison or equivalence relation should not be used as the key comparer for a
dictionary key. How does one deal with that? I thought of 4 options:

1. `Dictionary` enforces that the key type can only be `const` or `id`. My example against that is
   some code I am working on for finite automatons.  The `State` nodes are mutable. For certain
   operations, I create dictionaries from one state to another. I could create these as
   `Hash_Dictionary[id State, State]()` and that would work, but feels strange. It feels more
   natural to say `Hash_Dictionary[State, State](Identity_Equivalence)`. It feels like it is a
   property of the way they are compared that the `id` is used. An advantage of this is that it
   might then be possible to have the dictionary automatically select the `Identity_Equivalence` for
   `id` keys.
2. `Dictionary` allows any `Key` but enforces that the `Key_Comparer[Key]` only takes either `const
   Key` or `id Key`. This is not currently expressible in my language because it requires more
   complex relationships between capabilities and generics that I don't think I would otherwise
   need. So I don't know that I want to add that complexity.
3. In addition to `Equivalence` etc. I could have traits `Stable_*` like `Stable_Equivalence` that
   have the same methods but the additional semantic meaning that their result cannot change over
   time even if the values they are called on are mutated. This adds a lot of new traits that devs
   will forget to use correctly since they don't differ in interface, only in semantics.
4. Don't enforce the requirement in code. Just document that when creating a `Dictionary` one should
   use a comparison that is stable. (This is the .NET solution except they don't even spell any of
   this complexity out.)

I've decided to try option 2 for now. This does require adding to the type system of the language.
For a generic type `T` you will now be able to combine it with a capability constraint. For
dictionaries and sets this will be `shareable T`. This means the type `T` "upcast" to the lowest
capability in the capability set (shareable allows only `const` or `id`). I'll see how that goes. I
suspect that this functionality will also cover something that the Pony language supports which is
what they call alias types (`T!` in Pony syntax). This functionality ought to make `aliasable T` the
equivalent in Azoth. Note that `shareable T` is distinct from a viewpoint type like `const |> T`.
Viewpoint types answer the question what would be the type of a member of type `T` accessed on a
`const` object. While `shareable T` is a way of constructing a type that is a supertype of  `T`.

## Meaning of Equality and Partial Equality

In trying to define the `Partially_Equatable` and `Equatable` traits along with the traits for
ordered, there were unexpected issues and challenges. Mostly around IEEE floating point types
because they by default define `==` to be partial since `NaN == NaN` is `false` (or `none` in
Azoth). That is simply making equality a partial equivalence relation. Worse than that is that they
define `-0 == +0`. That makes equality not truly have the substitution property since `1/-0 == -Inf`
while `1/+0 == +Inf`.

Thinking about it, there aren't that many types where strange equality things pop up. The test cases
I have are:

* IEEE floating point (`NaN`, `-0`)
* IEEE decimal (`NaN`, `-0`, "cohorts" of the same number expressed with different exponents)
* `bool?`: should have three-value logic for which "logical equality" `none == none` would return
  `none`. Yet, developers will need an easy way to check if a value is `none`.
* SQL Null: In a DB API one might need to represent the SQL Null behavior (e.g. `DBNull[T]`) where
  `NULL == NULL` returns false.
* Unreduced fractions: if a data type represented fractions but didn't always reduce them then `1/2`
  should equal `2/4` yet they are not perfectly substitutable since once can check the numerator and
  denominator to tell them apart.
* Java `BigDecimal`: treats `1.0` and `1.00` as not `equals()` but `compare`s them as equal. This is
  because the precision affects how operations are performed but not logically what number is being
  represented. (This case will probably be handled differently in Azoth by using `with` blocks.
  However, it is plausible that someone will want to have a type with the same behavior.)
* `int?` etc: in C# `none == none` is true but `none <= none` is false. This a general problem with
  how nullable types that can be compared should be handled. One would like to easily test them for
  the value `none` but that value is not ordered relative to the other values. If one made `bool?`
  work so that `none == none` returned `none` that would be inconsistent with the expected behavior
  for `int?`
* Unicode Strings: is equality over Unicode code point sequences or are two canonically equivalent
  strings equal?

Thinking through these cases and the complexities, it struck me that perhaps the problem was that
developers had been conflating equality with a general equivalence relation. That was certainly
consistent with how equality operations have been loosely defined in programming. The reflexivity
and substitutability properties of equality ought to mean that their are no incomparable values.
Also in math equality is the unique relation that is both symmetric and antisymmetric. Again meaning
it can't really be partial. (see [Is = (equality) a partial order
relation?](https://math.stackexchange.com/a/4953462/191636)).

Given that definition of equality, the operations on floating points only define an equivalence
relation and partial order, not equality. That also clarifies that according to equality, `none ==
none`. Thus IEEE floats and decimals should not have a partial equality or have `-0 == +0`. Rather
than should be a relation that can be chosen by a `with` block. That also fits with the challenges
of checking them for equality using a tolerance. However, it is still the case that one needs an
operation to check if they are the same value. Likewise with the rationals case, it still seems that
`1/2 == 2/4` ought to be true since equality should be defined over the abstract value. Both of
these cases indicate that there is some form of more strict equality that is sometimes needed.

Additional Points:

* Having a partial equivalence relation or partial order return `false` for incomparable elements is
  too error prone. Developers forget about the edge case and write code as if the relation is total.
  That is why it makes sense to return `none` for incomparable elements.
* Returning `none` as well as having comparison return an ordering is consistent with the alternate
  definition of a partial order given on Wikipedia where it is a comparison that always returns one
  of four results: `<`,`=`, `>`, or `|`. The last meaning incomparable. This definition also avoids
  the strict vs non-strict complexity by offering the choice within a single ordering.
* Conflating reference equality with other kinds of equality is confusing and error prone. It is too
  easy to think you are doing reference equality and not or vice versa.
* It has been an ongoing challenge to come up with multiple equal and not equal operators. When
  there was only a single equality the plan was that the operators would be `==` and `=/=`. This
  made sense since `!` does not generally have the meaning "not" in Azoth. The inability to come up
  with acceptable not equals operators that work with multiple equality has left little choice
  except to use `!=` as the not equal operator.

Based on all the above, here is the design I came up with:

**Design:**

There is no `Partially_Equatable`. There is only `Equatable`. There are three ways of comparing
equality:

* *Reference Equality*: compare two reference types, including variable (`ref`) and internal `iref`
  references, for equality of reference. Language defined and not overridable. (In some cases
  structs act like references. The most common cases are `string`, `Array`, and `Span`. In those
  case the question of how length plays into the situation means that reference comparison on them
  isn't a good idea. For simplicity and to avoid improper overloading, the decision has been made to
  not allow overloading for now.).
* *Strict Equality*: (Needs a better name.) compare two constant values for equality. This should
  account for full behavioral substitutability. It may still allow multiple bit representations to
  be equal as long as they are treated identically by all operations. This is defined only for types
  that have something like this. Not for all types that define equality. That avoids developers
  thinking they should use it all the time rather than using regular equality. Examples: defined on
  IEEE floats and decimals, defined for Unicode strings where it compares code point sequences,
  would be defined on rationals so that `1/2` not strictly equal to `2/4`.
* *(Abstract) Value Equality*: compare two constant values for abstract value equality. This allows
  for a slightly weaker form of equality test where `1/2 == 2/4` but it is still total so that `none
  == none`. This is the primary equality operator that should be used almost all the time. Examples:
  defined on Unicode strings to compare canonically equivalent strings as equal, would be defined on
  rationals so that `1/2 == 2/4`.

Additional Notes:

* Logical Equality on `bool?` is done with the `iff` operator.

Two possible sets of operators

| Operation          | All Operators | Some Methods    |
| ------------------ | ------------- | --------------- |
| Reference Equality | `@==` `@!=`   | `@==` `@=/=`    |
| Strict Equality    | `===` `!==`   | `strict_equals` |
| Value Equality     | `==` `!=`     | `==` `=/=`      |

The all operators set has the downside that it uses prefix ! to mean "not" which is inconsistent
with the fact that suffix ! always means "may abort" and that the not operator is not x. The some
methods set has the downside that if it is a method strict equality over optional types doesn't work
well (stand-alone function would be fine) and it is more verbose. However, I like that it makes the
language simpler and really pushes toward one kind of equality that one normally uses.
