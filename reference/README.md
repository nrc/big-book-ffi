# Reference

This section is designed as a reference and you probably don't want to read it end to end. It is primarily aimed at those implementing and designing tools and low-level libraries, or users who need to do unusual and/or low-level interop work. Hopefully, if you're doing common integration work you mostly won't need this level of detail.

* [Functions and Methods](functions.md)
* Data types (TODO cover agreement, not agreeing means UB)
  - [Numeric types](numerics.md)
  - [Strings](TODO)
  - [Pointers](TODO) void pointers
  - structs, enums, unions
  - classes

## Linking

extern blocks

`#[link(...)]` attribute

## statics and consts

[`used` attribute](https://doc.rust-lang.org/reference/abi.html#the-used-attribute). Using the `no_mangle` attribute implicitly implies `used`. Use `extern` for external linkage
