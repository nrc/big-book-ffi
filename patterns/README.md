# Patterns

## Architectural patterns

* Modular interop - a high level approach for ensuring effective interop
* [Layered library design](layered.md) - how to structure libraries and crates for interop
* -sys crate
* Wrap a C library
* Serialization
* Cross-language ownership


## Design patterns

* Foreign dtor
* Object-based API (https://rust-unofficial.github.io/patterns/patterns/ffi/export.html)
* Rust version of C object
* Something about intermediate types like CString/OsString (https://rust-unofficial.github.io/patterns/idioms/ffi/accepting-strings.html, https://rust-unofficial.github.io/patterns/idioms/ffi/passing-strings.html)
* Transparent smart pointer
* Consolidated wrapper (https://rust-unofficial.github.io/patterns/patterns/ffi/wrappers.html)
* Strings (how to actually use them, see strings links above, https://snacky.blog/en/string-ffi-rust.html, https://dev.to/kgrech/7-ways-to-pass-a-string-between-rust-and-c-4iebZ)
)

## Programming idioms and best practices

* Representing Rust errors in C (https://rust-unofficial.github.io/patterns/idioms/ffi/errors.html)
* Representing C errors in Rust

## Anti-patterns

* Disguising pointers as values (unclear, disguises unsafety)
* Using C structs directly in Rust (back compat hazards including padding, due to different back compat between C and Rust)
