# Reference

This section is designed as a reference and you probably don't want to read it end to end. It is primarily aimed at those implementing and designing tools and low-level libraries, or users who need to do unusual and/or low-level interop work. Hopefully, if you're doing common integration work you mostly won't need this level of detail.

TODO assumes C/C++

* [Functions and Methods](functions.md)
* statics and consts - TODO [`used` attribute](https://doc.rust-lang.org/reference/abi.html#the-used-attribute). Using the `no_mangle` attribute implicitly implies `used`. Use `extern` for external linkage
* [Data types](data-types.md)
  - [Numeric types](numerics.md)
  - [Strings](strings.md)
  - [Pointers, references, and arrays](TODO) void pointers, fat pointers, const, arrays and slices, null/non-null, single allocation, no pointers into middle of an object, ZSTs, pointers to deallocated (e.g., dangling) mem, invalid metadata in wide pointers
  - structs, tuples, and unions
  - enums
  - properties - send, sync, eq, hash, etc.
  - classes? trait objects?

## Linking

extern blocks

`#[link(...)]` attribute
