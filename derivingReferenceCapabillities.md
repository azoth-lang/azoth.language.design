# Deriving Azoth Reference Capabilities

It is planned that the Azoth programming language will use a system of reference capabilities to statically manage memory and enforce memory and concurrency safety. This approach was pioneered by the Rust programming language which is the inspiration for the Azoth reference capability system. This paper shows how given a fairly flexible categorization of the possible reference capabilities along with a reasonable set of assumptions a short list of reference capabilities for possible use in Azoth can be derived.

Reference capabilities are a category of approaches in the academic literature and in programming languages which assign to each object/value reference a *capability*. The capability determines which operations are valid using the reference and how that reference may be passed to other variables or functions. Capabilities are typically conceived of as static properties of references and thus form a type system. Often, the different reference capabilities form a lattice or subtyping relationship. For Azoth, the most relevant such capability system is that of the Rust programming language. It provides a concrete embodiment of reference capabilities that is helpful for those readers not familiar with reference capabilities. The Rust system is comprised of both built in types and standard library types. It extends from borrowed references through owned references out reference counted references. Since reference counting is not relevant to Azoth, it will not be considered here. The table below summarizes the Rust reference capabilities except for the reference counted ones.

| Syntax        | Rights              | Description                                                                                                              |
| ------------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `Box<T>`      | read, write, delete | An owned reference                                                                                                       |
| `Box<T + 'l>` | read, write, delete | An owned reference that is limited in scope by the lifetime `'l`                                                         |
| `&'l mut T`   | read, write         | A borrowed mutable reference. Borrowed references are always constrained  by a lifetime though this is often implicit.   |
| `&'l T`       | read                | A borrowed immutable reference. Borrowed references are always constrained  by a lifetime though this is often implicit. |

Rust combines these references capabilities with an affine type system for values and references. The result is a language offering compile-time memory management and static guarantees of memory and concurrency safety.

## Rights

Reference capabilities are defined by the various rights they confer to the reference. Each capability system defines its own set of reference capabilities. These are given unique names and general descriptions. It is then difficult to compare different reference capability systems. It has happened repeatedly that different systems give different names to capabilities that are effectively the same.

To deal with this Boyland et. al. created a set of rights out of which to compose capabilities.[^Boyland] This allows different capability systems to be categorized and explained within a single framework by which rights each of their capabilities confers. Each right describes a specific use or property of the reference. The rights are fully independent and can be combined in any way. For example, the right to write to a reference does not imply the right to read from that reference. These rights are listed in the table below.

| Right | Name               | Description                                                              |
| ----- | ------------------ | ------------------------------------------------------------------------ |
| R     | Read               | The value referenced can be read from                                    |
| W     | Write              | The value referenced can be written to                                   |
| I     | Identity           | The reference can be compared to other references for reference identity |
| R̅    | Exclusive Read     | No other reference has a ability to the value with read access           |
| W̅    | Exclusive Write    | No other reference has a ability to the value with write access          |
| I̅    | Exclusive Identity | No other reference has a ability to the value with identity access       |

In addition to the above rights, they defined an additional ownership right "O". The ownership right provided the ability to assert a right and thereby revoke it from all non-owning aliases. Since capabilities in Azoth are purely static, this ownership right is not applicable.

These right can be combined to form capabilities. Any given capability can be characterized as a set of rights. Each of the above rights is either present or not. As a shorthand, capabilities can be named by stringing together their rights. So the immutable borrow `&'l T` capability of Rust is RIW̅. It allows the reference to be read from and compared for identity and guarantees that no other reference can write to the value at least as long as this reference exists.

## Rights Variables

One of the innovations in Azoth is the concept of references with a "ownership flag" or "drop flag". Such references must be treated as if they have ownership of the object. That is, all other references to the object in scope must be gone before they would be deleted. However, the actual deletion is controlled by the drop flag. This allows a caller to choose whether to pass ownership or not. If the caller is done with the object, they can pass ownership to the object to the function. If the caller intends to use the object again later, they can share the object with the function.

How can this ownership flag be represented in the system of rights supported by capabilities? For reasons that will be discussed later, a reference has ownership if and only if it has the I̅ right. However, while these references must be treated like they have ownership, whether they do or not is determined by a runtime value. To represent this, we introduce a variable for the I̅ right. The lowercase greek letter xi "ξ" is used for this purpose. The intent is that xi would stand for e**X**clusive **I**dentity. The variable "ξ" is thus an indicator that the reference capability will carry a flag that determines at runtime whether or not the capability has the exclusive identity right. For safety, the compile time restrictions on any such capability must be safe regardless of whether the reference has exclusive identity or not. Note that combining ξ with I̅ is nonsensical. Thus while all other rights are either present or not, this right effectively has three possible states, present, not present, presence flag.

## Restrictions

*Note:* the terminology, notation, and explanations around regions and lifetimes discussed in this section are in flux. No entirely satisfactory description of them has yet been found.

So far, we have discussed rights that grant a reference access, though they may do so be restricting what access other references can have. In Azoth, there can also be restrictions on capabilities. Similar to Rust, Azoth must track what Rust calls "reference lifetimes". Reference lifetimes limit the scope a reference is valid for. In Rust, all borrowed references have a lifetime, but owning references may or may not have a lifetime. We'll see that Azoth may be able to offer more flexibility in this regard.

To represent the lifetime restriction placed on a reference capability we use the notation "⤳ρₙ". This indicates that the reference directly or indirectly references a region ρₙ of the object graph. Each reference may reference a different region of the object graph. These are represented by distinct subscripts of ρ. For the purpose of describing general reference capabilities the notation ρₙ is used to indicate that some region is referenced. This should not be taken to mean all references are to the same region ρₙ but that each reference references some possibly distinct region ρₙ.

The region of a reference describes the complete set of objects reachable through that reference. However, for the purpose of reference analysis, it acts, like a Rust lifetime, as a limit on how long the reference can be held for. That is, for memory safety, the reference must not be held beyond the time when any portion of the region ρₙ will be invalidated by deletion. Otherwise, it would be possible to create a use after free bug. Some regions of the object graph have the property that they are effectively unbounded. That is, every object in that region will either never be deleted, or its ownership is controlled by the current reference and so it can exist as long as needed by the current reference. This is very similar to the `'static` lifetime in Rust. In Azoth, it has sometimes been labeled the `$forever` lifetime. However, multiple distinct regions may be unbounded in this way. Thus, a reference constraint with the ⤳ρₙ restriction may or may not actually be bounded by some lifetime. A reference to an unbounded reference is equivalent to a reference with no lifetime restriction. However, any reference with the restriction must be assumed to have some bound for the purposes of limiting reference usage. It may just be that the caller of a function knows that the reference is actually unbounded.

The bound on a region of the object graph is the intersection of the bounds on all the objects in that region. That is, given regions ρⱼ and ρₖ that are reachable from region ρᵢ (i.e. ρᵢ⤳ρⱼ and ρᵢ⤳ρₖ), the bound of ρᵢ is the intersection of the bounds on ρⱼ and ρₖ.

It is important remember that the bound of a region may be the result of a bound either on the object directly referenced on on object reachable through references from that object. In the former case, the reference may be directly to an object owned by another reference. Its bound is then determined by the lifetime of the owning reference. In the latter case, the reference may own and thus control the lifetime of the object it references, but that object may contain a reference to a shared object owned by another reference and thus bound by its lifetime.

## Assumptions

Given the available rights and restrictions, there are potentially 2⁶∙3=192 combinations of them (each right or restriction may be present or not, except the exclusive identity right which may be present, not present, or a presence flag). To reduce this to a set of reasonable options, certain assumptions must be made. In addition, there very idea that such an analysis can get at an appropriate system of references for Azoth is built on certain assumptions. This section attempts to make those assumptions explicit.

Assumptions limiting the combinations of rights and restrictions can be summarized with a pseudo-logic notation. This notation indicates that given certain combinations of rights and restrictions, other rights and restrictions should be limited. The presence of a right is represented by its symbol, the absence of a right is represented by its symbol proceeded by a not sign "¬". Symbols not listed may or may not be present. An implication arrow "⇒" is used to indicate that given the combination of rights and restrictions on the left, the rights and restrictions should be constrained by those stated on the right.

### Compile Time Capabilities

This analysis assumes that restrictions on references must be properties enforced at compile time. While this may seem absolutely necessary, note that the use of an ownership flag is a weakening of this assumption. It may be the case that there are other properties that could be enforced at runtime which would improve the usability of Azoth while still performing acceptably.

### No Ownership Types

One kind of restriction on references which can not be entirely captured by the reference capabilities described here is ownership types. An ownership type assigned a particular object as the "owner" of every reference. The owner need not directly reference the objects it owns. However, it is generally required that all paths through the object graph to an object pass through the owning object. Ownership types may allow more flexibility in the object graphs that can be constructed. Indeed, Azoth may eventually have a form of ownership types added to it. They may enable to creation of object graphs with cycles. However, it is hoped this can be an extension to the basic reference capability system of Azoth which follows something like the approach described here.

### No Structure Lifetimes

In Rust, structs and traits can be polymorphic over lifetime variables by taking lifetime parameters. This is a source of significant complexity in Rust. This analysis omits such types from consideration. It is hoped that the capability restriction offered by a single region bound for a reference will be sufficient. This could be possible because Azoth makes more extensive use of such bounds due to its object-oriented approach. In Rust, owning references rarely carry lifetime restrictions except with the use of trait objects. In Azoth, most references including owning references will carry lifetime bounds.

If lifetime parameters to classes and traits are needed in Azoth, it is hoped that like in Rust, they can be an extension of the basic reference capabilities that enables certain advanced scenarios.

### Static Borrowing

Rust style borrowing is being implicitly assumed. That is, a parent reference has a certain capability. A child reference is created from it to the same object. The capabilities of the parent reference are then restricted by the compiler as long as the child reference exists. The compiler must be able to prove that the child reference no longer exists at a certain point in order for it to allow the parent reference to be used with full capabilities.

It is hard to conceive of other ways compile time capabilities could operate. More limited systems have not restricted the parent reference because of the child reference. However, those systems typically either allowed some form of unsafe operation or greatly restricted the kinds of child references that could be created.

### No Null Capability

**¬∅, ¬(∅⤳ρₙ)**

The null capability with the empty set of rights should not be allowed. A reference with absolutely no capabilities would never be useful. The only use I've seen for this is under the hood in a type system to for example, describe move semantics by asserting that after the move, the original variable now has no rights. Move semantics are available in Azoth, but are outside of the reference capabilities described here.

### Identity on All Reference

**I**

All references should be comparable for identity and therefore have the identity right I. There are very few suggested uses of references without the identity right. In Azoth, comparison for identity means comparing the pointer value. If one has a reference at all, they can always forcibly compare the pointers. It seems that only in a system with true unforgeable object capabilities used for security would a reference that can't be compared for identity be truly reasonable. Once one assumes all references have the identity right, then the exclusive identity right I̅ takes on greater significance. It now implies that there are no other references in the application to the same object. It thus becomes a guarantee of uniqueness.

### Single Mutable Reference or Multiple Immutable References

**W⇒RR̅W̅**
**R⇒W̅**

Concurrency safety either requires critical sections or that all data be either readonly or writeable by a single source and not readable by any others. The Rust reference capabilities embody this and use it to ensure complete concurrency and thread safety. The goal is that Azoth provide similar guarantees. As such, any reference with the read right R must also be guaranteed that no other references can be writing to it (W̅). Any reference with write must be guaranteed that there are no other references that can read or write the same object (R̅W̅). Assuming a reference can write to an object, and has exclusive read and write, there wouldn't be any reason to restrict it from reading the object as well. Thus we assume that references with the ability to write also have the ability to read.

### Identity References Don't Impose Restrictions

**¬R¬WI⇒¬R̅¬W̅¬I̅¬ξ**

A reference that allows for identity comparison only but neither read nor write can be useful. The Pony language's `tag` capability is like this. They enable identification of an object without restricting other reference's ability to use that object. For example, an identity reference can still be used to remove and object from a collection. Since identity only references need not impose exclusive read, write, or identity on other references, it is assumed that they won't. This maximizes the flexibility of identity references and simplifies the language.

### Exclusive Identity is Truly Unique

**I̅⇒R̅W̅**

Since all references have the identity right, any reference with exclusive identity must be the only reference to that object. It consequently also has exclusive read and write access to the object.

### Read-Only Non-Unique Reference Weakening

**R¬WI¬I̅¬ξ⇒¬R̅**

Given a read-only reference that definitely isn't unique, but has exclusive read, it can be weakened to a reference without exclusive read.

---

RI Notes:
* RIR̅W̅I̅ allows mutability to recovered
* RIR̅W̅ξ can accept both RIR̅W̅I̅ and RIW̅ (because of the weakening rule that kicks in if the flag is false )


---

### Writable Exclusive Read Write Unbounded is Exclusive Identity

**WR̅W̅¬⤳ρₙ⇒I̅**

If a writeable reference has exclusive read and write access, and there is no lifetime bound on it, then it must also have exclusive identity. Since this reference has both exclusive read and exclusive write, the only other reference that could exist to it is an identity only reference. If such an identity reference

However, if an identity reference existed to it, then that reference would bound the lifetime of this reference.

## Remaining Capabilities

[^Boyland]: Capabilities for Sharing: A Generalisation of Uniqueness and Read-Only (2001) John Boyland , James Noble , William Retert
