# Data Types

Data in both Rust and C/C++ is just ones and zeros in the computer's memory. These ones and zeros are independent of the language which generated them and there is no intrinsic sense of 'compatibility'. However, when data is used, the compiler must have semantics for those ones and zeros which in Rust, C, and C++ is determined by the types of the data. When data is passed across the FFI boundary, two compilers are involved and each must have a type defined in its own language with which to understand the data. For this to produce correct results, corresponding types in the two languages must *agree*. An abstract example, if a function `f` is declared in both Rust and C and has a single argument with type `T_Rust` in Rust and `T_C` in C, then `T_Rust` and `T_C` must agree. If they do not then any operation on the data will be undefined behaviour.

This concept of agreement goes beyond what the bytes represent (e.g., that some sequence of four bytes should be interpreted as a little-endian, 32 bit, unsigned integer) and includes the safety and validity invariants of the type. These invariants may be due to rules of the language, or due to the specific type itself. What makes this difficult is that invariants due to the language may be specified in a reference or spec, but may just be assumed by the compiler authors and otherwise undocumented. Invariants due to a specific type may be documented but, especially if they are invariants which users would not usually need to be aware of or are considered implementation details, may not be documented (or only documented in the source code). These may still be a concern when writing interop code since C/C++ allows treating data in ways usually forbidden in Rust.

TODO safety and validity invariants - https://www.ralfj.de/blog/2018/08/22/two-kinds-of-invariants.html

Invariants due to specific data types can be found in their documentation or source code. Invariants due to the kind of data types can be found below and in sub-chapters. Rust also has some invariants which apply to all data (or nearly all data), we'll cover those in the next few paragraphs.

Much of Rust's ABI and invariants are de jure undefined and may be subject to change. There have been several RFCs which cover this kind of thing, and work is ongoing within the Rust project and in academia to better specify language-wide invariants. However, there is a large body of code which works today and is unlikely to be broken, so much of this stuff is de facto standardized.


TODO what does this all mean for doing FFI?

TODO there is a two step process c_type -> binding_type -> rust_type, the C and binding types must agree, binding type typically has minimal requirements, binding type to rust type is a pure Rust conversion but to be valid we must establish the invariants of the rust type which may have to be done in foreign code. E.g., `int* -> *mut i32 -> &mut i32` equivalence of `int*` and `*mut i32` is trivial (only wrinkle is the int types), but for the conversion from `*mut i32` to `&mut i32`, we must guarantee that the pointer is unique, which depends on what is happening in the foreign code.
    - let's break the whole chapter up along these lines

### Uniqueness and mutability

In Rust, data is immutable by default. Data must be known to be mutable from its type to be mutated (TODO phrasing). It is undefined behaviour to convert immutable data into mutable data, or to directly mutate immutable data (contrast this to C, where const-ness can be cast away).

Data can be mutated only if it is known to be unique, i.e., data cannot be accessed other than via the reference used to mutate the data. Such uniqueness may be established either statically (e.g., references `&T` and `&mut T`) or dynamically (e.g., `RefCell`). All dynamic tracking of uniqueness must use `unsafe` and raw pointers code at some level, usually wrapped so that end users only see safe abstractions.

References and values can only be mutated if they are declared as mutable (e.g., a `&mut T` can be mutated and a `&T` can never be mutated) and the compiler can prove they are unique at the point mutation occurs (e.g., a `&mut T` cannot be mutated if there exists a live `&T` referencing the same value). Raw pointers have the former constraint but not the latter. A `*mut T` pointer can always be dereferenced (an `unsafe` operation) and mutated. The programmer must ensure that when the pointer is dereferenced, TODO undefined?

An important aspect of Rust for preserving uniqueness is *move semantics*. When data is passed from one location to another (e.g., assigned to a variable or passed to a function), it is logically moved (you can think of this as a bitwise copy, then deleting the old copy, although the compiler may optimise that). That means that if data is unique before being passed, it is also unique after being passed. Compare this to C/C++ where data is copied or Java-like languages where passing most data implicitly passes a reference.

Some data in Rust is copied rather than moved. Primitive types, immutable references, raw pointers, and any data structure which implements the `Copy` marker trait are copied rather than moved.

Note than both moving and copying are simple, bitwise operations. Neither invokes a constructor or destructor (in contrast to C++).

If passing an object from Rust to C/C++, care must be taken around uniqueness. If passing by reference, then pointers/references in C/C++ are copied and so will not be unique. If passing by value, then data will be copied, not moved. Therefore, Rust data which implements `Copy` can be safely passed by value to C/C++ and passed around or stored. Immutable references/pointers can also be safely passed to C/C++ as long as they are never mutated or data is mutated through them. If non-`Copy` data is to be passed by value, or data is passed by reference and is mutated, more care must be taken around these invariants.

Invariants around pointer and reference types are covered in detail in the chapter on [pointers, references, and arrays](TODO).


TODO
    raw mut pointers
    requirement in foreign/unsafe code (interior mutability)
        UnsafeCell
    scope of requirement (data referenced from Rust? Allocated in Rust?)
        validity and safety invariants

### Borrowing

TODO
    uniqueness
    mutable and immutable borrows
        overlapping borrows
    lifetimes
        lifetime due to scope - presence of dtor/attribute? and NLL
            drop check and phantomdata
    storing data
    'static borrows
    variance

### Initialization

In Rust, all memory must always be initialized unless explicitly marked using [`MaybeUninit`](https://doc.rust-lang.org/nightly/std/mem/union.MaybeUninit.html). For most data, it should be ensured that data is initialized on the foreign side of the FFI boundary. If data may not be initialized, then the Rust type must be `MaybeUninit` (e.g., if passing a `Foo`, then the Rust type must be `MaybeUninit<Foo>`). See also the discussion on `null` in the section below on `pointers and references`.

### concurrency

TODO
    effect on uniqueness
    send/sync
    unsafe and dynamic guarantees (arc, mutex, scoped threads)
    thread-safety and FFI
        Rust stuff
        thread-safety guarantees from C/C++
    atomic/non-atomic access to shared memory (even volatile ops)
        memory model is C++20 (nomicon atomics link)

### Layout and alignment

TODO
    alignment and storage address
    size = multiple of alignment
    bounds/OOB access
    can't assume layout without repr, see structs and enums
    DSTs
    ZSTs

### Platform-specific invariants

CHERI
WASM - function/data pointers
ARM?
padding bits
two's compliment

## Kinds of data type

Rust and C/C++ have many different kinds of data type. These include primitive data, compound data (enums and structs, etc.), pointers, and more. For data to agree across the FFI boundary the kind of data type must correspond (and then the details must agree, which will be covered in the following chapters).

TODO what goes here vs in the sub-chapters?

### Primitive types

Primitive types are numeric (signed and unsigned integers, and floating point numbers), characters (but not strings), or booleans. These types have the same semantics and interpretation in C/C++ and in Rust. In particular, they are always passed by simple copying (i.e., without invoking a constructor, nor moved). The names of types and some details of their interpretation varies between C/C++ and Rust, see the chapter on [numeric types](numerics.md) for details. In particular, the names of types in both C/C++ and Rust can vary depending on the platform.

Because of the matching semantics and lack of aliasing, using these types for interop is usually very simple and efficient.

Both C and Rust have a void type: `void` in C and `()` in Rust (or can be implicit in both languages). These types trivially agree. Most zero-sized types cannot be used for interop, `()` is an exception when used as a return type, but cannot be used as a type parameter. For void pointers, see the pointers and references section, below.

### Compound data

Compound data types are structs, unions, and enums in C/C++ and Rust, tuples and tuple structs in Rust, and classes in C++. Structs, unions, and some enums basically correspond between C/C++ and Rust, see the following sub-chapters for details. Tuples in Rust cannot be used in FFI because they always have the default representation (see below and the chapter on [structs, tuples, and unions](TODO)). Tuple structs correspond with foreign structs. Classes in C++ correspond with structs in Rust (although this is a complex correspondence), see the chapter on [classes](TODO).

Individual compound data types are likely to have their own invariants which will need to be maintained in foreign code (or by Rust code for foreign types).

By default, the Rust compiler can layout data however it likes and this can change between compiler versions (or even for with the same compiler version, in theory). This is incompatible with FFI, and so you must specify an alternative representation for data types for them to agree with a foreign type. We'll cover this in detail in the following sub-chapters.

Aliases (`typedef` in C++ or `type` in Rust) are present only at compile time and do not affect the representation or the invariants of the data. Rust's 'newtypes' (usually a tuple struct with a single field) are not aliases and have the same behaviour as other compound types, i.e., can introduce new invariants and may have a different representation (unless explicitly specified), compared with the underlying data.

Strings, smart pointers, and array-like collections (e.g., `Vec` in Rust) are all compound data types in both Rust and C/C++. In principle, these do not require any special treatment over other user types. However, they are more likely to have important invariants which must be maintained for the sake of soundness. Several examples will be covered in the following sub-chapters.

### pointers and references

TODO

layout - same as C
    DSTs/wide pointers
validity: non-null, dangling, unaligned, aliasing (if &mut), pointed-to value is valid (safe?)
smart pointers
void pointers

### arrays and slices

TODO

### trait objects and class objects, and methods

TODO

## Generic types

TODO
