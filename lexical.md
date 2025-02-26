# Lexical Structure

## Raw Strings

Originally, strings were more similar to C#. There were regular strings and verbatim strings which
were prefixed with a pound sign and didn't allow any escape sequences. Two consecutive double quotes
could be used to put a double quote in a verbatim string. This was however less than satisfactory
since verbatim strings could still be awkward when they contained double quotes and weren't
flexible. Most importantly, they would have caused confusion with multiline strings introduced by
triple quotes because `#"""string"""` would have been a valid verbatim string. The use of the pound
sign to introduce verbatim strings came about by analogy to the use of pound sign to introduce
tuple, list, and set initializers.

The lexical structure of strings in Swift appears to have been very carefully thought out. Azoth
borrows its interpolated string and multiline string syntax from Swift. Swift added raw strings (or
delimited strings as the documentation calls them) by surrounding strings with pounds signs. This
offers the benefit that it easily allows double quotes and pound signs in raw strings while still
providing a means of using escape sequences. However, there was concern that this might be confusing
with the use of `##` for the preprocessor and that terminating with pounds signs was inconsistent
with tuple, list, and set initializers.

The use of Markdown style backticks for raw strings was briefly considered. However, it was quickly
realized that backticks would make an excellent syntax for code expressions and blocks that would be
consistent with their use in Markdown. The benefits of Swift style strings seem to outweigh the
drawbacks and have the benefit of bringing Azoth string syntax fully inline with Swift string
syntax.

## Code Expression and Block Syntax

Because Azoth documentation comments use Markdown, the use of backticks in Azoth syntax was
discouraged. Indeed, an early idea of using backticks for identifier escaping was rejected because
of this. However, backticks fit so well with code expressions and blocks where they almost exactly
match their use in Markdown that it was difficult to see why they shouldn't be used. Furthermore,
use of code expressions and blocks in documentation comments should be rare but is still possible
given the rules of markdown. Additionally, a fenced code block in documentation comments could be
directly marked as being DSL code rather than Azoth code thereby avoiding the need for nesting of
code.

## Unicode Symbols

For a long time in the design of Azoth the idea was to have both ASCII operators and their Unicode
counterparts and treat them as interchangeable. For example, `≤` would be equivalent to `<=`.
However, this was an additional source of complexity, variation in code, and topic for code
convention arguments. Thus there really should be only one operator set. Since the Unicode
characters are difficult to type and would probably require special editor support in many cases
(e.g., one types `<=` and the editor transforms it to `≤`), that set has to be the ASCII set.
