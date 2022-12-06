# Patterns

## Architectural patterns

When wrapping a foreign library for use in Rust, consider writing a first layer in C (especially if the legacy code is C++) with an API better suited for interacting with Rust. Then have a crate which is only bindings of C code into Rust (either hand-written or auto-generated). The next layer is a crate which only has the functionality of the foreign library (i.e., no client logic), but presented in a Rust-idiomatic way. The bindings crate will be all `unsafe`, the idiomatic crate should aim to have a 100% safe API. Clients should only use the idiomatic crate and never use the bindings crate (some advanced usages may require using the bindings in unanticipated ways, however these clients should create safe abstractions of their own rather than use the bindings directly). If following this pattern, it is common to give the idiomatic Rust crate the same name as the foreign library, and the bindings library the same name with the -sys suffix, e.g., `foo` and `foo-sys`. (On the topic of naming, it is idiomatic to always avoid using an `-rs` suffix on any Rust crate: it is nearly always obvious from context that the crate is a Rust library, so `-rs` usually adds nothing).

```
------------------------
     C/C++ library          libfoo
------------------------
       C wrapper            libfoo-ffi
------------------------
 Rust bindings (unsafe)     foo-sys
------------------------
Rust wrapper (idiomatic)    foo
------------------------
      Rust users
------------------------
```

* Modular interop
* Layered library design
* -sys crate
* Wrap a C library
* Serialization
* Cross-language ownership

## Design patterns

* Foreign dtor
* Object-based API (https://rust-unofficial.github.io/patterns/patterns/ffi/export.html)
* Rust version of C object (e.g., CString (https://rust-unofficial.github.io/patterns/idioms/ffi/accepting-strings.html, https://rust-unofficial.github.io/patterns/idioms/ffi/passing-strings.html))
* Transparent smart pointer
* Consolidated wrapper (https://rust-unofficial.github.io/patterns/patterns/ffi/wrappers.html)

## Programming idioms and best practices

* Representing Rust errors in C (https://rust-unofficial.github.io/patterns/idioms/ffi/errors.html)
* Representing C errors in Rust

## Anti-patterns


