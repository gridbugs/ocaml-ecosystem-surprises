# OCaml Ecosystem Surprises

- linking against native libraries
- os-specific linker arguments
- inline testing
- loading a wav file with mm
- ocaml-rs doesn't support ocaml5
- ocaml-rs float32 arrays are broken
- opam install and opam pin don't handle local packages in some cases
- dune sourcetree ignores directories begining with "."

I recently made my first non-trivial OCaml project - a synthesizer library
called [llama](https://github.com/gridbugs/llama/). It's made up of multiple
Opam packages, it links against native code, it calls into code in a foreign
language (Rust), it's (somewhat!) documented, it works on MacOS and Linux, it's
released to the Opam repository. The language ecosystem I've spent the most time
in is Rust/Cargo and as such I'm accustomed to things "just working". This is a
post about the times while developing my library that I was surprised or
confused or frustrated by something in the OCaml ecosystem.

## Linking Against Native Libraries

My library makes noise by sending samples to the sound card
via a small Rust library that uses
[cpal](https://crates.io/crates/cpal) to talk to the audio driver and
[ocaml-rs](https://crates.io/crates/ocaml) for OCaml interop. The rust code is compiled to an archive
`liblow_level.a` and I set up the dune project for the library based on
[`ocaml-rs`'s docs](https://zshipko.github.io/ocaml-rs/).

My dune file looked like this:
```
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

I only have a static archive of the Rust library - not a shared object (.so)
file. Searching the [dune (library ...) stanza
documentation](https://dune.readthedocs.io/en/stable/dune-files.html#library)
for the word "dynamic" I found:

> `(no_dynlink)` disables dynamic linking of the library. This is for advanced use
> only. By default, you shouldn’t set this option.

That's a bit of a scary looking message as this doesn't feel like I'm trying to
do anything too advanced but it does seem to fix the problem:

```
(library
 (name llama_low_level)
 (no_dynlink)
 (foreign_archives low_level)
 (c_library_flags
  (-lpthread -lc -lm)))
```

Next I made a small executable to test calling a function in the
`llama_low_level` library:

```
(executable
 (public_name experiment)
 (libraries llama_low_level))
```

Building this caused my terminal filled with errors. Here's the start of the
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
the `AudioToolbox` framework. So foreign archive must depend on some frameworks
on MacOS for doing "audio stuff" and I need to tell the linker about it.
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

Reading through `dune`'s library docs again and there's this field:
> `(library_flags (<flags>))` is a list of flags passed to ocamlc and ocamlopt
> when building the library archive files.

Sounds promising. Now we need a way to get `ocamlc` to pass custom flags to the
linker. `man ocamlc` contains:
```
-cclib -llibname
       Pass the -llibname option to the C linker when linking in "custom runtime"
       mode  (see  the  -custom  option).  This  causes the given C library to be
       linked with the program.

-ccopt option
       Pass the given option to the C compiler and linker, when linking in  "cus‐
       tom  runtime"  mode  (see  the -custom option). For instance, -ccopt -Ldir
       causes the C linker to search for C libraries in directory dir.
```

Putting these together:
```
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

```
(library_flags
 (-ccopt "-framework CoreServices -framework CoreAudio -framework AudioUnit -framework AudioToolbox))
```

Now it builds!

But only on MacOS.

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
explicitly passed to the linker.

On Linux I need to generate the linker flags from a command rather than
hard-coding them in the dune file. The only way to do this in dune is to
generate a text file containing an [S-expression
(sexp)](https://en.wikipedia.org/wiki/S-expression) representation of a list of
arguments and then using dune's `:include` keyword to load a list of arguments
from the file.

So I need to generate a file with the following contexts:
```
("-cclib" "-L/nix/store/qnf36msgsjh17sy9dakvqnvv7sgr8dfg-alsa-lib-1.2.9/lib -lasound")
```
Note the opening and closing parentheses. These are required as the file must
contain a sexp. Also this time around we do have to pass `-cclib` as `-ccopt`
doesn't work. I guess because we are passing the name of a library to link
against?

To generate this file we can use a dune rule:
```
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

This works on Linux but of course now it won't build on MacOS as we removed the
MacOS-specific linker flags and replaced them with the Linux ones. To work in
general (well just on MacOS and Linux) we can use dune's `(enabled_if ...)`
field to use a different rule to generate `library_flags.sexp` on MacOS.

```
(rule
 (enabled_if
  (= %{system} macosx))
 (action
  (write-file
   library_flags.sexp
   "(\"-cclib\" \"-framework CoreServices -framework CoreAudio -framework AudioUnit -framework AudioToolbox\")")))

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
Note that the system name for MacOS is "macosx". There's an "x" at the end.
Don't forget the "x" or the comparison will always be false.

Also note the third rule which creates an empty sexp file in the case that the
system is neither Linux nor MacOS. This is so that if someone deigns to build
this on a non-Linux non-MacOS system they don't get an error about the
`library_flags.sexp` file being missing (instead they will probably get some
sort of linker error).

This is all a bit of a mess. It's awkward to construct sexp files by manually
adding parentheses around the output of commands and adding the right level of
escaping quotes, and the catch-all rule for non-Linux non-MacOS systems has to
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
```
(executable
 (name discover)
 (libraries dune-configurator))
```

Now the dune file for `low_level` can run `discover.exe` in a single rule rather
than having a separate rule for each system:
```
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
