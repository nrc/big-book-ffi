# Introduction

Interoperation with other languages is much easier in Rust than in most languages. However, it is often still the most difficult part of adopting Rust (especially if you need to interoperate with C++ rather than just C). Cross-language interop is often generically termed FFI (which literally means foreign function interface, but is used to mean all aspects of interop).

The good news is that interop is extremely low cost (interop with C is cheap and interop with other languages is no more expensive than interop with C) and the fundamentals (ABI compatibility of many datatypes and functions, extern declarations, etc.) are built in to Rust. In many cases, there is no need for data marshalling or serialization, or adaptation for calling conventions, etc. Calling a Rust function from C or vice versa is no more expensive than calling a C function in a different library (and since LTO works across the language boundary, it can even be as cheap as a within the same file).

Generally speaking, Rust is ABI-compatible with C. That means that Rust can interoperate with any language which can interoperate with C (though FFI with languages other than C is likely to be more complex and to have some runtime overhead). There is community support for interop with C++, Ruby, Javascript, and Python. Interop with .net and Java is supported via P/Invoke and JNI respectively.

There are three challenges with Rust and foreign language interop: building the project, ensuring mechanical compatibility between types and functions, and ensuring that Rust's safety invariants are upheld. All FFI calls are unsafe in Rust. To ensure correctness, invariants around ownership and uniqueness, thread safety, and panics must be ensured at the FFI boundary.

TODO

* mechanics vs safety (FFI types vs idiomatic types)
* levels of abstraction (making FFI ergonomic/idiomatic)
* ffi and safety, and what is the challenge to writing an FFI layer
* differences with other interop (managed langs, etc)
* assessing the feasibility/difficulty of interop

## Architecture

The more well-defined the boundary between Rust and foreign code, the easier things will be. At the limit, if your Rust and foreign code can live in different processes (i.e., are different programs compiled separately) and can communicate via some form of IPC then you won't have to worry about a lot of the issues with interop at all! Rust has great support for serialization/deserialization, gRPC, and other IPC/RPC technologies which can facilitate this.

If you need Rust and foreign code in the same process, then they should be used in separate 'components' of your design. Do not attempt to have Rust and foreign code interoperate in a fine-grained way within a single component. If you are migrating from another language to Rust, plan the migration on a per-component rather than a per-file basis. It is worth putting some up-front effort into designing the API of these components and the language boundary. As well as the usual API design issues, making the API coarse-grained (i.e., avoiding many calls), using simple datatypes (the more C-like, the better) with simple invariants, and avoiding bidirectional interaction will make FFI issues simpler.

Using a generic FFI option, such as COM/WinRT, is a good option if components can be separate to this extent. You will still have to consider safety issues, but the mechanical issues of corresponding types, etc., are much easier. The windows-rs crate offers good support for COM and WinRT.

In terms of dependencies, Rust code can be either upstream (e.g., R -> C) or downstream (C -> R) of foreign code (it is possible to have many layers of dependencies, e.g., C -> R -> C but each dependency can be considered separately). It is possible to have Rust code embedded in a foreign library and thus have a bidirectional dependency, however, you should avoid this! It is difficult to manage and build the code, and makes interop error-prone.

In other words, you can think of interoperating code as either exposing a Rust API to C code (or other languages which interoperate with a C ABI), or as exposing a C API to Rust code. The former is usually encountered when writing a Rust component which can be used from other languages, the latter when new Rust code must interoperate with legacy code.

### Using Rust code from C

When designing a Rust library to be used from other languages, the design depends on whether the library is only designed to be used from other languages or if it is meant to be used from Rust code too (and in this case, whether the usage from Rust or from other languages is primary). If the Rust code will only be used from other languages, then design a crate with no public items other than `extern` ones which are C-compatible. If the Rust code must be used from both Rust and other languages, then it is usually better to have a pure Rust crate and a second wrapper crate which provides the C API. If the primary consumer will be Rust code, then design the Rust crate to have a Rust-idiomatic API; the wrapper crate may need to do considerable work to project a C API. If the primary consumer will be other languages, then design the API of the Rust crate to be C-idiomatic (but expressed in Rust), and the wrapper crate can be a thin wrapper (perhaps entirely auto-generated by CBindgen).

### Using C code from Rust

When wrapping a foreign library for use in Rust, consider writing a first layer in C (especially if the legacy code is C++) with an API better suited for interacting with Rust. Then have a crate which is only bindings of C code into Rust (either hand-written or auto-generated). The next layer is a crate which only has the functionality of the foreign library (i.e., no client logic), but presented in a Rust-idiomatic way. The bindings crate will be all `unsafe`, the idiomatic crate should aim to have a 100% safe API. Clients should only use the idiomatic crate and never use the bindings crate (some advanced usages may require using the bindings in unanticipated ways, however these clients should create safe abstractions of their own rather than use the bindings directly). If following this pattern, it is common to give the idiomatic Rust crate the same name as the foreign library, and the bindings library the same name with the -sys suffix, e.g., `foo` and `foo-sys`. (On the topic of naming, it is idiomatic to always avoid using an `-rs` suffix on any Rust crate: it is nearly always obvious from context that the crate is a Rust library, so `-rs` usually adds nothing).

```
------------------------
     C/C++ library          libfoo
------------------------
       C wrapper            libfoo-ffi
------------------------
 Rust bindings (unsafe)     foo-sys
------------------------
Rust wrapper (idiomatic)    foo
------------------------
      Rust users
------------------------
```

## Building

If you have a mostly Rust project with some foreign libraries, you should use Cargo. If you have a project with only a small amount of Rust, then you will probably want to use the existing build system and will need to find a way to integrate the Rust build into it. Integrating Cargo and rustc with other build systems is a big topic, and this section will only be a brief summary.

To build foreign libraries inside a Cargo project, the usual approach is to orchestrate the foreign builds from [build.rs](https://doc.rust-lang.org/cargo/reference/build-scripts.html). The [CC](https://crates.io/crates/cc) crate is often used to build C/C++/ASM from a build script.

To build Rust code from a different build system you have several options, depending on your project's constraints. The simplest approach is to have the build system just call `cargo build`, however this means the build system treats the whole Rust build as a black box, that Cargo will need network access (or you can vendor the crates, see below), and if you have multiple Rust crates they will not share dependencies (unless they can all be built with a single Cargo invocation).

Another approach is to use [`cargo vendor`](https://doc.rust-lang.org/cargo/commands/cargo-vendor.html) to compute and download dependencies and keep these checked-in to version control ('vendored'). Building the Rust sub-project can be handled by the build system which will call rustc directly.

There are also more sophisticated, part-automated approaches available for some build systems. E.g., [cargo-raze](https://github.com/google/cargo-raze) for Bazel or [reindeer](https://github.com/facebookincubator/reindeer) for Buck.

You will probably want to build a [cdylib](https://doc.rust-lang.org/reference/linkage.html) rather than the default rlib or dylib. That is because a cdylib uses the C ABI rather than Rust's unstable ABI. 

It is common to end up with multiple disjoint components in each language within a single project. You probably don't want to 'split' the project by language (e.g., having a single Cargo project for all Rust code or having a high-level 'rust' directory). It is usually better to have independent builds for each component (i.e., one Cargo project for each Rust component and separate sub-projects for non-Rust components), and the main library/application build system composes the output of each sub-project.

As well as promoting more componentized design, this has practical benefits for Cargo feature propagation, dependency versioning, etc. However, it might make builds slower because there is less sharing of artefacts.

## Bindings and types

Bindings between Rust and foreign functions can either be hand-written or auto-generated. To generate bindings for C/C++ functions in Rust, use [bindgen](https://github.com/rust-lang/rust-bindgen). To generate bindings for Rust functions in C, use [cbindgen](https://github.com/eqrion/cbindgen). These tools can either be called from build.rs to create bindings on the fly, or used from the command line to generate bindings which can be adjusted and checked in to version control (the latter being a good compromise between generated and hand-written bindings).

Choosing hand-written or generated bindings is a trade-off. Automatically generated bindings are less work, stay up to date if the foreign code changes, and are more likely to be bug-free. Getting all the types right in bindings is sometimes subtle and tricky, and is not checked at compile time. Furthermore, some bindings can be target-dependent, so any approach which does not generate bindings with knowledge of the target platform has an increased likelihood of bugs.

On the other hand, hand-written bindings can sometimes be higher quality since the programmer has more knowledge of how the code is used, and binding-generating tools have limitations including around modularity (bindgen does not expect to run multiple times in a single project and therefore types which are logically the same will have multiple definitions which can lead to incompatibility).

We recommend using auto-generated bindings where possible. In particular, wrapping generated bindings with idiomatic Rust code is less fragile in the face of change or consistency issues than trying to write better bindings by hand.

Another approach if you really need custom bindings but have significant amount of code (or target-dependence) is to write your own bindings generators, either from scratch or by forking bindgen. This is more reasonable if you have some source of truth for the generated bindings other than C headers.

Whether bindings are hand-written or auto-generated, they must follow the same rules and idioms.

To call a [foreign function from Rust](https://doc.rust-lang.org/nomicon/ffi.html#calling-foreign-functions), it must be redeclared in a Rust module inside an `extern` block, e.g.:

```rust
#[link(name = "some_c_library")]
extern {
    fn callable_from_rust();
} 
```

To expose a Rust function to C code, declare it using the [`extern` keyword in its signature](https://doc.rust-lang.org/reference/items/functions.html#extern-function-qualifier). Any extern function should use the `#[no_mangle]` attribute to prevent name mangling, e.g.:

```rust
#[no_mangle]
pub extern fn callable_from_c() {
    ...
}
```

For primitive types (e.g., `long`, `double`) in these bindings, it is recommended to use the type aliases in the [libc](https://crates.io/crates/libc) crate which match with C types. Libc also provides Rust versions of non-primitive types used in the C standard library; [windows-rs](https://crates.io/crates/windows-rs) provides similar Windows-specific types.

Rust integers, floats, and booleans correspond with C equivalents and no conversion is required (see the aliases in libc for the correspondence between Rust and C types). Note that booleans in Rust must be either `0` or `1`, technically this is true in C/C++ too, however, it is common to use integers as booleans and to treat any non-zero value as true. You must ensure that a value is `0` or `1` before treating it as a Rust `bool`.

Rust raw pointers can correspond with C pointers. Use [`std::ffi::c_void`](https://doc.rust-lang.org/stable/std/ffi/enum.c_void.html) for `void` pointers. 'Opaque pointers' (where the pointee is only used in one language) can be handled trivially. If the pointee is to be accessed from multiple languages, then you must consider the pointee type for compatibility.

Treating objects as opaque is a common idiom for interop (and [in C++](https://en.cppreference.com/w/cpp/language/pimpl)). For foreign types which should be opaque in Rust, you can use a struct with a single private field which is a zero-sized array (there used to be advice to use a zero-variant enum for opaque types but that is no longer recommended because it can lead to UB in some circumstances (because the compiler might assume a zero variant enum can never be created)). If you must pass an opaque struct by value, then you can make it the correct size (though this is obviously fragile). For Rust types which should be opaque in C, you can declare but not define the type. Both bindgen and cxx have built-in support for such opaque types.

Slices in Rust combine a pointer to data with the length of the slice into a wide pointer. These components can be passed to C for use as an array without any deep conversion. The slice must be disassembled when passed to C, and if an array is passed to Rust, then it can be re-assembled (see [the FFI omnibus](http://jakegoulding.com/rust-ffi-omnibus/slice_arguments/) for details).

User-defined Rust types (structs, unions, and some enums) can be passed to foreign code and accessed there. You will need to declare structs and [unions](https://doc.rust-lang.org/reference/items/unions.html) using [`#[repr(C)]`](https://doc.rust-lang.org/nomicon/other-reprs.html#reprc) (or rarely `#[repr(packed)]`), and ensure that all field types are C-compatible.

Only enums with no fields are C-compatible. You must specify the [type of the determinant](https://doc.rust-lang.org/nomicon/other-reprs.html#repru-repri) and may want to specify the values of variants.

Other Rust types should not be passed to C, unless they will be treated completely opaquely. This includes zero-sized types, trait objects, other dynamically sized types (such as slices and strings without being adapted), tuples, and enums with fields (technically it is possible to share enums with fields which are `#[repr(C)]` but the correspondence between C and Rust types is complicated and we advise against it).

Consider the traits derived for types which will cross the FFI boundary (e.g., `Send`, `Sync`, `Copy`, `Clone`, `Default`, `Debug`). These can affect the semantics of the types in Rust (e.g., `Copy`), can affect how tools generate bindings, and/or affect the ways in which types must be handled in foreign code (e.g, if a type does not implement `Send` then it must not be moved between threads even in foreign code where this is not enforced by the compiler). If you're using a tool to generate bindings, the documentation for that tool should have more details.

### Error handling

It is never OK to unwind across the FFI boundary, therefore neither Rust panics nor C++ exceptions can be used. Rust's `Result` type is an enum with fields and therefore cannot cross the FFI boundary. This all makes error handling somewhat challenging. I don't think there is a general solution, you basically just have to do whatever fits best with the C code and convert that error handling to idiomatic Rust error handling as part of the wrapping of the FFI bindings into idiomatic Rust (e.g., implement a set of functions and macros to convert a C error code into a Rust `Result`).

## Safety

All foreign code is considered unsafe by Rust. Therefore, working with foreign code is intimately related to working with unsafe code. If you are writing code which involves FFI you should have a good understanding of unsafe code in Rust. That is a big topic! Too big to cover in depth here, but I'll try and cover some of the basics and some of the interop-specific parts. See the resources below - the [Nomicon](https://doc.rust-lang.org/nomicon) is probably the best place to start.

Unsafe code does not give the programmer permission to violate Rust's safety invariants. Unsafe code *requires* the programmer to uphold those invariants rather than relying on the compiler to check them. Safety is not a local property, it is possible to do things in unsafe code which cause runtime errors in safe code. Safety is often subtle and unintuitive to reason about, see this [blog post](https://www.ralfj.de/blog/2020/07/15/unused-data.html) for some examples. The programmer must therefore carefully consider safety for any data which passes the FFI boundary, including how it is accessed in foreign code.

When a function is marked `unsafe` then it's whole body is treated as unsafe code, however, there is a big difference between an `unsafe` function and a safe function with an `unsafe` block - the former is unsafe to call, the latter is safe to call. You should make a function `unsafe` if the caller must help maintain safety invariants in any way. Making a function safe (with or without internal unsafety) indicates that the library and compiler will ensure safety with no requirements on the caller.

Safety invariants must be enforced at the boundary between safe and unsafe code. When interoperating with foreign code that means that safety invariants must be established as part of the FFI boundary. There are several techniques for helping to ensure safety at the boundary:

* runtime assertions (e.g., asserting that a pointer is non-null),
* types (both Rust and foreign types can encode information which can help ensure invariants),
* documentation (clearly documenting safety invariants makes them easier to understand and maintain).

Ultimately, we rely on invariants being upheld in foreign code which the Rust compiler cannot check. This is mostly up to the programmer, but can be helped with the above techniques.

Safety in the context of unsafe Rust specifically means memory safety. This can be divided into a few areas which might feel disjoint:

### Uniqueness and mutability invariants around pointers

Rust's key invariant for ensuring memory safety is that all values must be immutable or unique. This property can be ensured statically or dynamically, but must always be upheld. Even in foreign code, this invariant must be respected, at least as far as it is observable to Rust code. I.e., if Rust code has a reference to a value, then foreign code must not mutate that value unless it can be guaranteed that the value cannot be read by the Rust code.

### Pointer validity invariants

If a raw pointer may be dereferenced in Rust code or converted to a safe reference, then it must be [valid](https://doc.rust-lang.org/nightly/std/ptr/index.html). Since it is usually too late to ensure validity at the point of dereference/conversion, the validity requirement must be well-documented at all points where the pointer is passed, in particular at any FFI boundary. Some aspects of validity can be checked with assertions and the FFI boundary is usually the right place to do that.

Pointer validity includes:

* pointers must be non-null,
* pointers must point to initialised data which has not been deallocated,
* pointers must point to well-aligned data,
* if the size of a value derived from the pointer's type (including any padding) is n bytes, then the pointer must point to at least n bytes from a single allocated object.

### Thread safety invariants

You must ensure that data which is not `Send` is not passed between threads and data which is not `Sync` is not shared between threads, even in foreign code. Furthermore, if dealing with multi-threaded code, the uniqueness and mutability invariant will be especially difficult to uphold. Therefore, it is easiest if Rust data is always kept on a single thread in foreign code.

### Panics

Stack unwinding due to Rust panics, C++ exceptions, or any other cause, must never cross the FFI boundary. On the Rust side, you can use [`catch_unwind`](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html) to help with this. Note that when catching panics, exceptions etc., you must ensure that no data is left in an inconsistent state. That is often impossible to achieve and aborting the thread or process is the only reasonable behaviour.

### Derived safety invariants

Many types have their own invariants required in order to preserve safety. These are usually not exposed to the user, except in `unsafe` functions where some requirements on the caller should be documented. All such requirements must be satisfied even if the function is called from foreign code. In addition foreign code may be able to create objects in ways which are impossible in Rust (e.g., by deserialization or casting from raw bytes). In these cases, you must ensure all invariants are properly established (this can be difficult since if these invariants are not user-facing in Rust code they may be poorly documented).

A good example of a 'derived' invariant is utf-8 validity. Rust strings must always be valid utf-8 and this is relied upon to ensure memory safety, even though utf-8 validity is not directly a memory safety issue. Whenever you create a Rust string, you must ensure it is valid utf-8 (see the [String docs](https://doc.rust-lang.org/stable/std/string/struct.String.html) for details).

## Memory management

There are several aspects of the object lifecycle to consider: deallocating memory, calling destructors, and ensuring expected lifetimes of objects. In Rust the object lifecycle is closely tied to ownership, so we discuss these aspects in terms of ownership. The tl;dr is that keeping ownership of an object (in terms of program design, not necessarily Rust types) in the language in which it was created is usually the best strategy.

Independent of FFI, memory must usually be deallocated by the same allocator which allocated it. Without some rather specialist effort, the allocators used from different languages will be different. Therefore, you must deallocate memory in the same language where it was allocated. If objects are passed across the FFI boundary by pointer, and that pointer is morally borrowed, then there is no tidying-up required. If ownership is transferred, then the programmer must keep around a callback to the creating language to deallocate the memory, or pass the object back for destruction.

Note that destructors will not be called automatically in the foreign language. So these must be called explicitly when the object is destroyed.

A common pattern for this is that the foreign language has a wrapper type who's destructor handles calling the creating language's destructor explicitly and calls back into that language to deallocate memory (this pattern works to or from Rust).

If objects are passed by value rather than by pointer, then they must implement `Copy` in Rust. Otherwise they will be copied in the foreign language where Rust assumes they will be moved. Note that objects cannot implement both `Drop` and `Copy` so you will not need to worry about calling `drop` in this case.

Any object accessed from Rust (whether the object was created in Rust code or not) must abide by Rust's ownership and borrowing discipline. With regards to lifecycle events, this means that destroying a borrowed pointer must not destroy the underlying object, that an owned object must not be destroyed if there are any borrowed pointers to it (or owning pointers to it if there is multiple ownership, e.g., via `Rc`), and that an object should be fully destroyed when it goes out of scope if held by value, or when all owning pointers are destroyed if it is held by pointer. Regarding FFI, this generally requires that documentation is clear about whether raw pointers/C pointers are morally owning or borrowed (and that this is tracked through foreign code), and that the FFI boundary should not transfer ownership when there are extant borrowed pointers to the object.

## C++

Interoperating with C++ is much more complicated than interoperating with C. If you follow the advice above to only interoperate at component boundaries and you design your component APIs in a conservative, C-like way (possibly by having a C-like library wrap the C++ one), then Rust/C++ interop can be fine - it is even quite well supported by Bindgen. If you must have more fine-grained interop, then things get interesting.

If you can (and plain bindgen is not enough), we recommend using [cxx](https://cxx.rs/) to generate a bridge layer and bindings between Rust and C++ code. [autocxx](https://github.com/google/autocxx) is an extension if you prefer auto-generated bindings.

Quite a lot of C++ features work well across FFI, see the [bindgen docs](https://rust-lang.github.io/rust-bindgen/cpp.html) for details. There are more links to docs on C++ interop below. It can be a bit hit and miss figuring out exactly what works and what doesn't and unfortunately some issues are not caught at compile time.
