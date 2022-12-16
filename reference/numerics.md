# Numeric types

For numeric types to agree across an FFI, their kind (unsigned integer, signed integer, or floating point), size, and invariants must match. The size of most C/C++ types and `usize`/`isize` in Rust can vary depending on the platform. For all numeric types, if the size matches then the alignment will also match (on a single platform).

[std::ffi](https://doc.rust-lang.org/stable/std/ffi/index.html) defines type aliases for common numeric types which are platform-accurate; [libc](https://crates.io/crates/libc) defines a few more aliases for less common types. Using these aliases is usually easier than using Rust types directly.

## Integers, booleans, and characters

### Rust integers

`u8` ... `u64` and `i8` ... `i64` are unsigned and signed respectively with the number in the type indicating the number of bits.

`usize` and `isize` are 32 bits on 32 bit platforms and 64 bits on 64 bit platforms.

### C/C++ integers

A C/C++ integer is unsigned if it uses the `unsigned` keyword and signed otherwise.

A `char` is always 8 bits, a `short` is always 16 bits, and a `long long` is always 64 bits.

The size of `int` and `long` are platform dependent, see [std::ffi::c_{int|long}_definition](https://github.com/rust-lang/rust/blob/master/library/core/src/ffi/mod.rs#L150)

### 128 bit integers

Rust supports `i128` and `u128`. These types are mostly [not safe for FFI](https://github.com/rust-lang/rust/issues/54341) (will lead to UB) and must be avoided. In particular, they are not compatible with C's 128bit integer types where those exist. However, they can be used on [non-Windows aarch64](https://github.com/rust-lang/libc/pull/2719).

### booleans

Rust (`bool`) and C's (strictly, C99 and later, `_Bool`) boolean types are compatible. Technically, C++'s `bool` is not guaranteed to be the same representation as C's `_Bool`, but they are on all known platforms, so it is safe to assume that Rust's `bool` is compatible with C++'s.

It is common to use integers to represent booleans in C programs (especially older programs or when using older toolchains). These can be converted to Rust `bool`s if the size matches and they are guaranteed to only have values `0` or `1`. (It is possible to use `0` for false and non-zero for true with C's boolean operators, however, storing any value other than `0` or `1` in a Rust `bool` is UB. You can check and convert in either Rust or C code, but in the latter case you must not use a Rust `bool` in your FFI).

### characters

Rust and C character types are incompatible.

C character types can be converted to or from Rust's 8 bit integer types. `unsigned char` is always `u8`, `signed char` is always `i8`. `char` may be either `i8` or `u8` depending on the platform, see [std::ffi::c_char_definition](https://github.com/rust-lang/rust/blob/master/library/core/src/ffi/mod.rs#L104).

A Rust `char` is a 32 bit type which must be a valid [Unicode scalar value](https://www.unicode.org/glossary/#unicode_scalar_value). It is UB to create a `char` which is not valid Unicode. You should probably avoid using `char` in FFI unless you have a custom character type with the same size and invariant in your foreign code. Otherwise it is usually better to pass numeric bytes and use helper methods on [`char`](https://doc.rust-lang.org/stable/std/primitive.char.html) to create the Rust `char`.

TODO wchar_t

### Non-zero integers

There are (currently unstable) type aliases for non-zero integers in `core::ffi`. These map to the non-zero integer types in [`core::num`](https://doc.rust-lang.org/nightly/core/num/index.html) with the correct size for the C integer types. The user must maintain the non-zero invariant (whether that is a safety issue depends on how the types are used); i.e., Rust does not ensure that values with this type are in fact non-zero.

## Floating point

A C `float` is equivalent to a Rust `f32` and a C `double` is equivalent to a Rust `f64`.

## SIMD

SIMD vectors cannot be used in FFI (UB). There is an [accepted RFC](https://rust-lang.github.io/rfcs/2574-simd-ffi.html) to address this, but it has [not been implemented](https://github.com/rust-lang/rust/issues/63068).
