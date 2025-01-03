# Built-in Types

I'm not fond of the name "primitive type". It doesn't have a clear definition that makes it obvious
which types ought to count as primitives. Furthermore, there are types like `Any`, `Type`, `never`,
and `void` that don't seem like they should be "primitive", but are still provided by the language.
For a while I used the term "predefined types" instead. I have now settled on "built-in" types. This
seems to be a standard term that expresses what they are. I use the "simple types" term from C# for
the built-in value types.

## Numeric Types

In the original design numeric types were named after their bit length (e.g. `int16`), but there
were still fixed length types `int`, `uint`, and `float`. Those were the default sized types.
However, later it was decided that `int`, `uint`, and `decimal` should provide arbitrary sized
values.

**TODO:** Update this section, the default size is now unlimited.

### Default Sized Types

When the design had default fixed length types, here is how those lengths were determined. Both Rust
and Go use integer values to indicate the bit length of types. However, C# did not and it was
already planning for 64 bit architectures. The numeric sizes were selected based on the following
criteria:

1. Anything larger than 64 bit pointers is unlikely any time soon
2. Old 32 bit machines could have 64 bit pointers
3. We want to be friendly to embedded processors that might be special
4. We want standard sizes to be nice
5. Platform dependent sizes cause issues

The default sizes (i.e. those that leave off the bit length) were chosen as a compromise between
small enough memory use and enough digits of precision to be useful. Hence 32 bit integers with
their more than 9 digits were chosen as the default integer size while 64 bits were chosen for
floats and decimals because their 32 bit versions had only 7 digits of precision.

### Arbitrary Sized Numerics

As Azoth was created from the previous language Adamant, it became higher level. At the same time I
had been reading about the implementation of arbitrary precision integers. I became convinced that
their performance could be made good enough for most cases. They also came with a enough advantages
that it seemed worth it.

Advantages:

* Eliminates overflow errors
* Developers don't need to consider bit size for every type
* Eliminates an odd question of whether there should be synonyms for the default sized types (e.g.
  if `int` is a 32-bit integer, should there also be an `int32` type that could be used for clarity.
  If not, there were sort of random holes in the sized types since the defaults were chosen for
  subjectively having enough precision.)

## Optional Type

Different languages use different names for option types (see [Wikipedia Option type: Names and
definitions](https://en.wikipedia.org/wiki/Option_type#Names_and_definitions)). At first, the C#
convention of referring to them as nullable was followed because it was familiar. It also seemed to
make clear that developers should expect that for reference types it would be represented as
efficiently as a null pointer. However, the name "nullable" seemed to carry with it too much history
and implied that they were bad or dangerous. A more neutral name seemed appropriate. Looking at the
options, it seemed that "optional" was the clearest. Both Java and Swift name option types that and
they are two of the languages most are concerned with clear names. Like Swift the `T?` syntax means
it isn't necessary to write out the name optional.

When considering what to name the two values of an option type, `Some[T]` and `none`. Seemed the
clearly better choice. However, that has been further simplified by eliminating the need for a
`Some[T]` type.

## Tuples

Generally, I'm not a fan of tuple types. I feel like their unnamed fields are an anti-pattern
leading to confusing code. However, they are frequently used in languages that have them. Probably
too frequently used. It wouldn't be hard to create tuples in Azoth by declaring a `Tuple` type
overloaded on the number of type parameters or with a `Tuple[Values...]` type. That would certainly
be done if they weren't included in the language. Additionally, lists of types are essentially tuple
types so it will make working with generics easier. Given that, it makes sense to include them in
Azoth. *Actually, declaring a non-built-in tuple type turns out to be quite difficult because of the
interaction in the `params` keyword. The compiler needs to understand tuples. Declaring ways for the
compiler to construct an arbitrary tuple like type would requiring tuples as parameters leading to a
circular definition.*

In many other languages tuples have a syntax like `(x,y)` but that seems easily confused with
grouping. Also, there is no good syntax for a tuple of one value. Some languages use `(x,)` which
just seems ridiculous. When the new operator was being used for all values including structs, tuples
would be declared as `new (x,y);` and would not be ambiguous (that syntax could have been an issue
for placement new). However, this would still not have been a very distinct syntax for tuples. For
example, a new single tuple would be `new (x)` and new empty tuple would be `new ()`. Since Azoth
does not have built-in arrays, the square bracket symbols are available for use with tuples. This
provides a much more distinct syntax and has precedent in other languages. For example the language
[E](https://en.wikipedia.org/wiki/E_(programming_language)) uses them for tuples. However, it was
confusing, still too short and the square brackets seem too valuable to give up to a feature that I
fundamentally want to discourage.

For those reasons, various longer syntaxes were considered. Eventually, the idea of using the hash
sign for all composite type initializers was hit upon. This grew on me as a good syntax for tuples.
So `#(1, 2)` became the syntax with `#()` as an empty tuple. But note, I think actually other types
could use this composite constructor if it made sense. For example, maybe math vectors should.

Tuple element access using square brackets could be confusing. For example, `#(x, y)[0]`. Also it
makes it seem tuples could be accessed with non-constant indexes and would encourage people to
declare named constants for tuple indexes. It makes more sense to go with the admittedly ugly Rust
style of using integer field names like `t.0` and `t.1`. However, that has a parsing issue with
`t.0.1`. Admittedly, nested tuples are a bad idea anyway. Eventually, it dawned on me that these are
fields with integer names, so we could just escape the names the same way we escape keywords to use
them as names, so `t.\0.\1`. This has the further effect of making the syntax ugly. I like that
because I am discouraging tuples and even when they are used, I'd prefer people destructure them.
