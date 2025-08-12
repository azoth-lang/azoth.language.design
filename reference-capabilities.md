# Reference Capabilities and Movement

## Class Capabilities

It might be thought that classes should be able to be marked `iso`, `mut`, or read-only in addition
to being marked `const`. These then might mostly represent a restriction on what kind of references
can be contained in them. However, when one thinks through how this would work it doesn't work out.
First, if these are not properties of the type inherited by all subclasses of the type, then they
have no influence on how references to the type work. They are just standard references capable of
having any reference capability. One of the primary benefits of `const` types is that the compiler
can assume every reference to one has the `const` capability and therefore avoid breaking isolation.
Additionally, there is no reason to restrict the use of `iso` inside a class marked `mut`. From the
perspective of the consumer of the class, it behaves no differently than any other mutable type.
Instead, the limitation that is desired is that it act as a `move` type. This is much more clearly
indicated with the `move` modifier and consistent with the other changes caused by it (e.g. can
contain move types and have a destructor).

## Mutability is Distinct from Exclusive Access

It is troubling that in Rust a lock-free queue would be held through immutable references yet one
could call methods that mutated the queue even though they were not marked as mutating the object.
Ideally, there would be a way to mark thread safe classes so that their mutability was correctly
reflected, but it would then be considered safe to have multiple mutable references to it. There are
potential issues with that however due to memory invalidation. In particular, assigning an enum
struct to a different case could break a reference to a field in the struct (remember that it is
possible to promote a reference to a variable to a reference to an trait it implements). A
second common example is references to the elements of a list that would be invalidated when the
list was resized. Note that both of these have the character of using references to members of
objects and so it might be possible to have a scheme that ruled these out while still allowing for
the safe cases.

## Mutate Expressions

Originally, it was thought that when using a variable name one ought to receive only a read only
alias to the object. It would then have been necessary to use a mutate expression to get a mutable
alias even if the original alias supported mutation. This would be done as `mut x`. Thus any time
one passed a mutable reference to a function it was necessary to use the `mut` keyword if it came
from a variable or field. This was not necessary if the mutable reference was produced as the result
of a function or constructor call. There was also an exception for the `self` parameter.

This design reflected the idea that mutability should be restricted and visible. This principle is
the reason that the default reference capability is read only and that the type inference will infer
read only variable types in preference to mutably variable types. Mutate expressions made the
mutation of locally controlled state visible.

Early testing showed this was annoying when writing code and had very limited benefits. It was easy
to forget to include the `mut` keyword. Of course, that could have been due to a lack of familiarity
since there is no equivalent in other languages. However, it also seemed to have vary limited
utility. Typically it is clear from the function name and context whether something would mutate.
With read only being a strong default many variables already do not permit mutation. When the
variable declaration already has to be declared mutable it seems very redundant to require it again
at the use site. Removing this feature not only simplified the compiler, but streamlined the
language and the code written in Azoth.

## No Exclusive Mutation

It is reasonable to imagine having a reference capability that allows other read aliases but has
exclusive mutation access (i.e. no other mutable references). This is called transitional (`trn`) in
Pony and was tentatively given the keyword `xmut` in Azoth. Having this would provide more precise
types for mutable iterators. Without it, mutable iterators need to reference the collection with
`temp iso` which restricts there to be no other readers of the collection. However, with `xmut` it
could be `temp xmut` and allow other reading of the collection while it is being iterated. This is
the primary use case found so far. However, it should also have uses when extracting methods because
it may be necessary to have `xmut` to convey the capabilities of a reference in the middle of a
method.

Because of the above uses, adding `xmut` to Azoth was briefly tried. In the process, it was apparent
that it adds complexity for the developer who has to decide when to use it versus other reference
capabilities. It also has to work similar to `iso` where it must be changed to `mut` via flow typing
because mutable references must be made to it in order to call methods and pass it to functions.
That would then necessitate a way to recover `xmut` similar to but distinct from `move` (since
`move` would recover `iso` which would sometimes not be possible even though recovering `xmut` was).

Given the complexity it would add to the language, it didn't seem worth it for what appears to be an
edge case on an already rarely used feature (i.e. mutable iterators). Also, Project Midori
apparently was able to make due without it. Thus `xmut` has been removed. However, if more
compelling use cases are found it may be added back to the language especially if a way to make it
simpler for the developer to reason about can be found.

## Independence Distinct from Variance

The interaction between independence and variance is confusing to reason about. For a while, it was
thought they were related since independent type parameters can vary their capabilities in a
covariant way. Albeit in a way that doesn't behave the same as other variance. However, it seems to
be apparent now that they should be completely distinct.

Consider the incorrect code segment below. It is a simplification of the relationship that exists
between `List[T]` and `Iterable[T]`.

```azoth
// A read-only container
public trait Read_Box[T out]
{
    public get value(self) -> T;
}

// A container for an independently typed value
public class Box[T ind] <: Read_Box[T] // ERROR independence not maintained
{
    public override get value(self) -> T {...}
}

// In a method
let b: mut Box[mut Square] = Box[mut Square](Square());
let rb: Read_Box[mut Shape] = b;

let f: const Box[const Square] = freeze b;

rb.value.scale(2); // could mutate the square in f.value that ought to be const
```

If `Box` were not declared with an independent type parameter, then the implementation of
`Box::value` would force the return type of it to be `self|>T`. That would in turn force the return
type of `Read_Box::value` to be `self|>T`. However, there is nothing that requires `Read_Box::value`
to be declared with that return type. For example, an implementation of the trait could be using a
factory to create the `T` values and so they would not be accessed from `self`.

Note that `rb` shares with both the outer layer of `b` and the inner layer of `b`. However, because
`rb` does not allow writing, it does not interfere with freezing `b`. The declared types of both
`Box::value` and `Read_Box::value` are valid and seemly compatible. Even as the `Box::T` type
parameter capability varies, it will be accommodated by the fact that `T` is declared `out` in
`Read_Box`. Thus, if the above code were allowed, it would be a problem because now `rb.value: mut
Shape` could be used to mutate the shape inside `b` that ought to now be constant.

Given that, the above code should not compile and is rejected for not maintaining the independence
of the type parameter `T` with the subtype declaration `<: Read_Box[T]`. The correct thing to do is
declare the type parameter of `Read_Box` as `T ind out`. That has the added benefit of keeping the
tracking of the two layers separate even when upcasting to `Read_Box`.

From this example, it can be seen why independence is distinct from variance. As a result, it has
been decided to make them fully distinct. Hopefully, that will work well and be simple to use.
