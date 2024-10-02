# Type Variables

This section covers both generic types and associated types, collectively referred to as type variables.

## Use of Associated Types

For a long time, I was confused about when one should use an associated type vs a generic type
parameter. The rule I had learned that seemed to make sense was that generic type parameters are for
inputs and associated types are for outputs[^output1][^output2]. The other rule given was that you
use associated types when it only makes sense to implement the trait once [^once1][^once2][^once2].
However, these never seemed to give clear guidance. Both of the two criteria seem to be too
subjective and ill defined. In particular the Rust [Iterator
trait](https://doc.rust-lang.org/std/iter/trait.Iterator.html) is an example where many sources
claim that it makes sense to use an associated type. However, coming from a C# background that never
made sense. First, there are situations where it makes sense to expose iteration over multiple
types. Consider `MultiMapHashSet` that could reasonably implements both
`IEnumerable<KeyValuePair<TKey, HashSet<TValue>>>` and `IEnumerable<(TKey, TValue)>`. Also, in .NET
it is very common for methods to take or return enumerables where one wants to specify the type
enumerated over. I now think I understand it.

An associated type is a `static` (C# sense) type property. It should be used when the concrete type
fully determines the type in question. The associated type is not specific to any one instance.
Generic type parameters are essentially parameters to the constructor and specific to each instance.

For example, in a red-green tree, the type of the red node that corresponds to any green node is a
property of the node type. This also explains why associated types can be accessed as properties of
type they are on (e.g. *`ConcreteGreenNode`*`.RedNode`). It is hard to find good examples online
because so many sources are confused about what they mean. For example, the [Rust By Example:
Associated types](https://doc.rust-lang.org/rust-by-example/generics/assoc_items/types.html) gives a
graph with nodes and edges but it is confusing because it thereby conflates the node type with the
values stored at nodes in the graph. Either, the type is really a value type in which case it should
continue to be a generic parameter. Or the node type is a true node type in which case it should
probably be parameterized on the value type. The other good example I am aware of was for units of
measure. But I cannot remember the exact scenario. It was something like making an interface for
fractional quantities that had an associated type for the result to rounding to a countable (e.g.
`FractionalEvents` always rounds to `Events`). I think something similar happened when trying to
support euclidean norm in a generic way. There was no general way to express that `Length` *
`Length` gave `Area` and the square root of area was `Length`.

Returning to the `Iterator` example in Rust. The reason it is wrong is because the type being
iterated over isn't a property of the class of iterator. It is a property of the particular
instance.

I believe part of the confusion is that Rust and Swift allow something that they shouldn't. They
allow a generic type parameter to be assigned to an associated type of a trait. Consider the Rust
[`Map<I, F>` struct](https://doc.rust-lang.org/std/iter/struct.Map.html) which is returned by the
`map` method on iterators. It is generic on the value it iterators over and yields. But it
implements `Iterator` by setting the associated type to ta type determined by the generic parameter
`F`. This is a crossing over from an "instance" type to a "static" type.

```Rust
impl<B, I: Iterator, F> Iterator for Map<I, F>
where F: FnMut(I::Item) -> B
{
    type Item = B;
    // ...
}
```

Swift does a similar thing since they do not allow true generic protocol types. But they have
confused the issue with the introduction of their shorthand syntax for specifying associated types
which allows `IteratorProtocol<Element>`.

[^output1]: https://www.reddit.com/r/rust/comments/waxk1l/comment/ii3yulk/
[^output2]: "The use of "Associated types" improves the overall readability of code by moving inner types locally into a trait as output types" [Rust By Example: Associated types](https://doc.rust-lang.org/rust-by-example/generics/assoc_items/types.html)
[^once1]: Most comments on [What is the difference between associated types and generics?](https://www.reddit.com/r/rust/comments/waxk1l/what_is_the_difference_between_associated_types/)
[^once2]: [On Generics and Associated Types](https://blog.thomasheartman.com/posts/on-generics-and-associated-types) TL;DR section
[^once3]: [When is it appropriate to use an associated type versus a generic type?](https://stackoverflow.com/q/32059370/268898)
