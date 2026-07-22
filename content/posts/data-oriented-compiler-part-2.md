---
title: "Designing a Data-Oriented Compiler, Part 2"
date: 2026-07-22T11:36:05Z
tags: ['compiler', 'data-oriented design', 'go', 'kiwi']
description: "A closer look at the trade-offs in object-oriented compiler structures, and why rich node hierarchies eventually start to creak."
slug: "designing-a-data-oriented-compiler-part-2"
# draft: true
---

## Introduction

One of the easiest ways to start a compiler is to model the program as a tree of rich objects.

You define a `BinaryExpr`, a `CallExpr`, a `VarDecl`, a `WhileStmt`, and so on. Each node owns references to its children. Every compiler pass gets a visitor. Each node can carry methods for pretty-printing, type information, lowering hooks, or diagnostics. It is a natural design, and for small compilers it often feels exactly right.

That is not an accident. This style fits how many of us first learn compiler construction. It maps cleanly onto the grammar, keeps examples readable, and makes a recursive descent parser feel pleasantly direct. Bob Nystrom's [*Crafting Interpreters*](https://craftinginterpreters.com/) is a good modern example of how approachable and productive that style can be.

The trouble is that the same design that feels elegant at ten node types can start to feel heavy at a hundred. The issue is not that object-oriented techniques are wrong. The issue is that compilers, once they leave the toy stage, spend a lot more time **moving through data** than **sending messages to objects**. That is the point Chandler Carruth makes in [*Modernizing Compiler Design for Carbon Toolchain*](https://www.youtube.com/watch?v=ZI198eFghJk), and it is the lens I want to use in this post.

The central question is not "should a compiler be object-oriented?" It is: **what do we gain from rich node hierarchies, and at what point do they stop matching the actual workload of the compiler?**

## Where rich node hierarchies help

Before talking about their costs, it is worth being explicit about their strengths.

### They mirror the surface structure of the language

Most languages are easiest to explain in terms of nested syntax:

- an expression contains subexpressions,
- a statement contains expressions and blocks,
- a function contains parameters, a return type, and a body.

An object hierarchy mirrors that structure directly. A `BinaryExpr` with `Left`, `Right`, and `Op` fields is intuitive. A `FunctionDecl` that contains a list of parameters and a body block is equally intuitive. That correspondence is useful when the language is still in flux, because the compiler representation changes in the same shape as the grammar.

### They make parsing and debugging pleasant

Rich nodes are ergonomic during early development. They are easy to allocate, easy to print, and easy to inspect in a debugger.

For example, a parser can construct this shape directly:

```go
type BinaryExpr struct {
    Left  Expr
    Op    TokenKind
    Right Expr
    Span  Span
}
```

That is hard to beat for readability. If a parse goes wrong, you can dump the node tree and usually understand what happened immediately.

### They support compiler exploration

When you are still learning the language you are building, or still discovering what the semantic model wants to be, rich nodes lower the cost of experimentation.

It is easy to attach a new field to a declaration node, add a method for formatting, or create a fresh visitor for a new analysis pass. For early prototypes, that flexibility matters more than locality.

### They keep invariants local

One real advantage of richer structures is that they can make illegal states harder to represent.

If a `ForStmt` always has an initializer, condition, update clause, and body, then a dedicated node type can encode that invariant directly. The more structured the language feature, the more tempting it is to express the invariant in the type system and let the compiler implementation lean on it.

So there is a genuine case for object-oriented compiler structures: they are expressive, discoverable, and often the fastest route to a working frontend.

## Where they start to creak

Those benefits do not disappear. They simply stop being the whole story.

As the compiler grows, the hot path is usually no longer "construct a few nodes and print them." The hot path becomes:

- walk a large syntax representation,
- resolve and compare many names,
- attach metadata to many entities,
- lower one representation into another,
- and run repeated analyses over blocks, instructions, and values.

At that point, the trade-offs shift.

### Indirection becomes the default tax

In a pointer-rich AST, every pass pays for object identity whether it needs it or not.

Consider the visitor-based example below:

```go
type BinaryExpr struct {
    Left    Expr
    Right   Expr
    Op      Token
    Span    Span
    InferTy *Type
    ...
}

func (b *BinaryExpr) Accept(vst Visitor) {
    vst.VisitBinaryExpr(b)
}

func (t *TypeChecker) VisitBinaryExpr(b *BinaryExpr) {
    b.Left.Accept(t)
    b.Right.Accept(t)
    ...
}
```

This style is pleasant to write, but it introduces several layers of indirection into the hot path. The pass begins with an `Expr` interface value, dispatches to `Accept`, which then dispatches again to `VisitBinaryExpr`. Inside that method, the checker follows pointers to `Left` and `Right`, and each child repeats the same sequence. For a small tree this overhead is irrelevant. For a large codebase, repeated interface dispatch and pointer chasing become part of the baseline cost of simply *looking at the program*.

The deeper issue is not one virtual call in isolation. It is the cumulative effect of many tiny detours. A type checker is often asking simple questions: what kind of node is this, what are its operands, what name does it refer to, where should the inferred type be written? Those are data-access questions. Yet the representation forces the pass to hop through object boundaries and method calls before it can reach the few fields it actually needs.

There is also a layout problem hiding inside the API. A checker visiting `BinaryExpr` may only need the operator, child references, and a place to store inferred type information. It does not necessarily benefit from pulling along every other field that happens to live on the same node, such as full span data, comments, attributes, or cached analysis from some unrelated phase. When those fields share the same object, they share the same traversal cost.

A flatter representation changes the shape of the work. Instead of following object references, a pass can iterate linearly over compact node records:

```go
type BinaryExprRow struct {
    Left    NodeID
    Right   NodeID
    Op      TokenKind
    TypeOut TypeID
}
```

Now the checker can scan a dense slice of `BinaryExprRow` values, read only the fields it cares about, and write results into a parallel table keyed by `NodeID` or row index. The point is not that this is always prettier. The point is that it matches the actual workload more closely: repeated, predictable access over many small records.

None of this means a node like `BinaryExpr` is bad on its own. The problem appears when a pass has to process fifty thousand of them in sequence. Instead of streaming through compact records, it keeps bouncing across heap objects, pointer fields, interface values, and auxiliary allocations. Mike Acton's [*Data-Oriented Design and C++*](https://www.youtube.com/watch?v=rX0ItVEVjHc) remains one of the clearest explanations of why this matters: the machine cares about access patterns whether the type hierarchy is elegant or not.

### Identity gets tied to addresses

Pointers make identity feel free, but they quietly constrain the design.

If every pass refers to syntax nodes, symbols, or IR values by address, several things become more awkward:

- **Serialization and deserialization.**  
  A raw pointer is only meaningful inside one running process. If you want to write an AST, symbol table, or IR fragment to disk, ship it across a process boundary, or load it back later, pointer identity evaporates. A `NodeID` or `ValueID` survives that transition; `*Node` does not.

- **Deterministic dumps.**  
  Compilers benefit from stable textual output: IR dumps for debugging, golden files in tests, and reproducible logs when tracking down regressions. Pointer-based identity tends to leak process-specific details into that output, or at least encourages representations whose ordering depends on allocation history rather than logical structure.

- **Stable cross-phase references.**  
  A parser, resolver, type checker, and lowering pass all want to talk about "the same declaration" or "the same expression." If that identity is just a memory address, each phase becomes tightly coupled to a particular in-memory representation. Stable IDs make it easier for phases to communicate without depending on who owns the object.

- **Incremental recomputation.**  
  If a user edits one function, an incremental compiler wants to preserve unaffected work and recompute only what changed. That is much easier when entities are tracked by stable keys. Pointer identity makes this harder because a rebuild may allocate structurally identical nodes at entirely different addresses, even when their logical meaning has not changed.

- **Tooling that needs durable handles.**  
  IDE features, cross-reference indexes, jump-to-definition, cached diagnostics, and editor integrations all want handles that remain meaningful outside the immediate traversal that produced them. A pointer is a temporary implementation detail. A compact ID can become part of a stable protocol between compiler components and tools.

A pointer says "this object lives here right now." A compact ID says "this entity is the same thing across phases." For compiler pipelines, that distinction matters.

Consider a simple unresolved-name error. If the resolver reports it as "error on `*NameExpr`," that result is only useful as long as that exact node object is still alive and accessible. If it reports "error on `NodeID(418)` at `SpanID(27)`," the rest of the compiler can store it, sort it, print it later, or hand it to an editor without needing the original pointer at all. That is a much better fit for a pipeline made of multiple phases and tools.

### Behavior scales faster than representation

Rich hierarchies encourage a certain kind of local cleanliness: if you need a new behavior, add a method or a visitor.

That works for a while. Then the compiler accumulates:

- one visitor for pretty-printing,
- one for name resolution,
- one for type checking,
- one for CFG construction,
- one for constant folding,
- and several auxiliary maps keyed by node pointer because the node type itself can no longer absorb every concern.

One problem is that the visitor surface area tends to grow with the syntax tree, not with the actual needs of a pass. A type checker may end up implementing `VisitImportDecl`, `VisitUsingDecl`, `VisitBreakStmt`, and many other methods even when some of them do no real work. An import declaration, for example, may matter to parsing or module loading, but be irrelevant to expression typing. Yet once the pass is modeled as a full visitor, it still has to acknowledge that node somehow, often with an empty or nearly empty `Visit*` method.

That creates a lot of structural noise. A pass with dozens of `Visit*` methods can look substantial even when much of it is boilerplate. More importantly, it becomes harder to see which nodes actually matter to the phase. The interface says "this pass handles everything," while the implementation reality is often "this pass only cares deeply about a small subset of constructs."

There is also a boundary problem. Once every syntactic form is a candidate for its own `Visit*` method, it becomes difficult to decide what the unit of visitation should be. Should the `ElseIf` branch of an `IfStmt` have its own `VisitElseIfClause`, or should it be handled entirely inside `VisitIfStmt`? Should parameter lists, field lists, or generic constraints be separate visitation targets, or just internal details of the enclosing declaration? If the boundary is too fine, the pass logic gets fragmented across many tiny methods. If it is too coarse, a handful of large `Visit*` methods turn into mini interpreters for entire subtrees.

This is a sign that the behavior model is starting to drift away from the representation. The syntax tree is organized around grammar productions, but compiler passes are usually organized around semantic tasks: resolve bindings, infer types, build control-flow edges, collect constraints, lower expressions. Those tasks do not always line up neatly with the places where the tree offers convenient `Visit*` hooks.

The usual result is that each serious pass starts building a shadow model of the program anyway. The type checker keeps side tables for bindings and inferred types. The CFG builder groups statements into blocks and edges. The constant folder tracks evaluable expressions and literal values. At that point the visitor often remains as the traversal shell, but the real representation that drives the pass has moved elsewhere.

At that point, the code still looks organized, but the real state of the compiler has become fragmented. Some facts live on nodes. Some live in side tables. Some live in pass-local caches. Some live in maps keyed by addresses. The hierarchy remains visible, but the actual dataflow gets harder to see.

### Hot and cold data get mixed together

One of the quieter costs of rich objects is that they bundle data with very different lifetimes and access frequencies.

A parser may need comments and exact token trivia. A type checker may not. An IDE feature may care deeply about parent links. A lowering pass may only need node kind and child ranges. When all of that state lives on the same heap object, every traversal drags around fields that are irrelevant to the current pass.

This matters at the hardware level too. CPUs do not fetch individual fields; they fetch cache lines. If the data a pass uses frequently is packed together with rarely used fields, then each access risks loading extra bytes that the pass does not care about. Comments, formatting trivia, parent links, deferred annotations, and debugging metadata may all end up traveling through the cache simply because they share an object with a hot field like node kind or child reference.

Over a large traversal, that turns into avoidable memory traffic. Cache space gets occupied by cold data. Useful hot data gets evicted sooner. Sequential scans become less dense than they could be, and the processor spends more time waiting on memory than doing useful compiler work. For passes like name resolution, type checking, or IR lowering, which may touch tens of thousands of nodes in tight loops, this is often a bigger cost than any single dispatch or branch.

There is also a secondary performance effect: large node objects reduce how many useful records fit in cache at once. If the checker only needs four small fields per node, but every node carries a much larger payload, then fewer nodes fit into L1 or L2 cache, and each sweep over the tree becomes more expensive than necessary. A flatter representation can separate hot fields from cold ones so that the common case stays compact, while less frequently used metadata lives elsewhere.

That is not merely a performance complaint. It is also a design complaint. When hot and cold data are mixed together, it becomes harder to say what the minimal representation of a phase actually is. A cleaner layout often improves both the architecture and the runtime behavior at the same time.

## A concrete example: name resolution

Name resolution is a good place to see the difference in style.

Suppose the source contains many local bindings:

```c
fn f(a: i32) -> i32 {
    let x = a + 1;
    let y = x + 2;
    let z = y + x;
    return z;
}
```

In a classic object-oriented frontend, identifier uses might be represented as node objects containing:

- the source text,
- the source span,
- links to parent and child nodes,
- optional fields for resolved symbol and inferred type,
- and whatever extra flags a later pass wants to cache.

The resolver then walks the tree, looks up names in scope maps, and writes results back into the nodes or into side tables keyed by node pointers.

In a more data-oriented layout, the same pass might look more like this:

1. The parser writes `NameExpr` records into a dense node slice.
2. Each name is interned up front, so the node stores `InternedStringID` instead of raw text.
3. The resolver scans the relevant node range, looks up each interned name in the current scope table, and writes the result into a `[]SymbolID` side table indexed by `NodeID`.

The second design is not automatically better in every dimension. It is, however, much closer to the actual operation being performed: a batch pass over many small records with predictable reads and writes.

That is the kind of workload where object structure often stops helping and starts getting in the way.

## The real trade-off is not elegance versus speed

It is tempting to frame this as a moral contest:

- object-oriented designs are pleasant but slow,
- data-oriented designs are ugly but fast.

I do not think that framing is useful.

The more important distinction is between **representations that match the workload** and **representations that hide the workload**.

Rich node hierarchies often hide the workload well at the start. They let you think in terms of syntax categories, not memory layout. That is a feature when you are still discovering the language. But once you know that a pass is fundamentally "scan these nodes, read these IDs, write these annotations," then the representation should start reflecting that fact.

In other words, data-oriented design is not a rejection of structure. It is an attempt to choose the *right* structure for repeated compiler work.

## A hybrid approach is usually the right one

This is also why I do not believe the answer is "never use objects" or "never use trees."

Compilers naturally want different representations at different stages.

- During parsing, recursive structure is often the simplest thing to work with.
- During semantic analysis, stable IDs and side tables become increasingly attractive.
- During lowering and optimization, pools, indexes, and phase-specific tables often fit better than object graphs.

A good compiler can absolutely mix these styles. In fact, many real compilers do.

The mistake is not using a node hierarchy. The mistake is assuming that because a node hierarchy was the easiest way to build the frontend, it must remain the best representation for every later phase.

## What this means for Kiwi

For Kiwi, this line of thinking leads to a few practical biases:

- prefer interned names over raw strings once parsing is done,
- prefer stable IDs over pointer identity when data crosses phase boundaries,
- prefer flat or pool-backed storage when a pass mostly scans,
- and keep phase-specific data explicit rather than smearing it across one master node type.

That does not mean Kiwi will never allocate trees or use pointers. It means I want those choices to be deliberate. If a representation exists mainly because it is familiar, that is a weak reason to keep it once the compiler grows.

## Closing thoughts

Object-oriented compiler structures are attractive because they make the program look like the language it is compiling. That is a real strength. But a compiler is not only a model of a language. It is also a machine for repeatedly transforming structured data.

Once that second reality starts to dominate, rich node hierarchies often become less helpful than they first appeared. They still have a place. They are just no longer the default answer to every representation problem.

That is the trade-off I wanted to make explicit in this post.

In the next one, I will zoom in on a more specific design decision: why interned names, literals, and strings often give a compiler a much cleaner foundation than passing raw text through every phase.
