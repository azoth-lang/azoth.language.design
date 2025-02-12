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
