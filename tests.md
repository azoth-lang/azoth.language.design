# Tests

Testing has become so important that it makes sense to include it as a language feature. Rust has
demonstrated the utility of having tests in the same project as the code they are testing and how
that allows tests to have additional access to that code. However, there are issues with the Rust
implementation.

## Principals

* It should be possible to compile code but not tests (though the default should be to compile
  both).
* There must be a way for tests in one package to provide functionality for tests in another
  package.
* Tests should not be in the same file.
* Building both code and tests should produce separate binaries for each.
* It must be possible for tests to have their own dependencies.
* Tests should be published with a package so that they can be run to verify behavior when a package
  dependency is updated.

## Design

The design that fits the above principals is having separate test code files and package facets.

### Test Code Files

Test code files are distinguished by the extension `.azt`. By convention the tests for a class or
function in `File.az` are placed in `tests/File.azt`. This gives them access to the `protected`
members of the namespace `File.az` is in.

### Facets

Each package has multiple "facets" a facet is a sort of aspect of that package. The facets are:

* Main
* Tests
* Examples

Tests and examples being facets allows them to access unpublished members of the `Main` facet. At
the same time, being distinct facets means that their binaries are not included in the main facet
binary.

#### References

When a package references another, only the main facet is visible to the main facet of the other.
However, the test facet has access to the published members of both the main *and* tests facet. The
examples facet has access only to the main facet of the other.

In this respect facets act almost as if they were separate packages with references between them as
determined by the above rules.
