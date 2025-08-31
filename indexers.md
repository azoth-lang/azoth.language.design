# Indexers

Originally Azoth had `ref` and `iref` and list elements were accessed via `list.at(i)`. Since that
returned an `iref var T` it was possible to assign a list element with `list.at(i) = v`. When `ref`
and `iref` were removed and replaced with hybrid types and `Var[T]`, the syntax was carried over and
indexed setters were created to allow `list.at(i) = v`. However, indexed setters either require
indexed getters or odd rules around setters and methods. This is because the indexed setter must be
paired either with an indexed getter or with a method `at(read self, index:size)`. If indexed
getters are added then they seemingly duplicate the functionality of methods. If they are not added,
then Azoth must allow a setter and a method to have the same name and when analyzing `list.at(i) =
v` it must avoid turing the left-hand side into a call to the method even though it is a match.

Given the complexity of indexed getters and setters. Maybe instead it makes sense to allow these via
overloading function application. This is how Scala works where `list(i)` calls `list.apply(i)` and
`list(i) = v` calls `list.update(i, v)`. Of course in Azoth these would be operators instead of
magic method names. Indexed getters and setters do not cause the same issues as they do in VB.NET
because there are not types that directly support indexing. All indexers are on properties. Still,
it seems odd that there are these sort of collection like properties that don't have any other
members.

Providing indexers does fix with the idea that dictionaries and lists are functions mapping
keys/indexes to values. However, it opens up the possibility of other function like objects which
isn't great. It also may cause some confusion with a juxtaposition operator since `x (a)` looks like
both a call and juxtaposition.

Options:

```azoth
// indexers
v = list(0);
list(0) = v;

// indexed getters and setters
v = list.at(0);
list.at(0) = v;

// get/set methods (may conflict with getter/setter method groups e.g. o.prop.get)
v = list.get(0);
list.set(0, v);

// methods
v = list.at(0);
list.set_at(0, v);
```

Allowing `get` and `set` methods wouldn't conflict with `o.get.prop` since methods don't have
properties and so it would never be ambiguous.

One thing to think about is how it affects other names. For a dictionary, methods like `get_or_add`,
`try_get` etc. make sense. While `get_at_or_add`, and `try_get_at` are awkward.

## Decision

Allow and use `get` and `set` as method names.

* It keeps the language simple.
* It doesn't allow function-like objects.
* It creates a good API for other methods on collections.
