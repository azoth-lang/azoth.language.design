# Terminology

## Avoid

The following terms are to be avoided. Typical reasons include ambiguity,
multiple uses and programmer confusion.

* "static": The term static means many different things in different contexts,
  is misused by many languages and its English definition has a weak connection
  to its meaning in programming. While it could have been profitably used, at
  this point, it is best avoided entirely.

## Decisions

* "Generics", "Generic Class", "Generic Function", "Generic Parameters",
  "Generic Arguments": Generics seems to be the correct term for what is being
  done here. Sometimes these are called "Type Parameters", but in Azoth they may
  not be types. In some ways, the Azoth model is closer to C++ or D templates.
  However, the template terminology was avoided because templates are
  structurally typed and carry negative connotations in many people's minds. The
  term "Generic Parameter" and "Generic Argument" are less than ideal though
  because their English meaning is not specific enough.
* "Implicit Interfaces": Even though Azoth has "traits", the term "interface" is
  correct here. A [trait provides
  methods](https://en.wikipedia.org/wiki/Trait_(computer_programming)) for
  classes implementing it. An [interface specifies only a set of
  behaviors](https://en.wikipedia.org/wiki/Protocol_(object-oriented_programming))
  (i.e. methods) a type must support. In Azoth, implementing the interface of a
  class does not inherit any method implementations because it isn't possible to
  guarantee that the method implementations don't depend on state that isn't
  inherited. Thus, it is correct to say that the class implicitly defines an
  interface rather than a trait.
