---
title: "Designing a Data-Oriented Compiler, Part 1"
date: 2026-07-13T11:22:54Z
tags: ['compiler', 'data-oriented design', 'go', 'kiwi']
description: "An introduction to a new series on building a data-oriented compiler, using the Kiwi compiler as the running example."
slug: "designing-a-data-oriented-compiler-part-1"
#draft: true
---

## Introduction
I have been writing compilers long enough to recognize my default instincts. I reach for rich AST nodes, interface-heavy abstractions, pointer-linked structures, and visitor passes because they are familiar, expressive, and great for getting a prototype off the ground quickly.

What I have learned, though, is that those defaults quietly shape everything that follows. A compiler spends most of its life walking and rewriting structured data: tokens, syntax nodes, symbol tables, types, intermediate values, instructions, and diagnostics. When those representations are scattered across the heap, every later pass pays for it in locality, indirection, and complexity. Chandler Carruth makes this case directly in his CppCon 2023 talk, [*Modernizing Compiler Design for Carbon Toolchain*](https://www.youtube.com/watch?v=ZI198eFghJk), where he argues that data layout should be treated as a first-order architectural concern. Andrew Kelley's [*A Practical Guide to Applying Data-Oriented Design*](https://www.youtube.com/watch?v=IroPQ150F6c) makes the same point from the hardware side: if your access patterns fight the machine, the machine wins.

That shift in perspective is what pushed me to start **Kiwi**, a small C-like systems-language compiler in Go, with a more data-oriented mindset. Instead of asking only "what objects do I need?", I want to ask "what data will each phase touch, in what order, and how stable should that representation remain?" For example, a semantic pass that resolves thousands of identifiers is usually better served by dense tables of interned names and compact node IDs than by a forest of heap-allocated AST objects linked together by pointers.

The phrase "data-oriented" gets used a lot, sometimes loosely, so it is worth being precise about what I mean here. A data-oriented compiler is one that treats its internal representations as first-class design problems. Instead of defaulting to pointer-heavy trees, object graphs, and ad hoc maps everywhere, it pays close attention to layout, traversal patterns, stability of identifiers, and how information flows from one phase to the next.

This is the sense in which I use *data-oriented* throughout the series. I do not mean "prematurely optimize everything" or "ban pointers on principle." I mean choosing layouts, IDs, and traversal patterns that make the compiler easier to reason about, easier to evolve, and, as a result, often faster too.

## Where object-oriented compiler design starts to creak

I do not think object-oriented techniques are a mistake. They are often the most natural way to get a compiler working. A `BinaryExpr` node with `left`, `right`, and `op` fields is easy to understand. A visitor interface gives every pass a familiar entry point. Encapsulating behavior on the nodes themselves can make a prototype feel tidy.

The problem is that production compilers rarely behave like object-oriented business software. They do not spend most of their time sending rich messages to long-lived objects. They spend it scanning, classifying, rewriting, and correlating large volumes of structured data. That is exactly where pointer-heavy object graphs start to work against the machine rather than with it, a point Mike Acton emphasizes in his talk [*Data-Oriented Design and C++*](https://www.youtube.com/watch?v=rX0ItVEVjHc), and Chandler Carruth applies directly to compiler implementation in [*Modernizing Compiler Design for Carbon Toolchain*](https://www.youtube.com/watch?v=ZI198eFghJk).

In compiler code, the shortcomings usually show up in a few predictable ways:

1. **Too much indirection.**  
   A pass that wants to inspect ten thousand expressions may end up chasing ten thousand pointers just to read a node kind, a token span, and two child references. The code may look elegant, but the access pattern is scattered.

2. **Identity gets tied to addresses.**  
   If later phases refer to syntax, symbols, or IR values by pointer, then ownership, mutation, serialization, and incremental updates all become harder. Stable integer IDs are often less glamorous, but they are easier to compare, store, log, and move across phase boundaries.

3. **Behavior becomes easier to add than structure.**  
   In an OOP design, it is tempting to keep attaching more methods, visitors, and auxiliary maps to the same node hierarchy. Over time, the real compiler state ends up spread across the AST, side tables, caches, and pass-local bookkeeping, which makes it harder to answer a simple question like: "what data actually defines this phase?"

4. **Hot and cold data get mixed together.**  
   A type-checking pass may only need a node kind, an interned name, and a source span, but a rich node object often carries much more than that. When frequently accessed fields live beside rarely used ones, every traversal pulls extra baggage through memory.

To make that concrete, imagine a name-resolution pass over a file with many local bindings. In a conventional AST, each identifier use might be a heap object containing text, span information, child links, methods, and pass-specific annotations. In a data-oriented layout, the same pass might instead scan a dense slice of node records, look up interned name IDs in a symbol table, and write resolved symbol IDs into a parallel side table. Both versions are valid. The latter simply matches the actual workload more closely.

## What "data-oriented" means in practice

In this series, I will use *data-oriented* in a practical, compiler-specific sense.

It means asking questions like:

1. **What is the unit of storage?**  
   Do we store syntax as linked nodes, flat arrays, or a mixture of both? For example, a parser might append nodes to a contiguous post-order tree instead of allocating every expression separately.

2. **What is the unit of identity?**  
   Do phases refer to objects by pointers, names, or compact IDs? A `TypeID`, `NodeID`, or `ValueID` can survive reordering and makes cross-phase references much easier to reason about than raw addresses.

3. **What is the expected access pattern?**  
   Will a pass mostly scan linearly, jump randomly, or group work by kind? A liveness pass, for instance, often wants sequential block and instruction traversal, while an interning table wants fast keyed lookup.

4. **What state needs to remain stable across phases?**  
   Can diagnostics, symbols, and values refer to the same file and entity identifiers throughout the pipeline? Stable IDs are especially useful when you want error reporting, caching, or editor tooling to point at the same entities consistently.

5. **What becomes simpler when layout is explicit?**  
   Debugging, serialization, testing, incremental recompilation, and instrumentation all benefit when the internal model is regular. If your IR lives in pools keyed by IDs, dumping it, hashing it, or diffing it becomes much more straightforward.

This is not about chasing micro-optimizations from day one. It is about choosing representations that reflect the work a compiler actually does. Andrew Kelley makes this practical point well in [*A Practical Guide to Applying Data-Oriented Design*](https://www.youtube.com/watch?v=IroPQ150F6c): the win is not just speed, but clarity about what data exists, how it moves, and what each pass really touches.

## Why this approach is appealing for compilers

Compilers are a particularly good fit for data-oriented design because so much of their work is repetitive, batch-oriented, and phase-structured.

The lexer scans bytes. The parser appends syntax. Name resolution walks declarations and uses. Type checking annotates existing structure. Lowering turns one representation into another. Optimization passes sweep over blocks, instructions, and values. These are workloads built around traversal, classification, and transformation, not around polymorphic object behavior.

That has a few practical consequences:

- **The implementation becomes easier to reason about phase by phase.** If each phase reads from a few well-defined tables and writes to a few others, the architecture becomes easier to inspect and debug.
- **Performance work has a clearer starting point.** When data is laid out explicitly, it is easier to measure hot paths, reduce unnecessary allocation, and improve locality without redesigning the whole compiler.
- **Tooling falls out more naturally.** Stable IDs and regular storage make it easier to dump IR, attach diagnostics, build indexes, or support editor features that need durable references.
- **Incremental and persistent workflows become more plausible.** Once entities have stable names and compact identities, caching and invalidation logic have something concrete to hold on to.

None of this means every compiler should be purely columnar or pointer-free. Some problems are genuinely easier to model with trees, graphs, or rich wrapper types. The point is to be deliberate. Use indirection where it earns its keep, not just because it is the default shape of an object-oriented design.

## The running example: Kiwi

Kiwi is the compiler I will use as the running example for this series. It is a small compiler for a C-like systems language, implemented in Go, with a fairly standard pipeline:

- a frontend for tokenizing and parsing source files,
- semantic analysis for name resolution and type checking,
- an IR layer for lowered program structure,
- backend stages for later transformation and code generation,
- and shared support systems for source management, diagnostics, and interning.

What makes Kiwi useful for this discussion is not that it does anything magical. It is useful because it is small enough to inspect end to end, yet large enough for representation choices to matter.

Several of Kiwi's design choices already lean in a data-oriented direction:

- **Interned names and literals** keep repeated strings out of the hot path and let later stages pass around compact identifiers instead of heap-allocated text.
- A **flat parse tree** reduces dependence on pointer chasing and makes whole-tree traversal more sequential.
- IR values live in a **value pool** keyed by stable IDs instead of being spread across a large object graph.
- Instructions are grouped into **typed pools**, which makes it easier for later passes to iterate over specific operation classes.
- Shared systems such as the **source manager** and **diagnostic engine** give every phase the same view of files, spans, and errors.

Taken individually, none of these choices is especially radical. What interests me is how they reinforce each other. Once names are interned, spans are stable, nodes have IDs, and IR values live in pools, the compiler starts to feel less like a web of objects and more like a pipeline over explicit data.

## What I plan to cover

My current outline for the series looks like this:

1. **Trade-offs in object-oriented compiler structures**  
   Where rich node hierarchies help, and where they start to get in the way.
2. **Interning names, literals, and strings**  
   Why compact identifiers are often better than passing raw strings across phases.
3. **Representing syntax with flat trees**  
   How a post-order tree or similar layout changes traversal and ownership.
4. **Building an IR around pools and stable IDs**  
   Why values and instructions benefit from regular storage.
5. **Diagnostics, spans, and source management**  
   Keeping errors precise without tangling the rest of the compiler.
6. **Pass pipelines and backend staging**  
   Organizing transformations so the compiler remains extensible.
7. **Incremental compilation and tooling implications**  
   What becomes easier once representations are stable and explicit.

That outline will probably change as Kiwi changes, which is part of the point. I want the series to stay close to the code instead of pretending the implementation is finished before it exists.

## What comes next

In the next post, I will look at one of the earliest representation decisions in a compiler: how to model syntax without committing too early to a pointer-rich AST. Kiwi already leans toward a flatter representation, and that makes it a good starting point for discussing how data layout influences nearly every phase that follows.

If the series goes well, it should end up being less about "how to write a compiler in Go" and more about how to **structure compiler data so the implementation stays tractable as the language grows**. That is the part I am most interested in, and it is the thread I plan to follow from here.
