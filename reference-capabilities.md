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
