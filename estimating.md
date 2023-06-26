TODO exec summary

# Estimating the complexity of interop

This chapter aims to help you estimate the costs and risks of integrating Rust into an existing mixed-language project. You'll be able to make a better judgement if you read the rest of the documentation and can therefore understand the issues more deeply. Hopefully this chapter can give you a framework for estimation, and if you don't have time to dive deeper, then at least give you a rough idea.

This chapter does not aim to help with the question of whether Rust is a good choice for a project, only to cover the integration component of that choice.

## Rewriting a project or adding to a project?

Integrating Rust with a foreign language typically happens in one of two scenarios: adding Rust to an existing non-Rust project, or converting an existing non-Rust project into a Rust one. The difference being that in the second case, the goal is for the entire project (or most of it) to be Rust, whereas in the first case, only a small portion may ever be Rust.

When adding to a project, the parameters which affect integration are usually well-known. Typically the project will not change much at the same time as adding a Rust portion. So there is not much opportunity to change the cost or risk of adding Rust. In this case the more that the Rust portion is self-contained, the better. Adding a little Rust all over the project is a much riskier undertaking than adding single Rust component with a limited interface.

When converting a project to Rust, as good software engineers, we want to do the conversion as incrementally and iteratively as possible. However, this has the requirement that existing foreign code integrates with new Rust code. The more incremental the conversion, the more fine-grained the integration must be and this will make interop more difficult. From the perspective of integration, the best-case scenario is to do a clean rewrite with zero temporary interop (taking advantage of existing test suites, documentation, requirements, etc. but not doing an incremental rewrite). Where this is not possible, doing a rewrite one component at a time, rather than one function, data structure, or file at a time is highly desirable. In my opinion, the former is likely to be successful (although depending on many other factors), but the latter is doomed to failure in nearly all circumstances.

## Languages

As a general rule, integrating Rust with C is the easiest scenario. If the foreign language is C++, then things will be harder, but how hard will depend on the style used in the C++ code and the nature of the project (see below for more discussion). The more C-like the code (and especially the interface with Rust), the easier integration will be.

If the foreign language is a managed language (C#, Java, Python, Ruby, etc.) then integration is possible but will have very different characteristics to integrating with native languages. The costs and risks will be specific to the foreign language (some have much better support for Rust integration than others). These docs should be expanded in the future to better cover this scenario.

## Tooling

### Build and dependency management

Rust projects are natively built with Cargo. Cargo is a build system and package manager. These areas of functionality are tightly integrated and Cargo does not integrate easily with other build systems. Using rustc directly is a rare path and is thus unpleasant and poorly supported.

One straightforward scenario is where a Rust crate is not depended on by any foreign code (i.e., it is a leaf node in the dependency graph). In this case all foreign code is upstream of Rust code and build system integration is fairly straightforward. Similarly, if a Rust crate does not depend on any foreign code (at least outside of the standard library), then the Rust code can be compiled in relative isolation and again build system integration is simpler. Having multiple Rust components in one or other of these situations is also fairly straightforward, as long as they don't depend on each other and you don't mind some duplication of compilation where there are shared dependencies (i.e., the extra time and potential version incompatibility).

More overlap and constraints between components makes build system integration more complicated. If you will require multiple 'layers' of dependency between Rust and foreign code, and require the Rust components to play nicely with respect to Cargo's version resolution, then integration will be complicated and require effort (how much depends on which build system you're integrating with, see below, and the details of the project).

The most common pattern for build system integration, is to run Cargo's package management functionality offline. Rust dependency sources can then be stored in-tree and when building, they only need to be compiled (which, with some setup, does not require Cargo). Tooling exists to help with this (cargo vendor). The downside of this approach is that updating dependencies is a manual step. It also makes the source tree much larger and the VC history a little less clear. If your project can accommodate this pattern, then build system integration will be much, much easier than if it can't.

Some build systems have existing tools for Rust/Cargo integration (or are a well-troden path with good docs). If you are using these, integration will be easier:

* Buck
* Bazel
* make/cmake?
* TODO any more? Links and descriptions for above

Linking Rust binaries with other native binaries is generally straightforward because Rust follows native standards for each platform. Integration may be more difficult if you require dynamic linking since Rust has a bias towards static linking, and although dynamic linking is supported, it is a less common path. Link-time optimisation (and PGO) will work with Rust, including across languages (which means that there is not even an optimisation penalty for mixing Rust with C/C++ in many cases), however, Rust uses LLVM for its backend, and so LTO will only work within the LLVM ecosystem. TODO is this true? Is it possible to LTO without LLVM?


### CI

TODO

### Static analysis and related tooling

elf binaries, debuginfo, llvm ecosystem
    valgrind seems to work quite well
relying on gcc or VS means hard work
analyses which rely on optimisation might have issues
anything which relies on source code or AST will not work
Rust has fairly good tooling (rustfmt, clippy, miri, etc) often but not always works in the presence of foreign code
TODO - more specific stuff for widely used tools

### Debugging, profiling, and other developer tools

debugging and profiling generally work across language boundaries, IDEs generally don't (although usually you can get pretty good independent coverage of the different languages)
only likely to be problems if tooling is tied to gcc or VS ecosystems

## Nature of the project

This section covers some topics which are about various aspects of a projects architecture, design, and code style. These are mostly not 'black and white' topics, but rather have a bunch of nuance and subtlety. These are also mostly things where you can make choices early in our adoption of Rust which will make integration *a lot* easier.

If your project has a microservices-style architecture with components running on their own nodes or in their own processes and you can keep Rust in entirely new components, then integration should be easy (relatively). Rust has good support for many forms of IPC, networking, serialization/deserialization, etc. You may still have integration issues with build or CI which need considering, but for the code itself, you are in a best-case scenario.

Only slightly worse than the previous situation is if you have a monolithic application, but you can keep the new Rust code in a separate process. The integration will still be easy, but you're likely to have more design issues.

Assuming Rust code must be in the same process as existing foreign code, the more modular the architecture, the easier integration will be. Most challenges with interop are design challenges where it is difficult to ensure modularity (and the specific kind of modularity which make interop easier) in the face of requirements which favour tight integration of components. Keeping Rust to modules/components with strong, well-designed APIs will make interop easier.

More specifically, the kinds of modularity that benefit interop are:

* TODO

And a few things which don't help much:

* keeping data private and using accessor methods (unless those accessor methods enforce invariants of the data),
* TODO

Rust interop primarily uses the C ABI, but many C++ features work across the language boundary too. There are both C and C++ features which can make interop more or less difficult. Difficult can mean requiring more tooling, more conversion of data types at the language boundary, more invariants which must be enforced by the programmer, language features which must be manually emulated rather than implemented by the compiler, riskier code (i.e., bugs are more likely and/or harder to see), or that direct sharing of data or calling of functions is not possible.

The following features work without any runtime conversion and don't require tooling (though using bindgen may make things easier):

* primitive numeric and boolean data,
* structs and field access,
* simple (C-like) enums,
* function calls with C ABI,
* opaque pointers (i.e., pointers which are never dereferenced),

The following features require either manual emulation in Rust or some encoding:

TODO, for each, discuss the implications

* lifecycles methods (constructors, copy constructors, destructors, etc.),
* method calls,

TODO - features, C++ features
    enums
    inheritance
    move semantics
    pointer arithmetic, casting, etc
    generics, template tomfoolery
    Rust features into C/C++ - enums, traits, etc, etc
    
