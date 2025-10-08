# Dictionary Keys

Related to the problems with equality are the problems with dictionary keys. A dictionary must have
a way to compare its keys. For a hash based dictionary this is an equality comparer and
corresponding hasher. For a sorted dictionary this is a key ordering. The problems are analogous so
here we'll just look at key ordering.

## Actual Requirement

For a dictionary to work properly, keys must be found in the same location as when they were sorted.
This means that the key ordering must be stable over time regardless of what (safe) code is run.
There are three ways that could be true:

1. The key is constant (i.e. `const T`)
2. The key is an object id (i.e. `id T`)
3. The key can be mutable but the comparison is still stable because either:
   * The comparison happens by `id`.
   * The comparison considers only portions of the key that are immutable (whether they are marked
     `const` or not).

The question then becomes how should this be reflected in the API of the dictionary? There seem to
be five possible options.

## Option 1: Keys are `const` or `id`

The dictionary can be declared to restrict the keys to always have the capabilities `const` or `id`.
This can be achieved with the current design of the language by using the `shareable`.

```azoth
published trait Dictionary[shareable Key ind, Value ind nonwritable out] { ... }
```

**Pros:**

* Very simple
* Does not impose any constraints or complexity on the infrastructure of comparers.
* Having an `id` type as a key should allow for defaulting hash dictionaries to the identity
  equivalence. This would now be the expected and common case when using that kind of dictionary.

**Cons:**

* Prevents one from making a dictionary with mutable keys:
  * In particular it prevents a dictionary with mutable keys compared by `id` which might be useful
    to allow an algorithm that needs to look up by key but also uses the keys collection as a set of
    values that can be mutated. In particular, one can imagine a high performance situation where
    sometimes one loops over key-value pairs and mutates the keys based on the values. Without
    mutable keys, one would be forced to keep a second set of the keys and when looping through it,
    do a lookup on each key.
  * Also prevents the less important case of having a comparison on an immutable portion of the key.

## Option 2: Dictionary Comparers Operate on `const` or `id`

The dictionary can be declared to restrict the comparer to only operate on `const` or `id` versions
of the key type even though that type can have different capabilities. This seems to require several
odd additions to the language. First, it must be possible to express a further constraining of the
reference capability of a generic parameter by a capability constraint (e.g. `shareable |> Key`).
That is not currently supported. Nor does Pony support the equivalent `#share->Key` (though testing
shows it does support using arbitrary capabilities in arrow types which is not documented beyond
`box->A`). Second, the compiler must be able to determine that it is safe to compile code that
passes a value of type `Key` to a method with a parameter type of `shareable |> Key`. I can
logically determine that is safe since the default capability set is `aliasable`. However, it isn't
clear what the general functionality is that enables that.

Note that `shareable |> Key` is not a natural extension of `self |> Key`. The latter is a shorthand
for the method being generic over the capability of self and thus ultimately resolves to a concrete
capability viewpoint when the method is called from any given context. `shareable |> Key` is truly a
capability set viewpoint. It really seems to be a different operation. It is more properly an upcast
to the lowest capability in the `shareable` capability set. Indeed if it were to act like `const |>
Key` then it would make `mut` and `read` types into `const` which is not what is needed here.

```azoth
published trait Dictionary[Key shareable ind, Value ind nonwritable out] { ... }

published trait Sorted_Dictionary[Key shareable ind, Value ind nonwritable out]
    where Key <: shareable |> Key
{
    published get key_comparer(self) -> Ordering[shareable |> Key] { ... }

    published init(mut self, key_comparer: Ordering[shareable |> Key]) {}
}
```

**Pros:**

* Enables mutable keys compared by `id`.
* Requires additional language features and complexity.

**Cons:**

* Prevents one from having a dictionary of mutable keys compared by an immutable portion.
* Requires `shareable ind`

## Option 3: Stricter Orderings

The core equivalence and ordering traits do not require that they are stable when the values being
compared are mutated. This makes sense since they can be used for cases like sorting a list of
mutable values by their current state. An additional, parallel set of traits could be added that
impose the first restriction that the comparison is stable. The dictionary would then accept these
stricter traits for use when comparing keys. One challenge with this is that it is a purely semantic
difference with no difference in the exposed API. This would make it easy to incorrectly implement.
Especially, developers would easily forget to implement the more strict trait when appropriate and
thereby accidentally prevent the comparison from being used to compare dictionary keys.

**Pros:**

* Supports all use cases.
* Provides general stable equality semantics for other uses.

**Cons:**

* Requires many more traits.
* Easily misused by developers.
* Confusing to developers.

## Option 4: Do Not Restrict API

The dictionary type could be declared with no restriction on the key or the comparer that attempts
to enforce the condition. Instead, it is up to the developer to only construct dictionaries with
stable comparers. This is the approach adopted by all the existing languages I am aware of. I
believe this still requires that the `Key` generic parameter be declared `Key shareable ind`.

**Pros:**

* Supports all use cases.
* Doesn't require any additional or complex language features beyond `shareable ind`

**Cons:**

* Developers can accidentally do the wrong thing.
* Still requires `shareable ind`.

## Option 5: Keys are `const` or `id` but Sets Are Unrestricted

**Pros:**

**Cons:**

* Inconsistent
* Does not enforce safety on sets.

## Discussion

* I think option 3, stricter orderings, is out. It adds too much complexity and is still error
  prone.
* If capability set viewpoints or their equivalents are needed in the language for other reasons,
  then option 2 makes a lot of sense. But if not then it is an additional complexity that doesn't
  seem like a good idea.
* Essentially the same logic applies to the values for a set. For sets the use case of having
  mutable values distinguished by identity seems much more important. For dictionary keys it is
  conceivable that azoth could get away with just requiring that the key be `const` or `id` and if
  users needed something else, they could make a specialized collection type. For sets that doesn't
  seem viable.

## Decisions

Options:

1. ~~Keys are `const` or `id`~~ (At least not as a general solution that applies to sets
  too. For dictionaries only, it is still possible.)
2. Dictionary Comparers Operate on `const` or `id`
3. ~~Stricter Orderings~~ (Too complex and still error prone.)
4. Do Not Restrict API
5. Keys are `const` or `id` but Sets Are Unrestricted

For now, I am going to try option 2 and see what that does to the language. I will add the ability
to do `shareable T` where `T` is a generic type. This means the type `T` upcast to a shareable
capability. Applying this to an `independent` type requires that it be `independent(shareable)`.
This will be part of a larger system of allowing capability sets to be applied to generic types.
This seems like it will allow `aliasable T` to act as the equivalent of alias types in Pony (i.e.
`T!`).
