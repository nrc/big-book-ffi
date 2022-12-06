# Layered library design

When wrapping a foreign library for use in Rust, consider writing a first layer in C (especially if the legacy code is C++) with an API better suited for interacting with Rust. Then have a crate which is only bindings of C code into Rust (either hand-written or auto-generated). The next layer is a crate which only has the functionality of the foreign library (i.e., no client logic), but presented in a Rust-idiomatic way. The bindings crate will be all `unsafe`, the idiomatic crate should aim to have a 100% safe API. Clients should only use the idiomatic crate and never use the bindings crate (some advanced usages may require using the bindings in unanticipated ways, however these clients should create safe abstractions of their own rather than use the bindings directly).

If following this pattern, it is common to give the idiomatic Rust crate the same name as the foreign library, and the bindings library that name with the -sys suffix, e.g., `foo` and `foo-sys`. (On the topic of naming, it is idiomatic to always avoid using an `-rs` suffix on any Rust crate: it is nearly always obvious from context that the crate is a Rust library, so `-rs` usually adds nothing).

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

When making a Rust library available to foreign code, you can adopt a similar strategy. Here, we have an idiomatic Rust crate which can be used directly by Rust users and is idiomatic and mostly safe code. There is then a Rust wrapper which is more C-like and presents an API which is more convenient to use for FFI and includes unsafe functions which make clear the invariants callers must maintain. C bindings reflect this wrapper into the C world. This can be used directly by C code, or can there can be a C/C++ wrapper which is more idiomatic (this is much more useful for C++ rather than C, since it is possible to have an idiomatic C API with the direct bindings, but that is much harder for C++). C/C++ users (again, more likely C++) then use this wrapper library rather than the bindings.

```
------------------------
 Rust crate (idiomatic)     foo
------------------------
 Rust wrapper (unsafe)      foo-ffi
------------------------
       C bindings           libfoo-ffi
------------------------
C/C++ wrapper (optional)    libfoo
------------------------
      C/C++ users
------------------------
```

There are not strong naming conventions in this direction, and the above example names are not great.

## Tooling

You can use [bindgen](TODO) to generate the bindings layer (foo-sys in the example). You can use [cxx](TODO) to generate both the C wrapper and the bindings layer, or at least parts of both.

In the other direction, you can use [cbindgen](TODO) to generate C bindings (e.g., libfoo-ffi) or [cxx](TODO) to generate both the Rust wrapper, C bindings, and C++ wrapper (although in this case the layers are not clearly defined).

## See also

* [-sys crate](TODO) - separating the Rust bindings from the idiomatic Rust wrapper - a component of this pattern,
* [Wrap a C library](TODO) - the C wrapper layer - a component of this pattern.

