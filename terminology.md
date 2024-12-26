# Terminology

## Avoid

The following terms are to be avoided. Typical reasons include ambiguity, multiple uses and
programmer confusion.

* "static": The term static means many different things in different contexts, is misused by many
  languages and its English definition has a weak connection to its meaning in programming. While it
  could have been profitably used, at this point, it is best avoided entirely.
* "empty type": originally the term "empty type" was used to refer to the two types `void` and
  `never`. But in the course of implementing the compiler it was found that it didn't fully make
  sense to combine those into a single category. Further, since `void` really acts more like a
  special unit type, it was decided that it was misleading to apply the term "empty" to it.
* "source": This is an improper abbreviation of "source code". In fact, "source" does not clearly
  refer to code. Instead use "code" or "source code" (e.g. `CodeFile`).

## Caution

* "standard": Do not use the term standard to mean the typical or ordinary one. Prefer "ordinary"
  for this. The term "standard" has too many meanings and can lead to confusion.

## Decisions

* "Generics", "Generic Class", "Generic Function", "Generic Parameters", "Generic Arguments":
  Generics seems to be the correct term for what is being done here. Sometimes these are called
  "Type Parameters", but in Azoth they may not be types. In some ways, the Azoth model is closer to
  C++ or D templates. However, the template terminology was avoided because templates are
  structurally typed and carry negative connotations in many people's minds. The term "Generic
  Parameter" and "Generic Argument" are less than ideal though because their English meaning is not
  specific enough.
* "Implicit Interfaces": Even though Azoth has "traits", the term "interface" is correct here. A
  [trait provides methods](https://en.wikipedia.org/wiki/Trait_(computer_programming)) for classes
  implementing it. An [interface specifies only a set of
  behaviors](https://en.wikipedia.org/wiki/Protocol_(object-oriented_programming)) (i.e. methods) a
  type must support. In Azoth, implementing the interface of a class does not inherit any method
  implementations because it isn't possible to guarantee that the method implementations don't
  depend on state that isn't inherited. Thus, it is correct to say that the class implicitly defines
  an interface rather than a trait.
  **TODO:** update this. It is no longer true since the idea of trait functions
* Naming of types and functions with varying levels of direct support from the compiler has been
  challenging. There are a lot of different words in use by different languages and sources. Those
  words can be vague and confusing. At the same time, there have been multiple flavors of supported
  types that seem to need distinct names.
  * Simple Types: value types like `int32`, `int`, `decimal`, `float32`, `bool` etc. that have
    keywords in the language and represent single atomic values. Within this are the literal types.
    * Literal Types: `int[V]`, `bool[V]`, `decimal[V]`. The simple types of literal values.
  * Built-In Types: any type that has keyword in the language. This includes the simples types as
    well as types like `Any`, `void`, `never`, and `Self`.
  * Intrinsic Types and Functions: these are types and functions declared in packages but
    implemented by the compiler. The `#Intrinsic` attribute is used to mark these and the
    `azoth.compiler.intrinsics` package is an empty package that can be used to indicate a
    dependency on the compiler providing certain intrinsic types and functions.
