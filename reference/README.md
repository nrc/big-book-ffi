# Reference

This section is designed as a reference and you probably don't want to read it end to end. It is primarily aimed at those implementing and designing tools and low-level libraries, or users who need to do unusual and/or low-level interop work. Hopefully, if you're doing common integration work you mostly won't need this level of detail.

TODO mechanics vs safety
  I think this is the same as FFI types vs idiomatic types

TODO semi-opinionated
  some stuff is still undecided, but we describe the current state of the art
  try to cover different points of view and where there are differences, but not all

TODO assumes C/C++

* [Functions and Methods](functions.md)
* statics and consts - TODO [`used` attribute](https://doc.rust-lang.org/reference/abi.html#the-used-attribute). Using the `no_mangle` attribute implicitly implies `used`. Use `extern` for external linkage
* [FFI types]()
* [Idiomatic types](data-types.md)
  - [Numeric types](numerics.md)
  - [Strings](strings.md)
  - [Pointers, references, and arrays](TODO) void pointers, fat pointers, const, arrays and slices, null/non-null, single allocation, no pointers into middle of an object, ZSTs, pointers to deallocated (e.g., dangling) mem, invalid metadata in wide pointers
  - structs, tuples, and unions
  - enums
  - properties - send, sync, eq, hash, etc.
  - classes? trait objects?

## Linking

building C and Rust such that object files can be found, etc.

extern blocks

`#[link(...)]` attribute

## Safety and validity

Rust has several useful safety invariants. These are guaranteed to hold (by the compiler and by authors of unsafe code) in most Rust code, but they may be temporarily broken in code which is marked `unsafe`. Unsafe code does not permit writing code which is not safe, but rather indicates the author of the code is responsible for safety, rather than the compiler.

Since all foreign code is outside of the remit of the Rust compiler, all foreign code is unsafe and must be called from within an `unsafe` block or function. When writing an FFI layer, we usually want to present a safe API to Rust code, therefore the FFI layer must establish Rust's safety invariants and the safety invariants of any types in its API.

Precisely where safety invariants must hold is still being decided by the Rust project. The current preferred proposal is that the *unsafe boundary* is the *public API* of a *module* (including its sub-modules). Note that this is not defined in terms of unsafe blocks or functions! In more detail, for any public (i.e., visible from an outside module, not necessarily `pub`, `pub(crate)`, `pub(super)`, etc. would count if there is an enclosing module in the crate) function (or method) of a module, there is a set of safety requirements. For safe functions, this set is empty, for `unsafe` functions, this set should be documented. If all these requirements are satisfied, then calling the function will never result in a memory safety error occurring. During the function call, safety invariants (both Rust's and the module's) may be violated, but they must be re-established (perhaps relying on the function's safety requirements) by the time the function returns (or unwinds).

Rust also has validity invariants. These must hold in all Rust code both safe and unsafe (otherwise it is *undefined behaviour*). Validity invariants do not need to hold in foreign code, but must be re-established before control is returned to Rust (i.e., before crossing the FFI boundary). Contrast this with safety invariants which may be violated in foreign code and Rust code but must be re-established before crossing the unsafety boundary.

TODO safety and validity are per type and so establishing invariants might occur when data is reinterpreted.

### Example 

TODO passing a pointer and length from C code and using it as an array

C code:

```C
void do_thing(char* arr, int c_arr) {}


extern void do_thing_impl(char* arr, int c_arr);
```

Rust code:

```rust
pub unsafe fn do_thing_impl(arr: *mut u8, c_arr: usize) {}

fn do_thing_idiomatic(arr: &mut [u8]) {
    // regular Rust code
}
```
