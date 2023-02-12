# Functions and Methods

Interoperation of functions (including function-like things such as methods and closures) means calling Rust functions from a foreign language or foreign functions from Rust. This means declaring a function in Rust and defining it in C or vice versa. The definition and declaration must *agree* or there will be undefined behaviour at runtime. The definition must also be discoverable by code in the other language (this is partly an aspect of linking, described [previously](README.md#Linking) and partly an aspect of the function definition). This section describes what it means for function declarations and definitions to agree across languages, and how Rust functions must be defined in order to be discoverable by foreign code.

## Functions

### Definition

TODO

* visibility

#### Extern blocks

TODO https://doc.rust-lang.org/reference/items/external-blocks.html

[link attribute](https://doc.rust-lang.org/reference/items/external-blocks.html#the-link-attribute)
ABI
implicit unsafe
see also statics

#### Name

The names of functions (and other items) are [mangled](https://rust-lang.github.io/rfcs/2603-rust-symbol-name-mangling-v0.html) by the compiler by default. Name mangling means that the name of the symbol in the compiled binary is not the same as the name in the source code. Name mangling is not stable, and you should not rely on mangled names being the same between compiler versions.

Use the [`no_mangle` attribute](https://doc.rust-lang.org/reference/abi.html#the-no_mangle-attribute) to prevent name mangling of a function's name. E.g.,

```rust
#[no_mangle]
pub extern "C" foo() {}
```

If you will call a function from foreign code by name then you must use `no_mangle` (not doing so may cause linking errors or may cause incorrect runtime behaviour). If you will only call a function via a function pointer, then you don't need to.

Alternatively, you can use the [`export_name` attribute](https://doc.rust-lang.org/reference/abi.html#the-export_name-attribute) to explicitly specify the name to use for the exported symbol. E.g.,

```rust
#[export_name = "bar"]
pub extern "C" foo() {}
```

This function can be called using `foo` from Rust, and `bar` from C.

Likewise, by default C++ will mangle function names. This is inconsistent between platforms and compilers, so it is not advisable to use the mangled names (this can be done if absolutely necessary and tools like bindgen and cxx can help with this TODO is this true?). To prevent name mangling, define functions in an `extern "C"` block.

You can specify which section of the binary the function is placed in using the [`link_section` attribute](https://doc.rust-lang.org/reference/abi.html#the-link_section-attribute).

#### Calling convention

The [`extern`](https://doc.rust-lang.org/reference/items/functions.html#extern-function-qualifier) keyword is used on function definitions to specify the [calling convention](https://doc.rust-lang.org/nomicon/ffi.html#foreign-calling-conventions) (aka the ABI) used to call them (and on extern blocks to define the calling convention used to call the foreign functions declared inside, see above). The syntax is `extern "ABI"` where `ABI` is the optional [ABI identifier string](https://doc.rust-lang.org/reference/items/external-blocks.html#abi). E.g.,

```rust
pub extern "C" foo() {}
```

If no ABI identifier is supplied, then `C` is used. If `extern` is not used at all, then `Rust` is used.

The calling convention used in the declaration and definition of functions must match. This is likely to entail a somewhat complex interaction of defaults across different platforms and languages, and attributes in different languages. If you have control of both sides of the FFI, then making both `extern "C"` (either explicitly or by default) is probably the easiest option. You'll need to use other options if you want to match a calling convention in a library which cannot be changed.

The platform independent ABI identifier strings are:

* `Rust`: Rust's ABI; this is unstable and should not be used for FFI code,
* `C`: the default C calling convention,
* `system`: the platform default calling convention for calling 'system functions'. Usually the same as extern "C", except on Win32, in which case it's "stdcall".

The platform-specific ABI identifier strings are:

* `cdecl`: for x86_32 C code,
* `stdcall`: for the Win32 API on x86_32,
* `win64`: for C code on x86_64 Windows,
* `sysv64`: for C code on non-Windows x86_64,
* `aapcs`: for ARM,
* `fastcall`: corresponds to MSVC's `__fastcall` and GCC and clang's `__attribute__((fastcall))`,
* `vectorcall`: corresponds to MSVC's `__vectorcall` and clang's `__attribute__((vectorcall))`.

You might also come across `rust-intrinsic`, `rust-call`, and `platform-intrinsic`. These are used by the compiler and standard library, but you shouldn't use them in user code.

There also exist `-unwind` versions of the ABI identifier strings, e.g., `C-unwind`. These are all [unstable](https://github.com/rust-lang/rust/issues/74990), see the section below on unwinding for more details.

TODO `thiscall` is [unstable](https://github.com/rust-lang/rust/issues/42202), see discussion on methods

#### C/C++ linkage

C/C++ functions must have external linkage (this is the default, i.e., functions may not be marked `static`).

#### Signature

The types of all arguments in the function and its return type as written in the declaration and definition must *agree*. For more on type agreement, see the sections on [data types](TODO). The names of arguments do not need to match. In Rust declarations of foreign functions, `_` may be used instead of an argument name. No other patterns may be used in arguments. Patterns may be used instead of names in the usual way for Rust functions which are exported; the foreign declarations should use a name instead of a pattern.

The number of arguments in definition and declaration must match, including variadic arguments. Declarations (but not definitions) in Rust may be [variadic](https://doc.rust-lang.org/reference/items/external-blocks.html#variadic-functions) (to match variadic functions defined in C). E.g.,

```rust
extern "C" {
    fn foo(format: *const u8, args: ...);
}
```

If a function diverges, then in Rust it should have the `!` return type. In C/C++ the function should have a 'no return' attribute (`__attribute__((noreturn))`, `[[noreturn]]`, `[[__noreturn__]]`, `[[_Noreturn]]`, etc. depending on the language, version, and compiler).

TODO what if sigs don't agree?

If the return type must be used, then the Rust function should have the `#[must_use]` attribute and the C/C++ function the `__attribute__((warn_unused_result))` attribute. TODO [[nodiscard]] on ctors. Getting this wrong will lead to missing warnings which may in turn lead to runtime errors.

### Function calls

TODO

calling convention (should work)
calling variadics (just works)
see also data
unsafe

### const functions

TODO

## Unwinding

TODO

TODO -unwind ABIs

## Exceptions

TODO

## Closures

TODO

## Methods

TODO

* Virtual/static dispatch
* ctors
* dtors
* operator overloading

## Other

TODO

* async
* generators
* templates/generics
