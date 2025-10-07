# Closures

To simplify the type system, I'd like to be able to build closures out of simpler constructs.
Specifically, they could be built from basic function pointers. A function pointer type would be `@`
followed by the parameter and return types including which parameters are `lent`.

## Three Kinds of Closures

Function pointers can refer to basic functions. But there are three kinds of closures they can't
directly handle:

First, there are reference type method references. Code that takes a method reference (e.g.
`obj.method_name`) needs to close over the object reference and the virtual method call. One
challenge here is that a function pointer cannot point to a virtual method. Thus it is not simply a
pair of an object and a reference to the method. Perhaps a wrapper function that makes the virtual
call could be generated.

Second, there are value type and own struct method references. Code that takes a value method
reference (e.g. `val.method_name`) and that takes an own struct method reference (e.g.
`struct.method_name` where `self` is `iso` or `own`) needs to close over the inline value or struct.
Here a function pointer could point directly to the method because the method can be identified
statically. However, the value is not a reference and may be a large inline value. It needs to be
stored into an object. For structs, it must be possible to move the struct value out of an `iso`
reference and leave an invalid object that is no longer reachable.

Third, there are local closures which need multiple local variables etc. These could be a mix of
aliases and copies of the locals and `Ref[T]` or `Var[T]` references to the local variables
depending on the lifetime of the value.

## Object Model

One way to do this would be to have a abstract generic class with an `invoke` method that was
parameterized using a type list. A subclass could then be generated for each closure which could
hold the necessary data and variously make the virtual or function call. In some cases, generic
subclasses could be used to avoid an explosion of subclasses. The self parameter type would
determine what kind of function it was. For example, an `iso` self parameter type can be called only
once.

An issue with this is that `lent X` is not a valid generic parameter type. Thus it would be
difficult to make it generic over whether the parameters are lent. One could imagine having the
generic parameter being a function pointer type. However, it isn't clear how that would work. I
guess what you would need to do is have the subtype method have a sealed method that could be
exposed as a function pointer. But then how would it take the `self` reference? There would have to
be something unsafe going on. Or else, there would have to be some way to destructure the function
pointer type into a parameter list for the method.

## Value Model

Another way to do this might be to have a value type which held a function pointer and a value to pass to that pointer. Again, there would either need to be unsafe things happening, existential types, or some way to have generic parameter types including `lent`.

```azoth
published closed value Closure[Params..., Return, Throws]
{
    public value Method_Closure[T]
    {
        let obj: T;

    }
}
