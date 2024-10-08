# Operators

## Range Operator

There were many options for the range operator syntax. Initially, it was thought it should be
inclusive of the start value and exclusive of the end value. This would make the common C `for` loop
case the default. However, it was very difficult to come up with a good syntax for the inclusive
case. After looking at Swift and reviewing the [operators used by other languages for
range](http://rigaux.org/language-study/syntax-across-languages/VrsDatTps.html#VrsDatTpsRng) this
was changed. It appears most languages use `..` to be inclusive of both ends. This then opens up
`..<`, `<..` and `<..<` for ranges exclusive of their end values. This syntax was inspired by the
Swift `..<` operator for half open ranges. It was decided that C style for loops should now be rare
enough that it isn't an issue that they have a longer, more awkward syntax.

## Accept Operator

For years, an accept operator had been planned. This was to be the symbol `..` and would be used as
the accept method in the visitor pattern. It would also be a way to take a function pointer and use
it like a method on its first argument type. When Azoth was being designed, having a ranger operator
was found to be important and `..` has become a defacto range operator. It is a good fit for this.
While Swift uses `...` for this, it seems like that may be needed for something like list expansion.
For a while it was thought that `..` could be used as both the accept operator and the range
operator since the operand types would be distinct. However, those operators need to have radically
different precedence. The range operator should have a fairly low precedence while the accept
operator has a high precedence like member access does. Then it was thought these cases could be
distinguished syntactically by looking for an invocation afterward. However, `x..y()` could equally
mean that `x` accepts the visitor `y` or that a range is being constructed from `x` to the result of
`y()`.

## Overload Not a Trait Implementation

Operators are not tied to simple methods on special traits the way they are in Rust because that
would imply that they have universal semantics. That is, that every use of the operator is meant to
have the same semantics. However, just like method names, the same operator can have two different
meanings in different contexts. Indeed, a given operator may have different pre- and post-
conditions on different types.

## Overloaded Operator Operand Placeholders

Inspired by the Agda language there had been thought of using `_` as a placeholder in operator
overload declarations. At first, this was considered only to support future expansion of the
language to support arbitrary mixfix operators. However, when the user and string literal operators
were added, it seemed prudent to include the `_` placeholder with them. Then when overloading of the
range operators was considered, it was realized first that they had both prefix and suffix unary
forms to the placeholder was needed to distinguish these. Then it was further realized that there
were both nullary and binary forms of these operators. Thus `..` would be a nullary operator while
`_.._` would be a binary operator. For consistency, it then made the most sense to require the
placeholders for all binary operators.

## Operators as Functions and Infix Functions

It was evaluated whether Azoth should allow the use of operators as regular prefix functions. This
would be similar to how Haskell allows infix operators to be used as functions. Indeed, the syntax
could have been the same. That is an operator could have been used as a function name by surrounding
it with parentheses (i.e. `(+)(x, y)`). However, with the decision to follow the Scala 3
recommendation of having all operators be aliases for named functions, that becomes far less useful.
The developer can always just use the function name.

The inverse of that is the ability to use functions as infix operators. In haskell this is done by
surrounding the function name with backticks (i.e. ``x `div` y``). This is far less valuable in
Azoth because most functions will be methods and therefore already have an "infix" notation.
Additionally, it wouldn't be clear what precedence to give them. It would be especially confusing if
they were aliased as operators that did have precedence. For example, ``x `plus` y `times` z`` might
be expected to give higher precedence to the multiplication since that function was aliased as an
operator with higher precedence. Finally, there didn't seem to be a good syntax for doing this that
didn't use up symbols that could be better used for other purposes. Given all this it makes sense to
not support using functions as infix operators unless experience shows it to be important.
