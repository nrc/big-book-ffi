# The mechanics of FFI

No runtime, bare metal means low-level interop is free

where the ABI of Rust and foreign lang coincide, interop is free. Where it doesn't we require abstraction layer work
    stable and de facto stable Rust ABI

function calls and data must agree

runtime behaviour
    async/concurrency
    unwinding

## Declaring and defining functions

For a function to be called across languages, it must be declared in both languages (but only defined in one) so that both compilers can find it. The two declarations must end up associated with the same definition, and this requires linking object files correctly, see the chapter on [building and linking](TODO) for details. If declared correctly, calling the function requires no special effort and no runtime overhead (it can even be inlined across languages if using link-time optimisation (LTO)).

TODO dyn linking
TODO calling is unsafe

Global variables can also be accessed across the FFI boundary, see the [reference](TODO) chapter.

### Function defined in Rust

This section will cover defining a Rust function which can be called from a foreign language.

For a Rust function to be callable from a foreign language you must use the `extern` keyword to specify the ABI. `"C"` is the default and most common option, see the reference for a full list. If the function should be name-able from outside Rust, you should use the `#[no_mangle]` attribute to prevent name mangling. You'll usually need this, unless the function is only used via a function pointer.

E.g.,

```rust
#[no_mangle]
pub extern "C" fn foo() -> i32 {
    // ...
}
```

You'll need to declare the function in foreign code to use it. In C/C++ this will look the same as declaring a C function (usually declared in headers as required). E.g.,

```C
int foo();
```

### Function defined in C/C++

This section will cover defining a C/C++ function which can be called from Rust.

A function defined in C/C++ will be just a regular function, just don't declare it `static` (the default is `extern`, which is what we want).

In Rust the function must be declared. This is done inside an `extern block`. Functions declared in an `extern` block may not have bodies, are implicitly unsafe to call, and have a specified ABI (`"C"` by default). Various attributes control how function definitions are discovered and linked, see the [reference](TODO).

E.g.,

```rust
extern "C" {
    fn foo() -> i32;
}
```

## Data

When data is passed across the FFI boundary, its type must be known on both sides (i.e., there must be declarations in both languages) and those types must be compatible. Compatibility involves both a type's representation (how it is laid out in memory) and invariants of the type (due to either the language or the type itself). Compatibility is not quite a symmetric relation because while the representations must match, the invariants must be the same or stronger on the callee side than the caller side. Where representations are compatible but invariants are not, the missing invariants can be made requirements for the caller to satisfy. Compatibility is also platform-dependent since different syntactic types may have different representations on different platforms.

Type compatibility is a big topic. We'll summarise here and give a complete description in the [reference](TODO).

[std::ffi](https://doc.rust-lang.org/stable/std/ffi/index.html) defines type aliases for common numeric types which are platform-accurate; [libc](https://crates.io/crates/libc) defines a few more aliases for less common types. Using these aliases is usually easier than using Rust types directly.

Primitive types (integers, floating point types, and booleans) are straightforwardly compatible, though the names are different in C and Rust and the correspondence is platform-dependent (using `std::ffi` in Rust is the easiest solution). Characters are also a primitive type in C and Rust, but have different sizes and encodings in the different languages. C `char`s are compatible with Rust's `i8` or `u8` . A Rust `char` is compatible with a C `unsigned int` or `unsigned long`, however, a Rust `char` must always be a valid [Unicode scalar value](https://www.unicode.org/glossary/#unicode_scalar_value), this requirement must be satisfied in the C code before the data has the Rust `char` type.

If `T_rust` and `T_c` correspond, then `*const T_rust` and `const T_c*` correspond, and `*mut T_rust` and `T_c*` correspond. When calling foreign functions from Rust code, you can use `&mut T_rust` and `&T_rust`, respectively in the Rust function declaration (Rust references and raw pointers have the same representation). You can also use `Option<&mut T_rust>` and `Option<&T_rust>`, see the discussion on enums, below. You cannot do this in the opposite direction (i.e., use a Rust reference in a Rust function definition and a pointer in the C declaration), since the Rust reference types have more invariants which must be satisfied. (You technically can do this without any compile-time errors and require the additional invariants in the foreign code, however, these invariants are difficult to guarantee and failing to do so will cause undefined behaviour).

TODO ^ the pointees don't have to correspond fully. Box, Rc, etc.

C and Rust structs are compatible if they have corresponding fields and the field types are compatible (and the Rust struct is `repr(C)`). Sometimes a C struct may have hidden fields (sometimes called an opaque struct). Such types can be represented in Rust as a zero-sized type with a private field and no constructor function (so the type cannot be instantiated), you can include a field with type `PhantomData<(*mut u8, PhantomPinned)>` to prevent the type implementing the `Send`, `Sync`, or `Unpin` auto-traits. See the [opaque struct pattern](TODO) for more.

C and Rust unions are compatible if the the Rust union is `repr(C)` and their fields are compatible. Field compatibility is a bit more complex than for structs: the unions must end up the same size and any pair of fields which are used together must be compatible. It is up to the programmer to ensure compatible usage of the unions.

Rust enums with no embedded data which are `repr(C)` are compatible with C++ enums if they have the same number of variants, the specified (or implicit) determinant types match, and the values of all variants match. It is possible to loosen these restrictions slightly. It is *invalid* for an enum to have a value which is not a declared variant. So as long as there is a Rust variant which matches any C++ variant which may be passed across the FFI boundary, the enums are compatible.

Enums which are 'option-like' (i.e., have two variants, one with a single field and one with no embedded data) and where the payload type is a non-null pointer type are compatible with the corresponding (nullable) pointer type in C/C++. E.g., `Option<&Foo>` is compatible with `const Foo*` (subject to the restrictions described above for pointer compatibility). Similarly an option-like enum with the [non-zero numeric types](https://doc.rust-lang.org/nightly/std/num/index.html) is compatible with the (zero-able) numeric type in C/C++.

Most tuple types are incompatible with C/C++ types because their representation cannot be specified. The exception is the empty tuple, `()` which is compatible with the `void` type in C/C++ when passed by pointer.

In general, zero-sized types are incompatible with foreign types.

C strings are pointers to `char`s and as expected are compatible with `*mut u8` or `*mut i8`. Despite their resident module, `std::ffi::CStr/CString` should not be used directly for FFI because their representations are not guaranteed.


TODO C++ types

