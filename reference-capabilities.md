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
possible to promote a reference to a variable to a reference to an interface it implements). A
second common example is references to the elements of a list that would be invalidated when the
list was resized. Note that both of these have the character of using references to members of
objects and so it might be possible to have a scheme that ruled these out while still allowing for
the safe cases.
