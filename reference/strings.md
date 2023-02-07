# Strings

TODO see string patterns, pointer reference (since C strings are pointers)

## Rust, C, and C++ strings

There are many string types in Rust and C/C++. I'll cover them here, focussing on their representations and invariants, since that is what is most important for language interop. For correct FFI, you need to understand a string's layout in memory, whether the string is nul-terminated (and whether nul characters may be embedded within the string), and the encoding of the string (e.g., UTF-8).

### Rust

Rust has three classes of string types in the standard library, each of which has owned and borrowed[^slices] types (the latter of which is usually a dynamically sized type, see [the wide pointer section](TODO)). The owned type is called a "string" and the borrowed type a "str". You could also use a sequence of characters or bytes as strings, or define your own custom string type (see the below section on Windows strings for some examples).

The standard Rust string types are [`String`](https://doc.rust-lang.org/nightly/std/string/struct.String.html) and [`str`](https://doc.rust-lang.org/nightly/std/primitive.str.html). Both are UTF-8 strings and must always be valid UTF-8. A `String` is a newtype wrapping a `Vec<u8>`; `str` is a built-in type and always has the same representation as a `[u8]`. This means that a `String` is a pointer (a unique, non-null pointer to a sequence of `u8`s, i.e., essentially a `*mut u8` in terms of representation), a capacity (`usize`), and length (`usize`), in that order. A `&str` is a wide pointer consisting of a (non-null) pointer to a sequence of `u8`s and a length (`usize`). However, the order of the components of a wide pointer is unspecified and unstable (i.e., may change in the future).

Rust has the [`std::ffi::CString`](https://doc.rust-lang.org/nightly/std/ffi/struct.CString.html) and [`std::ffi::CStr`](https://doc.rust-lang.org/nightly/std/ffi/struct.CStr.html) types for working more easily with foreign language string types. These types are *not* directly FFI compatible with C strings. These strings must be nul-terminated, have no internal nul characters, but do not have to be valid UTF-8. Use the `as_ptr` method to get a an FFI-compatible pointer. The representation of these strings is not part of their interface.

Similar to `CString`/`CStr`, [`std::ffi::OsString`](https://doc.rust-lang.org/nightly/std/ffi/struct.OsString.html) and [`std::ffi::OsStr`](https://doc.rust-lang.org/nightly/std/ffi/struct.OsStr.html) are meant to make working with foreign string types easier but are *not* directly FFI compatible with foreign strings. `OsString`/`OsStr` are easily convertible to both platform-native strings and Rust strings (`String`/`str`). Neither their representation nor whether they are valid Unicode is part of their interface. On Unix platforms, `OsStr` can be cheaply interconverted with byte slices, however, these are not nul-terminated. On Windows, `OsStr` can be losslessly converted into a UTF-16 (wide) string, however, this requires copying and processing the string data; again, the output string is not nul-terminated.

[^slices]: technically, these are just dynamically sized string types and are not intrinsically borrowed (e.g., `Box<str>` is a valid type). However, in practice these types are nearly always used with borrowed references (e.g., `&str`) to represent borrowed strings. These are often called *string slices* since they can be a slice (aka substring) of the underlying string.

### C

C strings are pointers to a nul-terminated sequence of `char`s. They may have either pointer or array types (which are equivalent in C). C strings to not have a specified encoding, that is a program is free to interpret a C string as ASCII, UTF-8, UTF-32, or any other encoding.

### C++

The C++ standard library includes the `string` type (which is actually an alias of an instantiated generic type `basic_string<char>`). Like the C string type it does not have a specified encoding. Its methods are all byte-oriented (i.e., have no concept of a character beyond a `char`). It is not directly compatible with C strings and its representation is not part of its interface. It is easy to get a C string with the `c_str` method, whether this is guaranteed to return a pointer to the data in the string or a copy of it depends on the version of C++.

### Windows

Windows uses many different string types: `HSTRING`, `BSTR`, and the `PSTR` family of types.

[`HSTRING`](https://devblogs.microsoft.com/oldnewthing/20160615-00/?p=93675) is primarily used with WinRT and is immutable. It is usually (but not always) reference counted. It is nul-terminated, but may also include embedded nuls (it stores a length so doesn't rely on nul-termination). It's UTF-16 encoded. Empty strings are represented as a null pointer.

[`BSTR`](https://learn.microsoft.com/en-us/archive/blogs/ericlippert/erics-complete-guide-to-bstr-semantics) is primarily used with COM. It is a nul-terminated, mutable, UTF-16 string which may include embedded nuls. A null pointer is a valid `BSTR` and represents the empty string, though empty `BSTR`s may also be used. `BSTR`s always work in conjunction with the system allocator (`SysAlloc*`) and the length of the string is laid out in memory preceeding the data, and a nul character comes after the data in memory; neither are included in the `BSTR`'s length. A `BSTR` is a pointer and points at the first character, not the length.

The `PSTR` family of types are 'pointer to char's, pointing to a null-terminated sequence of characters (similar to C strings). If there is a `C` in the name it is an immutable string (otherwise its mutable), if there is a `W` then the characters are wide (two bytes per character) and the string is UTF-16 encoded. If there is no `W`, then the characters are one byte and there is no specified encoding (i.e., may be ASCII or UTF-16 or whatever; these are compatible with C strings). An `L` in the name can be ignored, e.g., `PCWSTR` and `LPCWSTR` are the same type.

There are Rust bindings for these types in [windows-rs](https://docs.rs/windows/latest/windows/core/index.html) and [macros](https://docs.rs/windows/latest/windows/index.html#macros) for creating some of these string types in Rust. The type bindings are best used only for FFI: most are newtype wrappers of raw pointers, so it is very easy to create dangling pointers and other memory safety errors when using them.

Windows primarily uses UTF-16. Rust does not have UTF-16 strings in its standard library (though as mentioned above, OsString can losslessly handle UTF-16). The [widestring crate](https://docs.rs/widestring/latest/widestring/) provides types including several UTF-16 string types which can make working with Windows strings much easier.

## FFI with foreign Strings

For the actual FFI, use the Rust string type which agrees with the foreign string type (see table below).

| Foreign type | Rust type |
|--------------|-----------|
| C string `[const] char [const] *` | <code class="hljs">*{const&#124;mut} c_char</code> |
| C++ `string` | [`cxx::CxxString`](https://cxx.rs/binding/cxxstring.html) |
| `HSTRING` | [`windows::core::HSTRING`](https://docs.rs/windows/latest/windows/core/struct.HSTRING.html) |
| `BSTR` | [`windows::core::BSTR`](https://docs.rs/windows/latest/windows/core/struct.BSTR.html) |
| `PSTR`/`LPSTR` | [`windows::core::PSTR`](https://docs.rs/windows/latest/windows/core/struct.PSTR.html) |
| `PCSTR`/`LPCSTR` | [`windows::core::PCSTR`](https://docs.rs/windows/latest/windows/core/struct.PCSTR.html) |
| `PWSTR`/`LPWSTR` | [`windows::core::PWSTR`](https://docs.rs/windows/latest/windows/core/struct.PWSTR.html) |
| `PCWSTR`/`LPCWSTR` | [`windows::core::PCWSTR`](https://docs.rs/windows/latest/windows/core/struct.PCWSTR.html) |

Creating most of these strings in Rust is usually possible via some macro or conversion function.

The more interesting question is when and how to convert between the FFI-specific types and more standard Rust types (and which types to use). That is out of scope for the reference, but see [TODO patterns](TODO).

### Memory management

The usual rules of [memory management with FFI](TODO) apply: memory must be released in the same language it was allocated, and using borrowed data is easier.

## FFI with Rust Strings

It is possible to pass Rust strings across FFI to foreign functions. However, if you are designing an API, it is usually easier to use foreign strings in the FFI and convert these to and from Rust strings internally in Rust code.

If you manipulate the contents of the strings (either in foreign code or unsafe Rust code), then you must respect both the usual invariants around [pointers](TODO), and Rust's string invariants (from [`String` docs](https://doc.rust-lang.org/nightly/std/string/struct.String.html#method.from_raw_parts)):

* the memory must have be allocated by the same allocator the standard library uses, with a required alignment of exactly 1,
* the `length` of the string must be less than or equal to its `capacity`,
* the `capacity` of the string must be the correct size of the allocation,
* the first `length` bytes of the string must be valid UTF-8.

Note that if you are using the string types in Rust functions with foreign bindings, then you must establish these invariants in the foreign code. Doing so in the Rust code is likely to be unsound.

To pass a Rust string to C++, you can use Cxx's bindings for [`String`](https://cxx.rs/binding/string.html) or [`&str`](https://cxx.rs/binding/str.html).

To pass a Rust string to C, you can use a struct with the correct layout (you could look at the standard library source code, or just use the Cxx bindings as a reference).

### Memory management

The easiest scenario is to create a `String` in Rust, pass a borrowed `&str` to foreign code and ensure that the foreign code does not store the pointer, pass it to another thread, call its destructor, or deallocate it.

If you must store the string in foreign code, then you must pass the owned type `String`. In this case, you must ensure the pointer remains unique (in particular, you must not keep a reference in the Rust code) and pass it back to Rust for destruction.

If you allocate memory for the string in foreign code, then you must not run its destructor in Rust, and you must pass the string back to foreign code for destruction. The easiest way to do that is to pass `&str` to Rust. If you must pass `String` (or a raw pointer used to produce a `String` in Rust code), then you must ensure that there is no copy of the pointer kept in foreign code, and that the pointer is returned to foreign code for destruction. Using a custom reference counted type might be a better alternative, see [TODO pattern](TODO).
