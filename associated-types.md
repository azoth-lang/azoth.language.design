# Associated Types

## Use Cases

### Red-Green Trees

The red nodes should have an associated type for the green node. In particular, see the need for
reflection in Draco's `SyntaxList<T>`.

### Units of Measure

When designing the units of measure library, there were several times where we had to work around
the lack of associated types.

Examples:

* Given a fractional unit like `FractionalPixels` what is the result of calling `RoundUp` etc. (e.g.
  in the case of `FractionalPixels` it is `Pixels`).

**TODO:** figure out exactly what the issue was and document it here.
