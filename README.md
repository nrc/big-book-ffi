# The Big Book of Rust Interop

A guide to integrating Rust with other programming languages.

Read the [rendered book](https://nrc.github.io/big-book-ffi).

To read, start at the [introduction](intro.md) or see the contents in [SUMMARY.md](SUMMARY.md).

## Building

You'll need to install [mdbook](https://rust-lang.github.io/mdBook/), the easiest way is `cargo install mdbook`.

Then, `mdbook build` to build and `mdbook serve` to serve up a local copy.

## Contributing

Contributions are welcome!

If you notice an error or think something could be improved, please open an [issue](https://github.com/nrc/big-book-ffi/issues/new) or PR.

This book is licensed under CC-BY-4.0.

# TODO plan

Questions to answer

* Define the landscape
  - precisely, what Rust can do
  - trivial stuff
    - bindings, built in & generated
  - what the problems are
  - why you need non-trivial stuff
* Framing
  - trivial vs non-trivial work
  - semantics overlap of two languages
    - the closer the easier (representation, plus invariants)
  - most non-trivial stuff unique to a project
  - more from strategy doc?
* What can we do without non-trivial stuff?
* What does an abstraction layer do?
  - safety
  - PL semantics stuff
  - project semantics stuff
* How to write the abstraction layer?

So, how does this fit into the book format?


* bindings
* representations
* safety

Other views of docs

* answers/summary for senior managers
* questions/answers for engineers planning - can we adopt Rust? How much work will it be?


