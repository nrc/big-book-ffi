# Introduction

It is extremely common to integrate multiple languages in a single program or system. This might occur due to different languages being better suited for different components, requiring new code to work with old code written in an older language, or when incrementally adopting a new language.

In some cases, different languages can be isolated in different processes (either on the same machine or different ones), language interoperation is then just a matter of finding compatible IPC solutions. Where this isn't feasible, libraries produced by different compilers will be linked together into the same executable and run together in the same process.

In the Rust world, the most common foreign language to interoperate with is C. Interop between Rust and C is easier than much other language interop because neither language has a required runtime, Rust has built-in support for interoperating with C and for the C ABI, and there are good supporting tools. However, there are many rough edges, and dealing with the difference in safety invariants is an involved task.

Stepping back, there are two aspects of Rust interop: the mechanics of aligning runtime semantics across different languages, and ensuring Rust's safety invariants are upheld. With interop between Rust and C, the former is mostly solved by tooling and built-in features. For interop between Rust and C++ or other languages, there is more work to do (work which is often specific to a project, rather than being a reusable solution). The safety aspect is solved by designing a safe API for foreign components. Some techniques for this are generic and apply to most situations, some are specific to the constraints of a project.

TODO - organisation of docs



