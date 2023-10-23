# Patterns

It was originally planned that pattern matching would work similar to how it works in Rust and Swift
with the `if let` and `while let` constructs. However, that both limits the use of pattern matching
and produces strange syntax.

It limits the use of pattern matching because it is not possible to match a pattern as part of a
boolean expression outside of the condition expression of `if let` or `while let`.

The syntax oddity it would introduce is that `let pattern` matches a pattern and so `let let
pattern` ought to be a valid pattern. If instead, one says that any identifier in a pattern
introduces a binding then one is faced with non-obvious variable declarations inside of match arms
and match expressions.

The C# style match expression introduced by `is` gives a clear indication of when a pattern match is
being performed.
