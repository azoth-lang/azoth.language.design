# Memory Allocation

## `deallocate()` Arguments

The C style memory allocation function `free()` takes only a pointer to the memory to be
deallocated. Theoretically, if additional information could be passed, the memory allocator could be
optimized. In particular, if the size of memory requested was passed, that could prevent the memory
allocator from needing to store that information. Rust went even further with there memory
allocation API and required that the exact size and alignment used when allocating the memory be
passed when deallocating. However, they seem to be ignoring issues where the memory size isn't
correctly known. They give an example of someone allocating extra memory and constructing a `Vec`
from it. The `Vec` would then attempt to deallocate the memory using the alignment it would have
used had it been the one to allocate it. That may not match the value actually used. Adding an
alignment value to every `Vec` seems like a bad idea. However, there are cases when even the size
won't be well known. Imagine consuming a list to construct an array. In that case, one might want to
not reallocate the memory of the list, but simply reuse it as the memory for the array. But since
the capacity of the list could be bigger than the size of the array, the array type would not know
the proper size to deallocate the memory with. For these reasons it seems onerous to require the
deallocate operation to require parameters besides the pointer to the memory to be deallocated.
Given that Azoth is a high level language and that this was good enough for C it makes sense to go
with that and not spend further time on it. The same argument applies to not requiring the previous
size or alignment when reallocating.
