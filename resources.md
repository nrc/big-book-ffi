# Resources

## Tooling

Bindgen is the most popular and mature tool and is maintained by the Rust project. It is used to create bindings for C code (and some C++) in Rust code. Cbindgen can be used to create C bindings to Rust code. The other tools below are for C++ interop; cxx is the current favourite tool with the community, but is not suitable for all use cases.

* [bindgen](https://github.com/rust-lang/rust-bindgen)
* [cbindgen](https://github.com/eqrion/cbindgen)
* [cxx](https://cxx.rs/), [repo](https://github.com/dtolnay/cxx)
* [autocxx](https://github.com/google/autocxx) (Google tool for 'integrating cxx with bindgen')
* [diplomat](https://github.com/rust-diplomat/diplomat)
* [rust-cpp](https://github.com/mystor/rust-cpp) (`cpp!` macro)
* [flapigen](https://github.com/Dushistov/flapigen-rs) (formerly Swig)
* [cxx-async](https://github.com/pcwalton/cxx-async)

You may want to use COM/WinRT for inter-language interaction, the best Rust support for COM and WinRT is in [windows-rs](https://github.com/microsoft/windows-rs/).

## Documentation

* [Nomicon chapter](https://doc.rust-lang.org/nomicon/ffi.html)
* [Unofficial FFI guide](https://michael-f-bryan.github.io/rust-ffi-guide/)
* [FFI omnibus](http://jakegoulding.com/rust-ffi-omnibus/)
* [Firefox docs for C++ interop](https://firefox-source-docs.mozilla.org/writing-rust-code/ffi.html)
* FFI [idioms](https://rust-unofficial.github.io/patterns/idioms/ffi/intro.html) and [patterns](https://rust-unofficial.github.io/patterns/patterns/ffi/intro.html)
* [Chrome docs for C++ interop](https://www.chromium.org/Home/chromium-security/memory-safety/rust-and-c-interoperability/)
* FFI chapter in [ANSSI-FR Secure Rust Guidelines](https://anssi-fr.github.io/rust-guide/07_ffi.html)

## Unsafe programming

Resources for learning about unsafe programming:

* [Chapter](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html) in The Book.
* [Nomicon](https://doc.rust-lang.org/nomicon)
* Unsafe code guidelines
  - [rendered](https://rust-lang.github.io/unsafe-code-guidelines)
  - [repo](https://github.com/rust-lang/unsafe-code-guidelines)
  - [issues](https://github.com/rust-lang/unsafe-code-guidelines/issues)
* [Ralf's thesis](https://publikationen.sulb.uni-saarland.de/handle/20.500.11880/29647)
* [Stacked borrows paper](https://plv.mpi-sws.org/rustbelt/stacked-borrows/paper.pdf)
* [GhostCell paper](http://plv.mpi-sws.org/rustbelt/ghostcell/paper.pdf)
* [Ralf's blog](https://www.ralfj.de/blog/)
* [Gankra's thesis](https://gankra.github.io/blah/papers/thesis.pdf)
* [Gankra's blog](https://gankra.github.io/blah/#articles)
* [MIRI repo](https://github.com/rust-lang/miri/)
