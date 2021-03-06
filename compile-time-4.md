# A few more reasons Rust compiles slowly

![header image](https://brson.github.io/tmp/calamity-header.jpg)

The Rust programming language compiles fast software slowly.

In this series we explore Rust's compile times within the context of [TiKV], the key-value store
behind the [TiDB] database.

[TiKV]: https://github.com/tikv/tikv
[TiDB]: https://github.com/pingcap/tidb

&nbsp;


## Rust Compile-time Adventures with TiKV: Episode 4

Lately we're exploring how Rust's designs discourage fast compilation.
In [the previous post in the series][prev] we discussed compilatio units,
why Rust's are so big, and how that affects compile times.

This time we're going to wrap up discussing _why_ Rust is slow
with a few more subjects: LLVM, compiler architecture, and linking.

[prev]: TODO


- [LLVM and poor LLVM IR generation](#user-content-llvm-and-poor-llvm-ir-generation)
- [Batch compilation](#user-content-batch-compilation)
- [Build scripts and procedural macros](#user-content-build-scripts-and-procedural-macros)
- [Static linking](#user-content-static-linking)
- [A summary](#user-content-a-summary)
- [In the next episode of Rust Compile-time Adventures with TiKV](#user-content-in-the-next-episode-of-rust-compile-time-adventures-with-tikv)
- [Thanks](#user-content-thanks)


## Tradeoff #4: LLVM and poor LLVM IR generation

`rustc` uses LLVM to generate code. LLVM can generate very fast code, but it
comes at a cost. LLVM is a very big system. In fact, LLVM code makes up the
majority of the Rust codebase. And it doesn't run particularly fast. In fact,
the most recent release of LLVM [caused significant regressions to Rust's compile time][rcr1]
for no particular benefit.

The Rust compiler needs to keep up with LLVM releases though to avoid painful maintainance problems,
so Rust builds are [soon going to be pointlessly but unavoidably slower][rcr2].

[rcr1]: https://lists.llvm.org/pipermail/llvm-dev/2020-May/141482.html
[rcr2]: https://github.com/rust-lang/rust/pull/67759

Even when `rustc` is generating debug builds, that are supposed to build fast, but are allowed to run slow, generating the machine code still takes a considerable amount of time.

In a TiKV release build, LLVM passes occupy 84% of the build time, while in a debug build LLVM passes occupy 35% of the build time ([full details gist][tpg]).

[tpg]: https://gist.github.com/brson/ba22165d6da4a976b278b0896db7e4e4

LLVM being poor at quickly generating code (even if the resulting code is slow) is not in intrinsic property of the language though, and efforts are underway to create a second backend using [Cranelift], a code generator written in Rust, and designed for fast code generation.

[Cranelift]: https://github.com/bytecodealliance/cranelift

In addition to being integrated into `rustc` Cranelift is also being integrated into SpiderMonkey as its WebAssembly code generator.

It's not fair to blame all the code generation slowness on LLVM though. `rustc` isn't doing LLVM any favors by
the way it generates LLVM IR.

`rustc` is notorious for throwing huge gobs of unoptimized LLVM IR at LLVM and expecting LLVM to optimize it all away. This is (probably) the main reason Rust debug binaries are so slow.

So LLVM is doing a lot of work to make Rust as fast as it is.

This is another problem with the compiler architecture &mdash; it was just easier to create Rust by leaning heavily on LLVM than to be clever about how much info `rustc` handed to LLVM to optimize.

So this can be fixed over time, and it's one of the main avenues the compiler team is pursuing to improve compile-time performance.

Remember a few episodes ago when we discussed how monomorphization works? How it duplicates function definitions for every combination of instantiated type parameters? Well, that's not only a source of machine code bloat but also LLVM IR bloat. Every one of those functions is filled with duplicated, unoptimized, LLVM IR.

`rustc` is slowly being modified such that it can perform its own optimizations on its own MIR (mid-level IR), and crucially, the MIR representation is pre-monomorphization. That means that MIR-level optimizations only need to be done once per generic function, and in turn produce smaller monomorphized LLVM IR, that LLVM can (in theory) translate faster than it does with its unoptimized functions today.


## Batch compilation

It turns out that the entire architecture of `rustc` is "wrong", and so is the architecture of most compilers ever written.

It is common wisdom that all compilers have an architecture like the following:

- the compiler consumes an entire compilation unit by parsing all of its source code into an AST
- through a succession of passes, that AST is refined into increasingly detailed "intermediate representations" (IRs)
- the entire final IR is passed through a code generator to emit machine code

This is the batch compilation model. This architecture is vaguelly how compilers have been described academically for decades; it is how most compilers have historically been implemented; and that is how `rustc` was originally architected. But it is not an architecture that well-supports the workflows of modern developers and their tooling, nor does it support fast recompilation.

Today, developers expect instant feedback about the code they are hacking. When they write a type error, the IDE should immediately put a red squiggle under their code and tell them about it. It should ideally do this even if the source code doesn't completely parse.

The batch compilation model is poorly suited for this. It requires that entire compilation units be re-analyzed for every incremental change to the source code, in order to produce incremental changes to the analysis. In the last decade or so, the thinking among compiler engineers about how to construct compilers has been shifting from batch compilation to "responsive compilation", by which the compiler can run the entire compilation pipeline on the smallest subset of code possible to answer a particular question, as quickly as possible. For example, with responsive compilation one can ask "does this function type check?", or "what are the type dependencies of this structure?".

This ability lends itself to the _perception_ of compiler speed, since the user is constantly getting necessary feedback while they work. It can dramatically shorten the feedback cycle for correcting type checking errors, and in Rust, getting the program to successfully type check takes up a huge proportion of developers' time.

I imagine the prior art is extensive, but a notable innovative responsive compiler is the [Roselyn] .NET compiler; and the concept of responsive compilers has recently been advanced significantly with the adoption of the [Language Server Protocol][lsp]. Both are Microsoft projects.

In Rust today we support this IDE use case with the [Rust Language Server][RLS] (the RLS). But many Rust developers will know that the RLS experience can be pretty disappointing, with huge latency between typing and getting feedback. Sometimes the RLS fails to find the expected results, or simply fails compeletely. The failures of the RLS are mostly due to being built on top of a batch-model compiler, and not a responsive compiler.

The RLS is gradually being supplanted by [`rust-analyzer`], which amounts to essentially a ground-up rewrite of `rustc`, at least through its analysis phases, to support responsive compilation. It's expected that over time [`rust-analyzer` and `rustc` will share increasing amounts of code][libraryification].

[`rust-analyzer`]: https://github.com/rust-analyzer/rust-analyzer
[libraryification]: http://smallcultfollowing.com/babysteps/blog/2020/04/09/libraryification/

Taken to the limit, a responsive compiler architecture naturally lends itself to quickly responding to requests like "regenerate machine code, but only for functions that are changed since the last time the compiler was run". So not only does responsive compilation support the IDE analysis use case, but also the recompile-to-machine-code use case. Today, this second use case is supported in `rustc` with "incremental compilation", but it is fairly crude, with a great deal of duplicated work on every compiler invocation. We should expect that, as `rustc` becomes more responsive, incremental compilation will ultimately do the minimal work possible to recompile only what must be recompiled.

There are though tradeoffs in the quality of machine code generated via incremental compilation &mdash; due to the mysterious challenges of inlining, incrementally-recompiled code is unlikely to ever be as fast as highly-optimized, batch-compiled machine code. In other words, you probably won't ever want to use incremental compilation for your production releases, but it can drastically speed up the development experience, while producing relatively fast code.

Niko spoke about this architecture in his ["Responsive compilers" talk at PLISS 2019][resp]. In that talk he also provided some examples of how the Rust language was accidentally mis-designed for responsive compilation. It is an entirely watchable talk about compiler engineering and I recommend checking it out. 

[resp]: https://www.youtube.com/watch?v=N6b44kMS6OM
[RLS]: https://github.com/rust-lang/rls
[Roselyn]: https://en.wikipedia.org/wiki/.NET_Compiler_Platform
[lsp]: https://langserver.org/



## Build scripts and procedural macros

Cargo allows the build to be customized with two types of custom Rust programs: build scripts and procedural macros. The mechanism for each is different but they both similarly introduce arbitrary computation into the compilation process.

They negatively impact compilation time in several different ways:

First, these types of programs have their own crate ecosystem that also needs to be compiled, so using procedural macros will usually require also compiling crates like [`syn`].

Second, these tools are often used for code generation, and their invocation expands to sometimes large amounts of code.

Third, procedural macros impede distributed build caching tools like [`sccache`]. The reason why surprised me though &mdash; rustc today loads procedural macros as dynamic libraries, one of the few common uses of dynamic libraries in the Rust ecosystem. `sccache` isn't able to cache dynamic library artifacts because it doesn't have visibility into how the linker was invoked to create the dynamic library. So using sccache to build a project that heavily relies on procedural macros will often not speed up the build.

[`syn`]: https://crates.io/crates/syn
[`sccache`]: https://github.com/mozilla/sccache


## Static linking

This one is easy to overlook but has potentially significant impact on the
hack-test cycle. One of the things that people love most about Rust &mdash; that
it produces a single static binary that is trivial to deploy, also requires the
Rust compiler to do a great deal of work linking that binary together.

Every time you build an executable `rustc` needs to run the linker. That
includes every time you rebuild to run a test. In the same experiment I did to
calculate the amount of build time spent in LLVM, Rust spent 11% of debug build
time in the linker. Surprisingly, in release mode it spent less than 1% of time
linking, excluding LTO.

With dynamic linking the cost of linking is deferred until runtime, and parts
of the linking process can be done lazily, or not at all, as functions are
actually called.


## A summary

We're four episodes into a series that originally was supposed to be about
speeding up TiKV compilation, but so far we've mostly complained in great depth
about Rust's compile times.

There are a great many factors involved in determining the compile-time of a
compiler, and the resulting run-time performance of its output. It is a miracle
that optimizing compilers ever terminate at all, and that their resulting code
is so amazingly fast. For humans, predicting how to organize their code to find
the right balance of compile time and run time is pretty much impossible.

The size and organization of compilation units has a huge impact on compile
times, and in Rust it is difficult to control compilation unit size, and
difficult to create compilation units that can build in parallel. Internal
compiler parallelism currently does not make up for loss of parallelism between
compilation units.

A variety of factors cause Rust to have a poor build-test cycle, including the
generics model, linking requirements, and the compiler architecture.


## In the next episode of Rust Compile-time Adventures with TiKV

In the next episode of this series we'll do an experiment to illustrate the tradeoffs between dynamic and static dispatch in Rust.

Or maybe we'll do something else. I don't know yet.

Stay Rusty, friends.


## Thanks

A number of people helped with this blog series. Thanks especially to Ted Mielczarek for their insights, and Calvin Weng for proofreading and editing.
