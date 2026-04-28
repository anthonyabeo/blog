---
title: "Obx - Oberon+ Compiler"
date: 2026-04-27T12:00:00Z
# draft: true
---

## ABSTRACT
[Obx](https://github.com/anthonyabeo/obx)  is a compact, pedagogical compiler for an extended [Oberon dialect](https://oberon-lang.github.io/). The implementation following a traditional linear pipeline and includes and small standard library for performing operating systems level tasks. The source is parsed into a small, explicitly-typed AST, validated by a straightforward semantic pass, lowered to a simple three-address intermediate representation, and emitted by a minimal backend. The compiler provide a few interfaces for interaction including a Command Line Interface and a [web-based playground]( https://obxplayground.trunkwhorl.com/) that allows users to check and (soon) run example programs, inspect outputs (diagnostics, control-flow graph, assembly, etc). 

## DESIGN

The toolchain follows a linear pipeline: `lexer → parser → semantic analysis → IR lowering → backend codegen`. The AST favors explicit node types and annotations to simplify reasoning about correctness; the IR preserves block structure with explicit temporaries to make basic optimizations and debugging tractable. Typical modules are `lexer`, `parser`, `sema`, `ir`, and `codegen`.

### Lexer
The obx lexer (scanner) is a small, pragmatic, Unicode‑aware state‑machine scanner. It converts source text into a stream of `token.Token` values consumed by the parser. The scanner  emits precise token positions, supports rich numeric and string literal formats (including hex strings), keeps comments and directive tokens discoverable for pre‑processing, and reports lexical errors as ILLEGAL tokens with helpful messages.

#### Files of interest
* Scanner implementation: `src/syntax/scan/scanner.go`
* State machine (tokenization rules): `src/syntax/scan/state.go`
* Token definitions: `src/syntax/token/token.go`
* Parser integration: `src/syntax/parser/ (uses the scanner)`

#### Overview of design goals
* Produce a compact, well‑typed token stream with source positions (Pos, End) for diagnostics.
* Good diagnostics for malformed literals (unterminated strings, malformed numbers, invalid hex digits).
* Efficient: scanner runs in a goroutine and emits tokens over a buffered channel.

#### Scanner architecture
state functions The lexer uses function states (a la Rob Pike's lexer pattern). state.go defines `type StateFn func(*Scanner) StateFn` and many small scan* functions implementing tokenization for identifiers, numbers, strings and so on. The main loop in scanner.go triggers the state machine:
```go
// scanner.go (simplified)
func (s *Scanner) run() {
  for state := scanText; state != nil; {
    state = state(s)
  }
  close(s.items)
}
```
Tokens are emitted via a channel `s.items chan token.Token` and read by callers through `NextToken()`.
The scanner records `start`, `pos`, `width` (last rune), and `line/column` counters. `emit` / `emitWithValue` creates token.Token{Kind, Lexeme, Pos, End} and resets `start`.

#### Example Usage
```go
import (
  "github.com/anthonyabeo/obx/src/syntax/scan"
  "github.com/anthonyabeo/obx/src/support/source"
  "github.com/anthonyabeo/obx/src/syntax/token"
)

src := []byte(`MODULE X; BEGIN END X.`)
mgr := source.NewManager()
sc := scan.Scan("example.obx", src, mgr) // registers source bytes with source.Manager

for {
  tok := sc.NextToken()
  if tok.Kind == token.EOF {
    break
  }
  fmt.Printf("%v %q @%d..%d\n", tok.Kind, tok.Lexeme, tok.Pos, tok.End)
}
```

### Parser
The Obx parser is pragmatic and purpose‑built: it turns a token stream from the lexer into an unambiguous AST while keeping diagnostics clear, making it easier for subsequent compiler phases (semantic analysis, IR lowering) to traverse and operation on it. Because some language constructs must be interpreted differently depending on prior declarations, the parser consults a small, incremental symbol table during parse to disambiguate constructs (e.g., type-guard vs. function call) and employs a panic error recovery style that uses synchronization tokens to continue parsing.

#### Files of interest
* Scanner: `src/syntax/scan/*`
* Token definitions: `src/syntax/token/token.go`
* Parser: `src/syntax/parser/*`

#### Goals
* Produce an unambiguos AST that directly represents programmer intent (so later passes need less rewriting).
* Provide accurate, early diagnostics with precise source spans.
* Keep parser implementation maintainable and well tested.
* Support incremental or interactive workflows in the future (editor plugins).

#### Parser architecture
The parser is a classic recursive-descent parser with `parse*` function for each grammar production. It parses Oberon+ programs into a series of compilation units (module or definition) while incrementally building a scope-based symbol environment (`ctx.Env`) for declarations.
```go
func NewParser(ctx *compiler.Context, fileName string, content []byte) *Parser {
	p := &Parser{
		sc:       scan.Scan(fileName, content, ctx.Source),
		ctx:      ctx,
		fileName: fileName,
	}
	p.next()

	return p
}

func (p *Parser) Parse() (unit ast.CompilationUnit) {
	switch p.tok {
	case token.MODULE:
		unit = p.parseModule()
	case token.DEFINITION:
		unit = p.parseDefinition()
	default:
		p.errorExpected("MODULE or DEFINITION")
	}

	return
}
```
This symbol environment makes the parsing context-sensitive. This is necessary to assist parsing given that the grammar is not context free. A prime example can be seen in the grammar for `Type Guard` vs `Function Calls` and deciding where `IDENTIFIER . IDENTIFER` is a module's reference an entity such as `IO.WriteLn` or a record field selector such as `r.f`.

```ebnf
A type guard v(T) asserts that the dynamic type of v is T (or an extension of T), i.e. program execution is aborted, if the dynamic type of v is not T (or an extension of T)

FunctionCall           = designator ActualParameters
ActualParameters = '(' [ ExpList ] ')'
```

### Semantic Analysis
Semantic analysis validates program meaning after parsing: it resolves names, checks types, performs control‑flow and return‑path analysis, builds the inheritance/layout view for records and methods, and emits user‑facing diagnostics.

Entry Point
```go
// src/sema/sema.go (simplified)
func (s *Sema) Validate() {
  resolve := NewNameResolver(s.ctx)
  checker := NewTypeChecker(s.ctx)
  flow := NewFlowChecker(s.ctx)

  for _, unit := range s.obx.Units {
    s.ctx.Env.SetCurrentScope(s.ctx.Env.ModuleScope(unit.Name()))
    resolve.Resolve(unit)   // name binding / resolution
    checker.TypeCheck(unit) // type inference/verification
    flow.Analyse(unit)      // control flow & return-path checks
  }

  inv := &InheritanceView{ctx: s.ctx}
  inv.RunAll(s.obx.Units)   // layout/vtable/RTTI across whole program
}
```
Why this order?
* Name resolution must run before type checking so that identifiers in the AST carry Symbol pointers (which include declared types).
* Type checking relies on resolved Symbols and populates AST nodes with semantic types.
* Flow analysis (return checks, loop/exit context checks) runs after types are known.
* Inheritance/layout is built last because it needs the full program type graph (all units) to compute layouts, vtables and possible RTTI.

#### Files of interest
Name resolution: `src/sema/resolve.go` (NamesResolver / NameResolver).  
Type checking: `src/sema/typecheck.go` (TypeChecker).  
Flow analysis: `src/sema/flow.go` (FlowChecker).  
Inheritance/layout: `src/sema/inheritance_view.go` (InheritanceView).  
Type definitions & utilities: `src/sema/types/*` (array.go, record.go, basic.go, etc.).  
Tests: `src/sema/*_test.go and src/sema/types/*_test.go`.  

#### Responsibilities and features
* ##### Name resolution
    1. Lookup identifiers in the lexical/module scopes.
    2. Resolve qualified identifiers (module.member) and enforce export rules.
    3. Handle special constructs that temporarily override symbols (e.g. WITH arms and CASE type arms) via per‑walk override maps.
    4. Bind AST nodes to ast.Symbol entries, set mangled names for codegen.
    5. Report “undeclared identifier” and similar name errors.

* ##### Type checking & inference
    1. Convert AST type descriptors into types.Type objects (see src/sema/types).
    2. Validate expression typing, operator compatibility, predeclared procedure contracts.
    3. Enforce assignment compatibility, indexing rules, field selection and pointer deref rules.
    4. Support type guards and narrowing via typeOverrides used in the type walker for locally narrowed types.
    5. Perform special checks for FFI/C interop and DEFINITION modules (C names, DLL names).

* ##### Control-flow analysis
    1. Verify RETURN presence for function procedures (ensure all paths return).
    2. Ensure EXIT appears only inside loops, FOR control variable not modified, etc.
    3. Assign labels to loops to make later passes (or diagnostics) precise.

* ##### Inheritance / layout
    1. After all units processed the code computes inheritance layout (record field offsets, vtable layout, RTTI), required by code generation and FFI mappings.

#### Key data structures and patterns
* ##### Environment / symbol table
    * The compiler uses an environment (ctx.Env) that exposes operations like Lookup, LookupQualified, ModuleScope, AddModuleScope, and scope push/pop.
    * AST symbols implement ast.Symbol (param/variable/procedure/type/module symbols). Each symbol contains an AstType() getter and other metadata (mangled name, props, CName/DLLName for externs).
    * Name resolution binds ast.QualifiedIdent and ast.Identifier nodes to ast.Symbols, and calls sym.SetMangledName(ast.Mangle(sym)) for codegen.

* ##### Symbol overrides & narrowing
    * NamesResolver uses a private symbolOverrides map[string]ast.Symbol to temporarily rebind a name in a nested region (e.g. case arms or with).
    * TypeChecker uses typeOverrides map[string]types.Type for type narrowing under guards (IF ... TYPE ... style constructs).
    * These are intentionally per‑walk, ephemeral, and not shared across compilation units.

* Type representation
    * The type system is implemented in src/sema/types/. Types are first‑class Go values (e.g., *types.RecordType, *types.ArrayType, *types.PointerType, *types.ProcedureType, plus basic types like Int32Type).
    * Helpers exist (e.g., types.Underlying, types.IsInteger, types.IsRecord, types.IsExtensionOf, types.IsPointerToRecord) to implement language semantics succinctly.
    * When the resolver binds a named type symbol, the ast.NamedType node gets NamedType.Symbol set so the typechecker can fetch the actual types.Type.

#### Example flows (name resolution → type check)
1. Name resolution sets ident.Symbol for every ast.Identifier.
2. TypeChecker, when visiting ast.Identifier, calls def.Symbol.AstType().Accept(t) to convert ast.Type into a types.Type (or simply read an already associated semantic type).
3. When encountering a field selection a.b, the resolver ensures a has a denoted type; TypeChecker then inspects types.Underlying(a.Type()) and verifies b exists on the record type; it assigns s.Symbol.SetType(field.Type) so downstream passes know the precise type.

### IR Lowering
IR lowering translates the language AST/HIR into a low‑level intermediate representation suitable for optimizer passes and backend code generation. In obx this is a two‑step process: desugaring produces a compact HIR (high‑level IR) that normalizes syntactic sugar and control flow, then the HIR is lowered into the OBX IR (`obxir`) — a simple, explicit instruction‑oriented IR with basic blocks, values and a small set of ops. The IR is designed to make optimisations (DCE, constant folding, register allocation) and multi‑target code generation straightforward.

#### Files of interest
- HIR / desugar: `src/ir/desugar/*` (notably `hirgen.go`, `hir.go`)
- OBX IR: `src/ir/obxir/*` (builder, instr, block, value)
- Tests / harness: `src/codegen/driver_test.go` uses `testutil.ParseSourceAndLowerToMIR`
- Optimisers: `src/opt/*` (passes that consume OBX IR)
- Codegen backends: `src/codegen/*` (emit assembly from OBX IR)

#### Why two stages (desugar → obxir)
- HIR (desugared AST) keeps source‑level constructs but in a canonical form: multi‑assignment, syntactic sugar (e.g., `WITH`, `CASE` sugar), high‑level expression forms are normalized. This simplifies lowering rules and makes it easier to test language transformations.
- OBX IR is small and explicit (instructions, blocks, values). The optimizer and codegen operate on a clean, predictable instruction set.
- Separation keeps lowering code manageable and isolates language desugaring from target‑specific or register concerns.

#### High level pipeline
1. Parse source → AST (parser)
2. Desugar / HIR generation (`src/ir/desugar/hirgen.go`)
   - Normalize constructs, produce HIR nodes (functions, blocks, statements, expressions)
   - reduce all loop-based constructs (while, repeat, for, etc) into a simple `LoopStmt`.
   - All symbols are made explicit (using the symbol and type information)
3. Lower HIR → OBX IR (`src/ir/obxir/builder.go`, `src/ir/obxir/*`)
   - Produce basic blocks, instructions, operands and explicit temporaries
4. Run IR passes: CFG construction, DCE, folding, register allocation (see `src/opt/`)
5. Backend codegen emits assembly for target

#### Goals for OBX IR
- Explicit control flow: basic blocks and terminators (jump/branch/return).
- Explicit temporaries/SSA-like values (not necessarily full SSA, but friendly to register allocation).
- Small, target‑neutral instruction set (loads/stores, arithmetic, calls, control ops).
- Preserve enough source metadata for diagnostics (mapping from IR instructions back to original source positions).
- Make common optimizations easy (constant propagation, folding, DCE).

#### Example: source → HIR → OBX IR (mini example)

Source (OberonX)
```obx
MODULE M;
VAR x: INTEGER;

PROCEDURE Add(a, b: INTEGER): INTEGER;
BEGIN
  RETURN a + b
END Add;

BEGIN
  x := Add(2, 3)
END M.
```

HIR (conceptual / simplified)
```
Module M {
  Global x : i32
  Func Add(a:i32, b:i32) -> i32 {
    Return (AddExpr a b)
  }

  Main {
    tmp0 = Call Add(2,3)
    Store Global x <- tmp0
  }
}

```
OBX IR (pseudo-instructions)
```
# Function Add
func Add:
  entry:
    %0 = MOV arg0       # a
    %1 = MOV arg1       # b
    %2 = ADD %0, %1
    RET %2

# Main (module init)
main:
  %3 = CONST 2
  %4 = CONST 3
  %5 = CALL Add, %3, %4
  STORE [global_x] <- %5
  RET
```
### Optimization
The optimizer in obx is a small, pragmatic pass framework that runs a set of local and global transformations over the OBX IR. It focuses on correctness and developer ergonomics: readable debug output, a configurable pass pipeline, and basic but effective transforms (constant folding, control‑flow tidy/merge, dead‑code elimination, SSA helpers). The code lives under src/opt/ and operates on the IR defined in src/ir/obxir/.

#### Key files
- Pass framework and manager: `src/opt/pass.go`
- CFG construction and control transforms: `src/opt/control.go`
- Dead-code and block-level DCE: `src/opt/dce.go`
- Constant folding: `src/opt/fold.go`
- SSA helpers (construct, place φ, rename): `src/opt/ssa.go`
- Tests: `src/opt/*_test.go`
- The optimizer is exercised end‑to‑end via lowering & codegen tests: `src/codegen/driver_test.go`

#### Goals of the optimizer
- Simplify and normalize control flow (clean CFG).
- Reduce work for backends by folding constants and removing unreachable / unused code.
- Provide a pass manager so users can configure opt levels and which passes run.
- Produce stable, debuggable transformations with source mapping preserved.

#### Pass framework
- Pass interface (in pass.go):
    - Name() string
    - Run(fn *obxir.Function, ctx *PassContext) *ChangeSet
- PassContext carries ephemeral analyses and a ChangeSet. Use it to cache analysis results or surface change logs between passes.
- ChangeSet signals if a pass changed the function and holds human-readable notes (used when verbose).
- PassManager:
    - ConfigurePasses(map[string]any) to pick passes by optlevel or explicit enable/disable strings.
    - RunOnce(fn) or RunFixedPoint(fn, maxIters) to iterate passes until a fixed point.

#### Control flow & CFG utilities (control.go)
- BuildCFG(fn):
    - Maps block labels to blocks, inspects terminators (CondBr/Jmp/Return/Halt) and populates Succs and Preds maps for each block.
    - Normalizes Return to jump to a dedicated exit block.
    - Calls CleanCFG to run iterative CFG cleanups.

- CleanCFG runs a suite of block/edge transforms until stable:
    - EliminateDeadBlocks — BFS from entry; removes unreachable blocks.
    - RemoveEmptyBlocks — splices out trivial single-jump blocks.
    - FoldRedundantBranches — replace conditional branches whose true/false targets are identical with an unconditional jump.
    - HoistBranch — move branch tests up a level where appropriate (simplifies nesting).
    - CombineBlocks — merge a block that ends with an unconditional jump into its single successor if safe.

- Dominator & DF computations:
    - ComputeDominators, ImmediateDominators, DominatorTree, ComputeDF

- Def/use collection:
    - ComputeDefUse computes defSites and useSites used by SSA pass.

#### Dead-code elimination and block-level cleanups (dce.go)
- Block-level DCE functions (see above) remove unreachable blocks and compact CFG.
- These functions also fix predecessor/successor lists and update terminators/instructions to keep CFG consistent.
- Higher-level DCE (eliminate unused instructions / values) is typically done after or as part of other passes; you can extend with value-level liveness analysis.

#### Constant folding pass (fold.go)
- ConstantFold iterates blocks/instructions in DFS order and asks instructions that implement obxir.Foldable to fold themselves.
- If instr.(obxir.Foldable).CanFold() returns true, the pass replaces the instruction with a MoveInst assigning the folded constant value (via Fold()).
- Fold logs are appended to the ChangeSet so verbose runs show what was replaced.

#### SSA helpers (ssa.go)
- SSAConstruct(fn) orchestrates building SSA form for easier downstream transforms:
    - Build CFG: BuildCFG(fn)
    - Compute dominators and def/use info: ComputeDom(fn)
    - Place φ-nodes: PlacePhiNodes(fn) — uses DF and def-sites from ComputeDefUse.
    - Rename values: RenamePhiNodes(fn) — creates versioned names and rewrites defs/uses.
- PlacePhiNodes and RenamePhiNodes follow the classic Cytron algorithm (worklist of def-sites, DF-based insertion, rename via DFS over dominator tree).
- SSA data placed into fn.SSAInfo enables SSA-aware passes.


### Codegen
The code generation (codegen) phase maps the compiler’s target-independent representation into efficient, correct machine code for a concrete CPU and ABI.

#### Where to look in this repo
- `src/codegen/` — root of codegen implementation.
    - `isel/` — instruction selection (pattern matching and lowering).
    - `ralloc/` — register allocation (live range analysis, linear-scan / graph color helpers).
    `asm/` — assembly-level primitives, operand/instruction modeling and emission.
    `target/` — target interface, per-target implementations (arm64/, riscv/), and framer.go.
    - `driver.go, driver_test.go` — harness and examples for end-to-end generation tests. Look at examples/ (e.g., examples/basics/Main.obx) for input programs used during development.

#### Goals:
- Lower ObxIR ops into target instruction sequences.
- Assign registers and memory locations respecting calling conventions and liveness.
- Emit target-appropriate frames, prologues/epilogues, and relocations.
- Produce code that balances correctness, code size, and runtime performance.

#### Pipeline stages (practical walkthrough)
1. Input IR and lowering
    - Start with the compiler’s mid-level IR (HIR/OBX IR). Lower complex operations to a small set of lower-level operations that match available target instructions (e.g., decompose compound ops, guarantee legal operand types).
    - Lowering makes instruction selection tractable and separates target-independent transforms from target-specific patterns.

2. Instruction selection (src/codegen/isel)
    - The selector maps lowered IR nodes to target instruction patterns. This repo organizes pattern matching into isel rules and a small pattern matcher (bottom-up rewrite / pattern-matching approach).
    - Patterns capture when an IR node and its children can be implemented by a single instruction or a short sequence; selection prefers cost-minimal matches.

3. Legalization and target-specific lowering (src/codegen/target, framer.go)
    - After selection, nodes that are not directly representable by a target are legalized (split into representable parts or emulated via helper sequences).
    - framer.go crafts the function frame: stack slot layout, spill slots, callee-save handling, and stack alignment according to the target ABI.

4. Liveness analysis and live range building (src/codegen/ralloc/liveness.go etc.)
    - Compute live intervals and interference for temporaries and virtual registers; this is required for any register allocator.
    - Use flow analysis to understand where values are needed and where they can be reused or spilled.

5. Register allocation (src/codegen/ralloc)
    - The repo contains allocator implementations and helpers for linear-scan and graph-coloring style approaches. Linear-scan offers simpler, faster allocation (often used for JITs and fast builds); graph coloring gives better allocation quality at the cost of complexity.
    - The allocator produces mappings from virtual registers to physical registers or stack slots and inserts spill/reload instructions where necessary.

6. Rewriting and instruction scheduling (micro-scheduling)
    - After allocation, some instruction sequences may need re-emission to ensure register constraints are satisfied. Small scheduling passes reorder instructions locally to reduce stalls and improve pipeline behavior (architecture-specific).
    
7. Assembly emission (src/codegen/asm and target backends)
    - Final encoding and textual or binary emission of instructions happen in the asm and target/ backend code (e.g., arm64/ implementation).
    - The emitter is responsible for proper instruction encoding, constant pools, relocations and linking metadata.

## KEY CHALLENGES AND TRADE-OFFS
#### Non‑context‑free grammar: the practical problem and the parser workaround 
The language’s grammar is not strictly context free: some syntactic choices depend on the current name bindings (is this identifier a type, a value, or an import?). A pure CFG parser would produce ambiguous or incorrect ASTs in those cases.
Workaround we used During parsing we build and consult a lightweight symbol table to disambiguate constructs where necessary (essentially a parse-time isType lookup). This keeps the AST cleaner and reduces downstream ambiguity: QualifiedIdent nodes are created with the intended shape, and where possible the parser attaches the name binding early.

##### Tradeoff
- Benefit: fewer ambiguous AST shapes; downstream passes (typecheck, lowering, codegen) have a simpler input and fewer ad-hoc reclassifications to perform.
- Cost: it couples the parser to the current symbol environment. That coupling forces careful handling of forward declarations and reclassification when later resolution yields different bindings.

#### IR and lowering tradeoffs Explicit vs implicit temporaries
- We favor explicit temporaries in the IR. That increases IR size, but much simplifies analyses, register allocation and correctness reasoning.

#### Preserve source semantics vs early optimization
- Lowering is conservative about source-level semantics (order-of-evaluation and side-effects) — it does not perform aggressive optimizations that would change error reporting or observable behavior.
- Aggressive optimizations are deferred to later optimizer passes after lowering. This separation keeps lowering simpler and safer.

#### Two‑stage lowering vs single pass
- The pipeline uses two major lowering stages (HIR → OBX IR → target-lowerings). This isolates language normalization from target-specific lowering.
- Cost: an extra pass and more intermediate structures.
- Benefit: clearer invariants, easier testing, and improved maintainability.

#### Representing compound types and ABI concerns
- OBX IR is deliberately target neutral. ABI-specific concerns (register vs stack parameter placement, calling conventions) are resolved in the backend (codegen).
- FFI/extern symbols carry ABI metadata so codegen can emit correct call sequences for foreign functions.

#### Backend design tradeoffs Pattern-based selection vs hand-coded lowering
- Pattern-based instruction selection (bottom-up rewrite, pattern matcher) makes it easy to add and tune instruction patterns and to target multiple ISAs.
- Hand-coded lowering can be simpler initially but is less flexible when adding targets or when you need to capture complex instruction idioms.

#### Register allocation: linear-scan vs graph coloring
- Linear-scan is fast and simple — a good pragmatic default.
- Graph coloring reduces spill counts and improves runtime in tight hot paths, but it is heavier to implement and slower to run.

#### Spilling strategy
- We prefer to spill values with the longest next-use distance (heuristic that works well in practice).
- Different strategies (calleesave-first, callersave-first) are considered depending on calling convention and hot-path profile.

## LINKS
- Code Repository: https://github.com/anthonyabeo/obx — see the README for quick build/run steps.
- Web Playground: https://obxplayground.trunkwhorl.com
- Oberon+ documentation: http://github.com/oberon-lang/specification/blob/master/The_Programming_Language_Oberon%2B.adoc
