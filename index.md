- linking against native libraries
- os-specific linker arguments
- inline testing
- loading a wav file with mm
- ocaml-rs doesn't support ocaml5
- ocaml-rs float32 arrays are broken
- opam install and opam pin don't handle local packages in some cases
- dune sourcetree ignores directories begining with "."

I recently made my first non-trivial OCaml project - a synthesizer library called [llama](https://github.com/gridbugs/llama/). It's made up of multiple Opam packages, it links against native code (written in Rust), it's (somewhat!) documented, it's released to the Opam repository.