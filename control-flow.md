# Control Flow

## No Python Style Loop Else

Python's loop else condition runs after the loop as long as break was not used. A feature like this
was considered, though it would have needed a better keyword. However, that construct is more useful
when selecting an item out of a collection, and in Azoth this will *not* be done with the `for`
loop. Since `for` loops operate on iterators, using a filter on the iterator is a better and easier
way to do this. Also, it wasn't clear how to combine such a feature with the existing loop else
feature. If it were added, a keyword sequence like `if not break` might be good.

## Control Flow Requires Blocks not Parenthesis

Control flow like `if`, `for` etc. require block statements and do not have parentheses around the
expression just like in Rust. Originally, control flow was C style where the expression portion must
be surrounded with parentheses, but the statement can be any statement. The current approach solves
the dangling else issue and also makes loop else unambiguous (i.e. people won't think the loop else
is for an if). More importantly, I think this makes scopes clearer. Whatever effect scopes have on
value lifetimes, they will be visually apparent in the code. With C style control flow, there is a
scope introduced, but it may not be visually apparent when it is not surrounded by curly braces.

## `loop` Keyword

As in Rust the justification is that the control flow analyzer treats it differently. Also, it just
makes the thing clearer than something like `while true`.

## `=>` Operator

The expression evaluation operator was the result of trying to come up with a syntax for if
expressions and after considering many different syntaxes, realizing that this was essentially the
same problem as match, and Rust used `=>` for match expressions which I was likely to copy just for
familiarity.

Before this syntax, the use of a keyword `is` was considered. It was thought of as mirroring the
exiting of functions and loops.

| Structure                 | Exit     |
| ------------------------- | -------- |
| Function                  | `return` |
| Loop                      | `break`  |
| Choice (`if` and `match`) | `is`     |

This might have looked like:

```azoth
let x = if cond
        {
            DoSomething();
            is 5;
        }
        else
        {
            SomethingElse();
            is 6;
        };

let y = if cond is "Hello" else is "World";
let z = match v
        {
            [0, y]
            {
                Action();
                is y;
            },
            [x, 0] is x,
            [x, y] is x + y,
        };
```

But using `is` for this was also inconsistent with with C#'s use of `is` for type checking. It
seemed prudent to leave the `is` keyword available for that. (Indeed, later versions of the design
adopted `is` for pattern matching.)

## `next` Instead of `continue`

Most C family languages uses the `continue` keyword to go to the next iteration of a loop. Azoth
does not. Instead, it follows Ruby and R in using a `next` keyword. There are two reasons for this.
First, it conveys the meaning better. The continue keyword has always seemed like it should continue
from the current point, not continue the next iteration. Second, when combined with a loop label it
can become even more confusing. A statement like `continue outerLoop;` would read as if the current
loop should be exited and the outer loop should continue where the inner loop stopped (i.e. the
semantics of `break`). Whereas `next outerLoop;` very clearly reads that it will execute the next
iteration of the outer loop.

Since Azoth follows the C family of languages in so many other regards, `continue` will be
recognized as a contextual keyword synonym for `next` that generates a non-fatal error.

## Loop Label Syntax

Since the beginning I have wanted to support Java style loop labels with break and next referring to
labels. I have always thought that was the only proper use case of the C# goto. Likewise since
jumping between switch/match cases is not supported there is no need for the C# style goto keyword.
However, it has been a real challenge to find a good syntax for both declaring and using loop
labels. In Adamant, there were named scopes and it was thought that loop labels should be
syntactically similar to named scopes. However, Azoth doesn't have anything like that. In Azoth, all
loops can be expressions. That makes it more challenging because the syntax has to be unambiguous
even when it is part of an expression. Looking at other languages, a label followed by a colon is
the almost universal syntax for loop labels. Only Kotlin breaks with that and uses the at sign (e.g.
`label@ for ...`). I resisted using the colon because in Azoth it is so tightly associated with
typing and subtyping. That is part of why the named parameter syntax doesn't use colons. At one
point, I thought that treating labels like pointers and using a syntax involving the at sign made
sense. However, this wasn't quite right. If the label was the target of a goto that it could be
reasonably treated as the address of an instruction. However, the break statement breaks out of the
loop. It using the loop label as a way of referring to the loop not as an address for the loop.

I've designed to accept the label colon syntax for loop labels because it is so universal and there
aren't any better options. Even if type ascription expressions are added to Azoth using the colon,
this syntax won't be ambiguous because a loop keyword following the colon could never be the start
of a type name. There was also some concern about the syntax for break with loop label to ensure
there was no ambiguity between the loop label and the value expression. However, the use of loop
labels will be very limited and there will only be a few labels in scope at any given time. I think
it is safe to treat something as a label if it matches a loop label. If that was not the intended
meaning and it shadows a local variable, the developer should get a compiler error since the loop
will not have the correct type.
