# Jenny
Jenny is an experimental [opinionated](https://github.com/victor-smirnov/digital-philosophy) imperative statically typed programming language and metaprogramming [transpiler](https://en.wikipedia.org/wiki/Source-to-source_compiler) on top of and for C++ and [Memoria](https://bitbucket.org/vsmirnov/memoria/wiki/Home). Currently, this project is meant to be mostly a showcase for Memoria framework.

## Rationale
C++ has some painful points which are very hard to fix given the necessity to maintain compatibility with existing codebase. Jenny is a new language with the spirit of C++ (low level, zero-cost abstractions) but without its ancient legacy from 1980ths. Simple things should be simple. But complex things should be simple too and only [advanced metaprogramming](https://en.wikipedia.org/wiki/Metaclass) will save us from a curse of descriptional complexity. What we definitely don't need is a new language competing with C++ that is not compatible with its existing codebase. Jenny will be able to use most of existing C++ libraries with minimal efforts. And some Jenny programs can be exported as C++ projects to be used with C++ applications at the source level, including template metaprogramming. So folks currently heavily investing in C++ should continue doing this. Jenny objects are C++ objects and vice versa, no bridging is necessary.

## Core Features

1. Extensible syntax for streamlined [DSL](https://en.wikipedia.org/wiki/Domain-specific_language) integration.
1. Native [SDN](https://bitbucket.org/vsmirnov/memoria/wiki/String%20Data%20Notation) integration for in-code embedded data structures and DSLs.
1. Type-level metaprogramming in large. Like C++ templates but with human-friendly syntax and debugging support.
1. Metaclasses and metaobjects. Compile-time AST-level metaprogramming (in large).
1. Lifetimes. Like Rust's borrowing semantics but much more flexible.
1. Native Fibers and Coroutines for high-performance IO.
1. Native language support for Tuples, Variants and Pattern Matching.
1. Native integration with Memoria runtime (high-performance AIO subsystem, custom memory management), Linked Data model, containers and stores.
1. [AOP](https://en.wikipedia.org/wiki/Aspect-oriented_programming) and Contracts. The more formal semantics is made explicit, the better for refactoring.
1. Language profiles to enable/disable certain language features: Safe (UB-free + memory protection) language subset (by default) for general-purpose applications development, Full profile for ninjas, AOT-only profile, JIT-requiring profile, etc.
1. Metaprogramming-aware refactoring tools. Code is alive while it can continuously evolve.
1. Modules.
1. Integrated package manager.
1. Clang parser integration for interoperability with C++.
1. Versioned (DVCS-like) Memoria store (in-memory, on-disk) as a physical program structure (instead of a bunch of a plain text files). Code and embedded data are in the same place, reachable with transpiler and refactoring tools.
1. IDE-friendly event-driven dataflow-based transpiler architecture.
1. Embedded HTTP server providing native RESTful API for transpiler, refactoring tools, interactive auto-completion, code and data (language server).
1. Basic Web application for code and data navigation and editing, using the transpiler's REST API.
1. Infrastructure-aware transpiler/refactoring. Want to split your monolith into well-sized services or vice versa? Jenny will do it for you!

## Highlights

Jenny, as a programming language, is focused on data structure design and code-as-data metaprogramming. Though every program can be written in a classical textual form, physical Jenny program structure is a versioned multimodel database of parsed abstract syntax trees for Jenny and embedded DSLs, together with associated embedded data structures (annotation metadata, documentation, i18n databases, type traits, etc). The database is dynamically updatable and optimized for analytics, with [MVCC](https://en.wikipedia.org/wiki/Multiversion_concurrency_control) and branching semantics. Structured data representation makes code analytics much simpler and faster. More on this later.

Jenny is not a compiler, its a source-to-source transpiler producing C++ code as an output (together with other artifacts like Protobuf interfaces and SWIG bindings), either for direct compilation into an executable or a C++ library module to use with other C++ applications. Jenny is not a competition or replacement for C++, but a *legacy-free* modernization for better productivity. The language will be co-evolving with C++, stacking on top of its semantics and a huge set of libraries. Codebase matters and C++ has it. 

The transpiler will be developing using service-oriented cloud-friendly asynchronous data-flow model, with public API and interactive Web interfaces. Classical file-in/file-out command-line mode will also be provided for integration with existing build tools. Currently, Memoria provides only in-memory and on-disk (single-machine) storage options, but this is more than enough for typical use case of such transpiler. Interested integrators can implement cloud-native storage option for Memoria for themselves and reimplement the transpiler at the scale of the Cloud.

## Metaprogramming in Jenny

Jenny is based on a (pretty large) subset of C++20 syntax with multiple extensions to support core features like pattern matching, fibers and coroutines, metaclasses and metafunctions, annotations, embedded SDN, and so on. The main difference is that together with template metaprogramming and compile-time computations via *constexpr* functions, there are dedicated type-level and ast-level *metafunctions*. They look and feel like normal functions but invoked by the tranpiler at the compile time and consume types as value objects and produce types and AST framgments as value objects. Something like that:

```c++
// Some template
template <int... Values> struct IntList {};

// The metafunction. Annotetion informs the transpiler that this function 
// will be used to create types at copile time.
@Metafunction
Type CreateIntListType(int size) 
{
  // Here metaprogramming goes. We are creating the type from template
  // by specifing its prameters. Both types and templates are regular
  // objects here. 
  Template tpl(IntList);
  auto builder = tpl.builder();
  for (int c = 0; c < size; c++)
  {
    builder.addValue(c);
  }
  
  return builder.build();
}

// Declaring a type using metafunction. Will be invoked at compile time.
using IntListOf10 = CreateIntList(10);
```

Populating the IntList template with a sequence of numbers can be easily done with good old template metaprograms, and Jenny will be supporting the most of current template metaprogramming (TMP) out of the box. So, C++ libraries like Boost Hana should work pretty well in Jenny programs (modulo, possibly, preprocessor metaprogramming). But this way of building types [does not scale](https://en.wikipedia.org/wiki/Turing_tarpit) beyond simple cases:
1. Debugging support to template metaprograms is fairly limited.
1. Only linked-list based data structures are possible, so any access operation is O(N).
1. The size of such data structures is pretty small.
1. It's very hard to enforce any schema on TMP-based arbitrary data structures. Concepts to the rescue but we need much more scalability. 

Classical TMP in Jenny is possible, it's not deprecated, but recommended only for simple, immediately obvious and convenient cases. Everything non-trivial is encouraged to use metafunctions.

Metafunctions are executed at compile time. In his way, they are somewhat similar to constexpr functions, but can use almost entire set of C++ lanuage feasures and most of the API. Metafunctions can read and write files, databases and network. They can create types by template instantialtions and any code by direct AST construction. Metafunctions are not limited with amount of data they can process. 

It may seems highly unreasonable why a metaprogram needs to read an arbitrary file? For example, for type metadata (type traits), when the structure of traits [goes beyond simple key/value model](https://github.com/victor-smirnov/memoria/blob/master/include/memoria/core/datatypes/traits.hpp). If type traits are hierarchically structured, it's higly error-prone to describe them as nested C++ type lists (via variadic templated). It doesn't scale beyond simple cases. Complex metadata should better be stored in the more suitable formats for that, like templatized [SDN](https://bitbucket.org/vsmirnov/memoria/wiki/String%20Data%20Notation) documents.

Another use case may be from web development. [JSX](https://www.w3schools.com/react/react_jsx.asp) is an example of DSL merging fluently imperative JavaScript code and declarative HTML fragments. Today C++ is defenitely not a web developer's choice, but Jenny may have a chance, because it can integrate many different resources into a consistent logical model of the application.



## Using Memoria for Code Model

Memoria is a C++ metaprogramming framework for general-purpose copy-on-write based [persistent data structures](https://en.wikipedia.org/wiki/Persistent_data_structure) on top of key-value memory model. There are two basic complex data "formats":
* B-Tree-based dynamic versioned data *containers* and
* Pointer-based LinkedData *documents* allocated compactly in arenas with predictable binary data layout.

Containers are versioned and may have arbitrary size. Documents have value semantics and can be stores as data values in containers. They may also have arbitrary binary data size, but optimized to be small. LinkedData documents are similar to JSON, but unlike the latter, the former support much more data types natively, besides strings, numbers, maps and arrays. LinkedData documents are immediately quaryable, no data type conversion is necessary for processing. They are nearly ideal storage format for parsed abstract syntax trees.

Versioning in Memoria is similar to (D)VCS. Containers can be updated, and updates are grouped to snapshots (commits). Once a snapshot is committed, it became immutable. All further updates will require creating a new version. Different versions can be merged by moving specific updates from existing versions to a new one. The main difference from DVCS is that *all versions are materialized and available for reading simultaneously*. No "checkouting" is necessary. This enables various cross-version source code analysis techniques. Because of [snapshot isolation](https://en.wikipedia.org/wiki/Snapshot_isolation), code analyzers can be run as long as they need, incoming incremental updates do not affect the code they are currently operating with. Moreover, writers do not affect readers in any way. Once snapshot reference is obtained from the version history, all further data access is mostly lockless (modulo short-term locks in a cache and in the OS kernel). 

But such powerful functionality comes with cost. Like in many DVCS, versions in Memoria are stored as a difference between current and "parent" versions. But this difference is stored as a set ob updated blocks (i.e. materialized form of update), which are typically of the size between 4KB and 1MB. Because of that, versions in Memoria are much larger than versions in DVCS. Fortunately, for typical Jenny's usecases (large code bases) storage space is not an issue. Compiled artifacts (object code, intermediate files etc) require way more storage space than source codes.

Jenny's *Code Model* is a set of specialized Memoria containers, LinkedData types, domain-specific query languages and related logic implementing transpiler workflow, parsing ASTs for Jenny, C++ and various embedded DSLs, synchronization with other (remote) repositories, various auxiliary functions and data types supporting code analyzers and refactoring. 

## Roadmap
TBD
