# Type Lists

Maybe generic type lists (e.g. `Example[T...])`) are really just a generic parameter that is
restricted to be a tuple type in the VM. Then there are ways of manipulating tuples and tuple types
that enable the desired functionality.

One issue with this is how `lent` works. If a type list represent the parameter types of a method,
one might want to be able to control which params are `lent`. You also need a way to bridge from
methods taking a tuple and methods taking individual parameters. Unless the calling convention
somehow makes those identical.

Maybe tuples sort of don't exist in the VM. When used as parameters, they expand to multiple
parameters. When used as variables and fields that expand to multiple variables and fields. Only
when returning or throwing them would they need a data structure representation. Of course, it may
be that the data structure representation isn't much different than how they would be represented
stored. I guess the exception would be reordering. Local variables might have their storage order
changed. Having them not exist would severely limit what methods they could have, but Azoth doesn't
really require them to have methods.
