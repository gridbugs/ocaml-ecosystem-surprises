I recently made my first non-trivial OCaml project - a synthesizer library
called [llama](https://github.com/gridbugs/llama/). It's made up of multiple
opam packages, it links against native code, it calls into code in a foreign
language (Rust), it's (somewhat!) documented, it works on macOS and Linux, it's
released to the opam repository. Non-trivial! The language ecosystem I've spent the most time
in is Rust/Cargo and as such I'm accustomed to things "just working". This is a
post about the times while developing my library that I was surprised or
confused or frustrated by something in the OCaml ecosystem that didn't "just work".

Before we get into it do I need to point out that my day job is working on the
Dune build system. Despite being frustrated with the tooling at times I put up
with it because I love programming in OCaml. I also love programming in Rust and
use it for many personal projects. My comparisons between OCaml and Rust are
mostly facetious. I know there are more people working on Rust than OCaml and it
has momentum on its side being relatively young and cool. It makes different
trade-offs and has plenty of problems of its own.

The opinions I express here are my own and not those of Tarides or the (other)
dune developers.

## Linking Against Native Libraries

My library makes noise by sending samples to the sound card
via a small Rust library that uses
[cpal](https://crates.io/crates/cpal) to talk to the audio driver and
[ocaml-rs](https://crates.io/crates/ocaml) for OCaml interop. The Rust code is compiled to an archive
`liblow_level.a` and I set up the dune project for the library based on
[`ocaml-rs`'s docs](https://zshipko.github.io/ocaml-rs/).

My dune file looked like this:
```clojure
(library
 (name llama_low_level)
 (foreign_archives low_level)
 (c_library_flags
  (-lpthread -lc -lm)))
```

Trying to build this:
```
$ dune build
...
Error: No rule found for dlllow_level.so
```

I only have a static archive (.a) of the Rust library - not a shared object (.so)
file. Searching the [dune (library ...) stanza
documentation](https://dune.readthedocs.io/en/stable/dune-files.html#library)
for the word "dynamic" I found:

> `(no_dynlink)` disables dynamic linking of the library. This is for advanced use
> only. By default, you shouldn’t set this option.

A bit of a scary looking message - my use case doesn't seem too advanced. Adding `(no_dynlink)` does seem to fix the problem:

```clojure
(library
 (name llama_low_level)
 (no_dynlink)
 (foreign_archives low_level)
 (c_library_flags
  (-lpthread -lc -lm)))
```

Next I made a small executable to test calling a function in the
`llama_low_level` library:

```clojure
(executable
 (public_name experiment)
 (libraries llama_low_level))
```

Building this caused my terminal to fill with errors. Here's the start of the
long error message:
```
$ dune build
File "bin/dune", line 2, characters 14-24:
2 |  (public_name experiment)
                  ^^^^^^^^^^
Undefined symbols for architecture arm64:
  "_AudioComponentFindNext", referenced from:
      cpal::host::coreaudio::macos::audio_unit_from_device::h062e0db473d1abd3 in liblow_level.a(low_level-0e968d755c3460c1.low_level.2e2e0021-cgu.6.rcgu.o)
```

Searching for "AudioComponentFindNext" led me to some Apple developer docs for
the `AudioToolbox` framework. So the foreign archive must depend on some frameworks
on macOS for doing "audio stuff" and I need to tell the linker about it.
Eventually I found the appropriate linker flags on stack overflow to copy/paste
into my project:

```
-framework CoreServices -framework CoreAudio -framework AudioUnit -framework AudioToolbox
```

But what do I have to _do_ with these flags? If I was compiling a C program I
could pass them to the linker via `clang`:
```
clang foo.c -Wl,-framework,CoreServices,-framework,CoreAudio,-framework,AudioUnit,-framework,AudioToolbox
```

The `-Wl` flag tells clang to pass its argument to the linker. I need to find out how to do this in dune/OCaml.

Reading through `dune`'s library docs again and there's this field:
> `(library_flags (<flags>))` is a list of flags passed to ocamlc and ocamlopt
> when building the library archive files.

Sounds promising. Now I need a way to get `ocamlc` to pass custom flags to the
linker. `man ocamlc` gives two condenders:

> -cclib -llibname
>
> Pass the -llibname option to the C linker when linking in "custom runtime"
> mode  (see  the  -custom  option).  This  causes the given C library to be
>linked with the program.
>
> -ccopt option
>
> Pass the given option to the C compiler and linker, when linking in  "custom  runtime"  mode  (see  the -custom option). For instance, -ccopt -Ldir
> causes the C linker to search for C libraries in directory dir.


Putting these together:
```clojure
(library
 (name llama_low_level)
 (no_dynlink)
 (foreign_archives low_level)
 (c_library_flags
  (-lpthread -lc -lm))
 (library_flags
  (-ccopt -framework -ccopt CoreServices -ccopt -framework -ccopt CoreAudio -ccopt -framework -ccopt AudioUnit -ccopt -framework -ccopt AudioToolbox)))
```

Both `-ccopt` and `-cclib` can be used interchangeably in that example and they
seem to work. The docs for `-cclib` suggest that it's only for passing library
names but it's not documented what happens if you pass it something other than
`-llibname`.

I later found out that you can avoid passing `-ccopt` before each linker
argument by putting the linker arguments in quotes:

```clojure
(library_flags
 (-ccopt "-framework CoreServices -framework CoreAudio -framework AudioUnit -framework AudioToolbox))
```

Now it builds!

But only on macOS.

On Linux the appropriate linker incantation is `-lasound` if you're lucky and
the linker knows where to look for the `libasound.so` library. In general though
the linker arguments are the output of running:
```
$ pkg-config --libs alsa
```
On my Linux (NixOS) computer this is:
```
-L/nix/store/qnf36msgsjh17sy9dakvqnvv7sgr8dfg-alsa-lib-1.2.9/lib -lasound
```

Keen-eyed readers will have noticed that `libasound.so` is a dynamic library and
might be wondering why this works despite setting `(no_dynlink)` in the dune
file. I take the fact this this does work to mean that when the docs for
`(no_dynlink)` say "disables dynamic linking of the library", they are just
referring to how the OCaml library links against the libraries listed in
`(foreign archives ...)` (for us this is just "low_level") - not libraries
explicitly passed to the linker (such as `asound`).

On Linux I need to generate the linker flags from a command rather than
hard-coding them in the dune file. The only way to do this in dune is to
generate a text file containing an [S-expression
(sexp)](https://en.wikipedia.org/wiki/S-expression) representation of a list of
arguments and then using dune's `:include` keyword to load a list of arguments
from the file.

So I need to generate a file with the following contexts:
```clojure
("-cclib" "-L/nix/store/qnf36msgsjh17sy9dakvqnvv7sgr8dfg-alsa-lib-1.2.9/lib -lasound")
```
Note the opening and closing parentheses. These are required as the file must
contain a sexp. Also this time around we do have to pass `-cclib` as `-ccopt`
doesn't work. I guess because we are passing the name of a library to link
against?

To generate this file we can use a dune rule:
```clojure
(rule
 (action
  (with-stdout-to
   library_flags.sexp
   (progn
    (echo "(\"-cclib\" \"")
    (bash "pkg-config --libs alsa")
    (echo "\")")))))

(library
 (name llama_low_level)
 (no_dynlink)
 (foreign_archives low_level)
 (c_library_flags
  (-lpthread -lc -lm))
 (library_flags
  (:include library_flags.sexp)))
```

This works on Linux but of course now it won't build on macOS as we removed the
macOS-specific linker flags and replaced them with the Linux ones. To work in
general (well just on macOS and Linux) we can use dune's `(enabled_if ...)`
field to use a different rule to generate `library_flags.sexp` on macOS.

```clojure
(rule
 (enabled_if
  (= %{system} macosx))
 (action
  (write-file
   library_flags.sexp
   "(\"-ccopt\" \"-framework CoreServices -framework CoreAudio -framework AudioUnit -framework AudioToolbox\")")))

(rule
 (enabled_if
  (= %{system} linux))
 (action
  (with-stdout-to
   library_flags.sexp
   (progn
    (echo "(\"-cclib\" \"")
    (bash "pkg-config --libs alsa")
    (echo "\")")))))

(rule
 (enabled_if
  (and
   (<> %{system} macosx)
   (<> %{system} linux)))
 (action
  (write-file library_flags.sexp "()")))

(library
 (name llama_low_level)
 (no_dynlink)
 (foreign_archives low_level)
 (c_library_flags
  (-lpthread -lc -lm))
 (library_flags
  (:include library_flags.sexp)))
```
Note that the system name for macOS is "macosx". There's an "x" at the end.
Don't forget the "x" or the comparison will always be false.

Also note the third rule which creates an empty sexp file in the case that the
system is neither Linux nor macOS. This is so that if someone deigns to build
this on a non-Linux non-macOS system they don't get an error about the
`library_flags.sexp` file being missing (instead they will probably get some
sort of linker error).

This is all a bit of a mess. It's awkward to construct sexp files by manually
adding parentheses around the output of commands and adding the right level of
escaping quotes, and the catch-all rule for non-Linux non-macOS systems has to
repeat the `enabled_if` conditions from the previous rules in its negation of
them.

Generating sexp files is difficult in dune files but it's the only way to
dynamically pass arguments to the linker (and to do a bunch of other operations
that require dynamic input). The recommended way to probe the current system
is not to do it from a dune file but instead write an OCaml program to do it.
There's a helper library for this purpose:
[dune-configurator](https://opam.ocaml.org/packages/dune-configurator).

Here's the OCaml program that generates `library_flags.sexp`:
```ocaml
module C = Configurator.V1

let macos_library_flags =
  let frameworks =
    [ "CoreServices"; "CoreAudio"; "CoreMidi"; "AudioUnit"; "AudioToolbox" ]
  in
  List.map (Printf.sprintf "-framework %s") frameworks

let () =
  C.main ~name:"llama_low_level" (fun c ->
      let linker_args =
        match C.ocaml_config_var_exn c "system" with
        | "macosx" -> macos_library_flags
        | "linux" -> (
            let default = [ "-lasound" ] in
            match C.Pkg_config.get c with
            | None -> default
            | Some pc -> (
                match C.Pkg_config.query pc ~package:"alsa" with
                | None -> default
                | Some conf -> conf.libs))
        | _ -> []
      in
      let cclib_arg = String.concat " " linker_args in
      C.Flags.write_sexp "library_flags.sexp" [ "-cclib"; cclib_arg ])
```

This goes into a file `discover.ml` in a separate directory from the `low_level`
library's files, and it's got a dune file which just builds an executable:
```clojure
(executable
 (name discover)
 (libraries dune-configurator))
```

Now the dune file for `low_level` can run `discover.exe` in a single rule rather
than having a separate rule for each system:
```clojure
(rule
 (target library_flags.sexp)
 (action
  (run ./config/discover.exe)))

(library
 (name llama_low_level)
 (no_dynlink)
 (foreign_archives low_level)
 (c_library_flags
  (-lpthread -lc -lm))
 (library_flags
  (:include library_flags.sexp)))
```

## Rust Interoperability

Rust interoperability is easier than I expected! The crate (Rust library)
[ocaml-rs](https://crates.io/crates/ocaml) defines some annotations you can
attach to functions to expose them in a way that's compatible with OCaml's `external` syntax.

In Rust you write:
```rust
#[ocaml::func]
pub fn send_sample(mut t: ocaml::Pointer<OutputStreamOcaml>, sample: f32) {
    let output_stream = t.as_mut();
    output_stream.send_sample(sample);
}
```
...and then in OCaml you can write:
```ocaml
external send_sample : t -> float -> unit = "send_sample"
```

You can even get dune to build your cargo project for you:
```clojure
(rule
 (target liblow_level.a)
 (deps
  (source_tree low-level-rust))
 (action
  (progn
   (chdir
    low-level-rust
    (run cargo build --release))
   (run mv low-level-rust/target/release/%{target} %{target}))))
```

This rule creates the `liblow_level.a` file that was used in the previous
section. The `low-level-rust` directory in the `(source_tree low-level-rust)`
contains a regular old cargo project (ie. it has a `Cargo.toml` file and a
`src` directory containing Rust code).

There's lots more information about interoperability between OCaml and Rust in
the [ocaml-rs book](https://zshipko.github.io/ocaml-rs/).

I intended to release my library on opam and one caveat of releasing packages
with a Rust component to opam is that the sandbox environment used to build opam
packages does not have internet access, and building a Rust project typically
involves downloading dependencies. To get around this, the source tarball
refered to in the opam package's manifest must already include all the Rust code
needed to build the project. Setting this up is easy - run `cargo vendor` in the
Rust project to create a `vendor` directory containing the code for all the
project's dependencies. The command also prints some instructions for building
with the vendored code:

```
To use vendored sources, add this to your .cargo/config.toml for this project:

[source.crates-io]
replace-with = "vendored-sources"

[source.vendored-sources]
directory = "vendor"
```

So I went ahead and created the file low-level-rust/.cargo/config.toml after
vendoring all my dependencies, and to make sure that no dependencies were being
clandestinely downloaded I replaced `run cargo build --release` with
`run cargo build --release --offline` in the `dune` file, and also removed all my
cached cargo packages by running `cargo cache --remove-dir all`.

```
$ dune build
File "src/low-level/dune", line 1, characters 0-224:
 1 | (rule
 2 |  (target liblow_level.a)
 3 |  (deps
 4 |   (source_tree low-level-rust))
 5 |  (action
 6 |   (progn
 7 |    (chdir
 8 |     low-level-rust
 9 |     (run cargo build --release --offline))
10 |    (run mv low-level-rust/target/release/%{target} %{target}))))
error: no matching package named `cpal` found
location searched: registry `crates-io`
required by package `low_level v0.1.0 (/Users/s/src/llama/_build/default/src/low-level/low-level-rust)`
As a reminder, you're using offline mode (--offline) which can sometimes cause surprising resolution failures, if this error is too confusing you may wish to retry without the offline flag.
```

Hey what gives?

The `cpal` library has been vendored and its source code has made its way to the
`_build` directory:
```
 $ ls _build/default/src/low-level/low-level-rust/vendor/cpal/
examples  CHANGELOG.md  Cargo.toml  Dockerfile  README.md
src       Cargo.lock    Cross.toml  LICENSE     build.rs
```

And yet cargo cannot see it.

Fifteen or so minutes of confusion turned to disappointment when I learnt
through trial and error that the `.cargo` directory wasn't being copied because
`(source_tree ...)` silently ignores directories whose names begin
with a ".". This meant that cargo didn't know to look in the `vendor` directory
for packages. If you've been bitten by this and found this page by searching "dune
ignores hidden files" then feel free to share your story at the github issue:
[Hidden folders are ignored in source_tree
dep](https://github.com/ocaml/dune/issues/7135). I fixed the problem by renaming
the directory to `dot_cargo`. Don't be tempted to name it `_cargo` as
`source_tree` also ignores directories whose names begin with a "_". Some helpful
advice from the aforementioned github issue was to explicitly tell dune about
the `.cargo` directory in a dune file. Combined with suggestions from the
ocaml-rs book, the dune file in the Rust project (next to `Cargo.toml`) is:
```clojure
(dirs :standard .cargo \ target)
(data_only_dirs vendor)
```

Directories starting with "." and "_" are ignored by default to prevent dune
from accidentally copying `.git` or `_build` into the `_build` directory, but
this feels like a case of a tool trying to be helpful by making assumptions that
turn into surprising behaviour when the assumptions don't hold. These types of
problems tend to be hard to debug as the symptoms of the problem are far removed
from the source of the problem (cargo couldn't find a dependency because I put
its configuration in a directory whose named started with a "."). In addition
it's often hard to find documentation or advice online for this reason. It
wasn't obvious that this was a quirk of `(source_tree ...)` so I didn't know to
check its documentation and even if I did, at the time of writing its
documentation just says:

> (`source_tree <dir>)` depends on all source files in the subtree with root `<dir>`.

The actual place I should have looked is in the [documentation for `(dirs
...)`](https://dune.readthedocs.io/en/stable/dune-files.html#dirs):

> The `dirs` stanza allows specifying the subdirectories Dune will include in a
> build. The syntax is based on Dune’s Predicate Language and allows the
> following operations:

> - The special value `:standard` which refers to the default set of used
>   directories. These are the directories that don’t start with `.` or `_`.

...but I only know that `dir` was related to my problem after reading solutions
to my problem in a GitHub issue. I only found the issue when I went to create an
issue of my own and it was suggested as a duplicate based on the title.

I find this UX anti-pattern to be pervasive in the OCaml tooling ecosystem
which I think is why so often I'm surprised by something one of our tools does
and why our tools tend to fail in complex, difficult to understand ways. There
are so many special cases aimed to be helpful but when a special case fails and
prevents me from doing something that should be simple I wish that the tools
were dumber and failed in simpler ways which were easy to understand and fix or
workaround.

If I got to name this pattern I would call it "Jar Jar-ing". In the Star Wars
prequel trilogy Jar Jar Binks is always eager to help but his attempts usually
backfire in ways that make the situation worse. It's also a play on the word
"jarring". You get it.

But I digress.

## Adventures Trying to Read a `.wav` File

I wanted the ability to load and play `.wav` files in my synthesizer library. I'd
been interested in checking out the
[ocaml-mm](https://github.com/savonet/ocaml-mm) multimedia library for a while
and this seemed like a good time. I got my hands on some old-school drum samples
that I wanted to use for my synth, and made a little program that used `mm` to
read a `.wav` file:
```ocaml
let () =
  let wav_file = new Mm.Audio.IO.Reader.of_wav_file "./cymbal.wav" in
  let sample_rate = wav_file#sample_rate in
  print_endline (Printf.sprintf "sample_rate: %d" sample_rate)
```
This prints out `sample_rate: 44100`. We're off to a good start.

Let's try reading some audio samples from the file:
```ocaml
let () =
  let wav_file = new Mm.Audio.IO.Reader.of_wav_file "./cymbal.wav" in
  let buffer = Mm.Audio.create 2 10000
  let _ = wav_file#read buffer 0 1 in
  ()
```
Running this:
```
Fatal error: exception File "src/audio.ml", line 1892, characters 21-27: Assertion failed
```

Less promising, but maybe I screwed up the indexes or something. Let's check
which assertion failed:
```ocaml
match sample_size with
  | 16 -> S16LE.to_audio sbuf 0 buf ofs len
  | 8 -> U8.to_audio sbuf 0 buf ofs len
  | _ -> assert false
```

My wav file uses 24-bit samples but I probably can't hear the difference between
24-bit and 16-bit drum sounds so I used `ffmpeg` to downsample my `.wav` to
16-bit:
```
$ ffmpeg -i cymbal.wav -af "aformat=s16:sample_rates=44100" cymbal-16bit.wav
```

And make sure that it worked:
```
$ file cymbal-16bit.wav
cymbal-16bit.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 16 bit, mono 44100 Hz
```

And updated the code to load the new 16-bit `.wav` file, and its output was:
```
Fatal error: exception Mm_audio.Audio.IO.Invalid_file
```

So I gave up and switched to Rust.

[Hound](https://crates.io/crates/hound) is a Rust library for reading and
writing `.wav` data. Unlike `mm` which supports a wide range of media, `hound`
only supports `.wav`. Also unlike `mm`, `hound` is actually capable of reading
`.wav` files. I'd already gone to the effort of getting Rust interoperability
working to get access to the `cpal` library for talking to the audio driver so
it was easy to add an additional Rust dependency on `hound` and write a little
wrapper that reads a `.wav` file and copies the contained audio samples into an
OCaml array.

The first thing I did when I started this project was getting the Rust
interoperability working as it's necessary to actually make sound at all via the
`cpal` library. A nice unintended consequence was that when I couldn't find a
high-quality OCaml library to load `.wav` files I could just use a Rust library
instead. It's no secret that there are far more people writing Rust libraries
than OCaml libraries but since OCaml/Rust interop is so easy we can just fill
in any gaps in our library ecosystem with Rust libraries until our own libraries
are up to scratch.

I ran into an interesting bug while copying audio samples into an OCaml array.
Currently if you have a Rust function that returns an array of 32-bit floats:
```rust
#[ocaml::func]
pub fn make_float_array() -> Vec<f32> {
    vec![0.0, 1.0, 2.0]
}
```
...and you refer to it in OCaml:
```ocaml
external make_float_array : unit -> float array = "make_float_array"
```
...when you call the function in OCaml you get an array whose iteration
behaviour works as expected, but indexing it with `Array.get` always returns
`0.0`. In some cases arrays of floats in OCaml are represented in a special
format but `ocaml-rs` was only converting 64-bit floats into this format. At the
time of writing I have an open PR to fix this bug.

## Inline Testing

My synthesizer library can decode MIDI both from files and external devices. I
elected to write my own MIDI parser rather than use the one from `mm`. Fool me
once, etc., and besides, parsing MIDI is pretty straightforward. The most
complicated part is probably "Variable Length Quantities" - an encoding for
integers where the most-significant bit of each byte is used to mark the final
byte of the integer. Kind of like C strings but for integers.

To check my understanding of the MIDI spec I wrote some tests:
```ocaml
let variable_length_quantity a i = (* Parse a variable length quantity. *)

let%test_module _ =
  (module struct
    (* Run the parser on an array of ints. *)
    let make ints =
      run variable_length_quantity (Array.map char_of_int (Array.of_list ints))

    let%test _ = Int.equal 0 @@ make [ 0 ]
    let%test _ = Int.equal 0x40 @@ make [ 0x40 ]
    let%test _ = Int.equal 0x2000 @@ make [ 0xC0; 0x00 ]
    let%test _ = Int.equal 0x1FFFFF @@ make [ 0xFF; 0xFF; 0x7F ]
    let%test _ = Int.equal 0x200000 @@ make [ 0x81; 0x80; 0x80; 0x00 ]
    let%test _ = Int.equal 0xFFFFFFF @@ make [ 0xFF; 0xFF; 0xFF; 0x7F ]
  end)
```

I based this off dune's [documentation for inline
tests](https://dune.readthedocs.io/en/stable/tests.html#inline-tests) and it
worked well. I can run my tests with `dune test`. Tests are written using the
`ppx_inline_test` package.

In Opam when you depend on a package that you only need for testing it's
possible to mark the package as `with-test`:
```
  "ppx_inline_test" {with-test}
```

This is useful since normal users of a package probably don't want to install
the package's test-only dependencies, but when developing the package or in
CI you can install the package with `opam install <package> --with-test` to
install its test-only dependencies along with its regular dependencies.

To experiment with this feature I tried installing my library on without passing
`--with-test`:
```
File "test/dune", line 6, characters 7-22:
6 |   (pps ppx_inline_test))
           ^^^^^^^^^^^^^^^
Error: Library "ppx_inline_test" not found.
```
OK that's true, I don't have `ppx_inline_test` installed but I'm also not trying
to run my tests so what's going on here?

It turns out that packages that do pre-processing like `ppx_inline_test` cannot
be marked as `with-test`; they must be unconditional dependencies. This is
because preprocessor directives like `let%test` are not valid OCaml syntax, and
the OCaml compiler is unable to parse the files until
_something_ has gone through and removed all the preprocessor directives.

Of course I _could_ just add `ppx_inline_test` as an unconditional dependency but
it seems like a shame since my MIDI library has no dependencies besides `ocaml`
and `dune`. `ppx_inline_test` has many dependencies of its own, so adding it
unconditionally to my MIDI library would actually be adding dependencies on:
```
base
csexp
dune-configurator
jane-street-headers
jst-config
ocaml-compiler-libs
ppx_assert
ppx_base
ppx_cold
ppx_compare
ppx_derivers
ppx_enumerate
ppx_globalize
ppx_hash
ppx_here
ppx_inline_test
ppx_optcomp
ppx_sexp_conv
ppxlib
sexplib0
stdio
stdlib-shims
time_now
```

Anyone using my MIDI library would now have to install all of those packages.

This came as a bit of a shock since I've been spoiled by Rust where I can do:
```rust
fn some_function() { ... }

#[test]
fn test_of_some_function() { ... }
```
...and the tests will only be compiled when running `cargo test`. I can add
test-only dependencies in Rust but they are only installed when someone goes to
run the tests. I can depend on test-only preprocessors from external libraries
without users of my libraries having to know or care.

The recommended workaround for OCaml is to publicly expose all internal APIs you
want to test with `ppx_inline_test` and perform the tests in a different package
which depends on `ppx_inline_test`.

In my case I needed to expose the internal parser used in the MIDI library:
```ocaml
module For_test : sig
  module Byte_array_parser : sig
    include module type of struct
      include Byte_array_parser
    end
  end
end
```

I put it inside a new module `For_test` whose name is meant to scare users away
like the stripes on a poisonous snake. I'm only exposing this API for tests and
if you use it your code will probably break without warning.

Then in a new package which I named `llama_tests` and will never release to
Opam, I depend on both my MIDI package and `ppx_inline_test`, and I moved the
tests there.

I've been told that requiring developers to expose private APIs for testing
purposes encourages them to think more about how they structure their private
APIs. This may be true but I can't help but feel like this extra barrier to
testing will also have the practical effect of (possibly subconsciously)
reducing the number of tests that get written. Dune already has a higher
barrier for testing than cargo since you have to explicitly enable inline tests
in the dune file of the library under test and install `ppx_inline_test`. In
Rust if I have a _passing curiosity_ about whether my code works in some case
all I have to do is add a function tagged with `#[test]` and run `cargo test`.

## Local Packages and Opam

My synthesizer project is made up of several interdependent Opam packages in a
single git repo. Opam has a feature where if you run `opam install .` in a
directory containing one or more .opam files it will install the packages they
describe along with their dependencies. If you just want the dependencies
installed and not the local packages themselves you can run `opam install .
--deps-only`. This is a handy command for installing the dependencies of a
project you intend to work on.

I recently tried setting up my synth project on a new machine. After a fresh
clone of the git repo in a fresh Opam switch, I ran:
```
$ opam install . --deps-only
[ERROR] Package conflict!
  * Missing dependency:
    - llama_midi >= 0.0.1
    no matching version

No solution found, exiting
```

`llama_midi.opam` was definitely present. Another command, `opam pin .` is kind
of like `opam install .` except it also marks any local packages
(.opam files in the current directory) as being local and won't try to install
them from the Opam repository. That must be my problem:
```
$ opam pin .
This will pin the following packages: llama, llama_core, llama_interactive,
llama_midi, llama_tests. Continue? [Y/n]
```
Sounds promising...
```
llama_midi, llama_tests. Continue? [Y/n] Y
Processing  3/5: [llama: git] [llama_core: git] [llama_interactive: git]
llama is now pinned to git+file:///.../llama#main (version 0.0.1)
llama_core is now pinned to git+file:///.../llama#main (version 0.0.1)
llama_interactive is now pinned to git+file:///.../llama#main (version 0.0.1)
Package llama_midi does not exist, create as a NEW package? [Y/n]
```
So far so good...
```
Package llama_midi does not exist, create as a NEW package? [Y/n] Y
llama_midi is now pinned to git+file:///.../llama#main (version ~dev)
Package llama_tests does not exist, create as a NEW package? [Y/n] Y
llama_tests is now pinned to git+file:///.../llama#main (version ~dev)
```
And...
```
[ERROR] Package conflict!
  * Missing dependency:
    - llama_midi >= 0.0.1
    no matching version

[NOTE] Pinning command successful, but your installed packages may be out of sync.
```
Same problem as before, although now it prints "Pinning command successful" as
if to taunt me when it very clearly did not work.

I stared at this for a little while before I noticed that the `llama`,
`llama_core` and `llama_interactive` packages were all version `0.0.1` while
`llama_midi` and `llama_tests` are version `~dev`. In Opam, you don't mark local
.opam files as representing a particular version of a package. The only place
where version numbers of packages are recorded is the Opam package repository.
So in order to know about version `0.0.1` it must be checking the versions of
those packages in the Opam repo.

At the time, only `llama`, `llama_core` and `llama_interactive` had been
released to the Opam repository. `llama_midi` was a new package that had been
added since the most-recent release of the llama packages and it's now a
dependency of the `llama_core` package:
```
depends: [
   ...
  "llama_midi" {= version}
]
```

In a classic Jar Jar move, Opam checked whether each of the local packages had
also been released to the Opam repo and for the ones that had, it acted like the
local packages had the version numbers corresponding to the latest releases for
the purposes of solving dependencies. This meant that while resolving the
dependencies of `llama_core` version `0.0.1` it tried to find `llama_midi`
version `0.0.1` (which does not exist) since the `llama_midi {= version}` in
`llama_core`'s dependencies means the version of `llama_midi` which is the same
as the installed version of `llama_core`.

We can get Opam to stop trying to be all helpful about package versions by using
the `--with-version` option to force a particular version of all the packages it
pins. Since they'll all be given the same fictional version, you can pass
whatever you like to that argument:
```
opam pin . --with-version sigh
```

This still doesn't work.

It installed dependencies, but it fails to build `llama`'s Rust dependencies:

```
#=== ERROR while compiling llama.sigh =========================================#
# context     2.1.5 | macos/arm64 | ocaml-base-compiler.4.14.1 | pinned(git+file:///.../llama#main#06deea42efa6b84653be43529daf8aa08dc106
68)
# path        /.../_opam/.opam-switch/build/llama.sigh
# command     ~/.opam/opam-init/hooks/sandbox.sh build dune build -p llama -j 7 @install
# exit-code   1
# env-file    ~/.opam/log/llama-60822-042f19.env
# output-file ~/.opam/log/llama-60822-042f19.out
### output ###
# error: failed to get `anyhow` as a dependency of package `low_level v0.1.0 (.../_opam/.opam-switch/build/llama.sigh/_build/default/src
/low-level/low-level-rust)`
# [...]
# Caused by:
#   failed to query replaced source registry `crates-io`
#
# Caused by:
#   download of config.json failed
#
# Caused by:
#   failed to download from `https://index.crates.io/config.json`
#
# Caused by:
#   [7] Couldn't connect to server (Failed to connect to index.crates.io port 443 a
fter 0 ms: Couldn't connect to server)
```

It tried to build the  `llama` package in the Opam sandbox (with no internet
access, remember) and obviously this doesn't work when building the package from
the repo as I don't check-in the vendored Rust dependencies. Building the rust
component meant that cargo tried to download dependencies but couldn't due to the lack of
internet access in the Opam sandbox.

I don't even want to build or install the `llama` package though - just its dependencies! Unlike `opam
install . --deps-only`, the `opam pin .` command also installs the local
packages - not just their dependencies. There is no `--deps-only` flag for `opam
pin` and no `--with-version` for `opam install` so as far as I can tell there's
actually no way to just install the dependencies of local packages when you have
some unreleased local dependencies.

I guess I just have to wait until `llama_midi` gets released.

The experience of setting up dependencies of a fresh clone of a project is
actually something that is set to improve with an upcoming dune feature. Dune is
getting support for package management and lockfiles so pretty soon it will be a
single `dune` command to install and build all the dependencies of an OCaml
project.
