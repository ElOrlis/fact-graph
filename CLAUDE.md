# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Fact Graph is a cross-platform (JVM/JavaScript) Scala 3 knowledge graph for modeling the U.S. Internal Revenue Code and related tax law. It uses sbt with Scala.js cross-compilation.

## Build & Development Commands

```bash
sbt compile            # Compile JVM build
sbt fastOptJS          # Compile JS build (dev)
sbt fullOptJS          # Compile JS build (production)
sbt test               # Run all tests (JVM + JS)
sbt clean test         # Fresh build + tests
sbt publishLocal       # Publish to local repository
```

Run a single test:
```bash
sbt "testOnly *DollarSpec"     # By class name pattern
```

Formatting uses scalafmt (v3.9.9, Scala 3 dialect, 120 max column, trailing commas required):
```bash
scalafmt               # Format all files
```

**Prerequisites:** Java 21, sbt, scalafmt (via coursier), Node.js (for JS tests).

## Architecture

### Core Abstractions (in `shared/src/main/scala/gov/irs/factgraph/`)

The system has a **schema-instance duality**:

- **`FactDictionary`** — Schema registry. Holds `FactDefinition` entries keyed by `Path`. Loaded from XML. Can be frozen to prevent mutation.
- **`FactDefinition`** — Schema template for a fact. Lazily builds its `CompNode` on first access (enabling circular/forward references).
- **`Graph`** — Runtime container. Holds the dictionary + `Persister` (storage). Entry point for `get()`, `set()`, `delete()`, `explain()`. Manages `factCache` and `resultCache` (both cleared on save).
- **`Fact`** — Runtime instance of a fact at a specific `Path`. Wraps a `CompNode`, provides caching, mutation, and validation.
- **`Factual`** (trait) — Common interface for `Fact` and `FactDefinition`. Provides graph navigation via `apply(Path)`.

### Three-Monad Evaluation System (`monads/`)

Evaluation composes three monads for domain-specific needs:

- **`Result[A]`** — Computation state: `Complete(v)`, `Placeholder(v)` (computed but may be incomplete), or `Incomplete`. Placeholder is critical for graceful degradation when dependencies are missing.
- **`MaybeVector[A]`** — Cardinality: `Single(x)` for scalar facts, `Multiple(vect)` for collection-derived facts.
- **`Thunk[A]`** — Lazy wrapper enabling short-circuit evaluation (e.g., `All` stops on first false, binary operators skip RHS if LHS is Incomplete).

### Expression System (`Expression.scala`)

`Expression[A]` is an enum with 12 cases representing all computation patterns: `Constant`, `Writable`, `Dependency`, `Switch`, `Extract`, `Unary`, `Binary`, `Arity4`, `Reduce`, `ConditionalList`, `Aggregate`, `Collect`. Each supports three evaluation modes: `.get()` (eager), `.getThunk()` (lazy), `.explain()` (provenance).

### CompNode Types (`compnodes/`)

~75 node types implementing concrete computations. Each implements `CompNodeFactory` for dynamic creation from XML config. Categories:
- **Leaf/data nodes:** `BooleanNode`, `IntNode`, `DollarNode`, `StringNode`, `DayNode`, `CollectionNode`, etc.
- **Arithmetic:** `Add`, `Subtract`, `Multiply`, `Divide`, `Floor`, `Ceiling`, `Round`
- **Comparison:** `LessThan`, `GreaterThan`, `Equal`, etc.
- **Logic:** `All`, `Any`, `Not`
- **Aggregation:** `Maximum`, `Minimum`, `CollectionSum`, `Count`
- **String:** `Length`, `Paste`, `Regex`, `Trim`
- **Date:** `Today`, `LastDayOfMonth`, `PayrollMonthsBetween`

### Operator Hierarchy (`operators/`)

Stateless singleton operators that power CompNodes:
- **`UnaryOperator[A, X]`** — Single input
- **`BinaryOperator[A, L, R]`** — Two inputs with lazy RHS (enables short-circuit)
- **`AggregateOperator[A, X]`** — Vector reduction to scalar
- **`CollectOperator[A, X]`** — Collection traversal
- **`ReduceOperator[A]`** — Left-fold

### Path & Navigation (`Path.scala`, `PathItem.scala`)

`Path` represents location in the graph (e.g., `/jobs/*/income`). `PathItem` variants: `Child(name)`, `Member(uuid)`, `Wildcard`, `Unknown`, `Parent`. Paths can be absolute or relative. `.asAbstract` converts UUIDs to wildcards for schema lookup. `.populateWildcards(graph)` expands wildcards to concrete paths.

### Domain Types (`types/`)

Opaque types for financial/tax precision: `Dollar` (BigDecimal with scale 2, banker's rounding), `Rational`, `Day`, `Days`, `Tin`, `Ein`, `Pin`, etc. Dollar parsing handles formats like `"1,234.00"`, `"(1.23)"` (negative), `"-1.23"`.

### Limits & Validation (`limits/`)

`Limit` trait with implementations: `Max`, `Min`, `MaxLength`, `MinLength`, `MaxCollectionSize`, `Match` (regex). Attached to writable facts only. `fact.validate()` returns `LimitViolation` sequences.

### Persistence (`persisters/`)

`Persister` trait with `InMemoryPersister` default. Storage keyed by `Path`. Save returns `(Boolean, Seq[LimitViolation])`.

### Explanation / Provenance (`Explanation.scala`)

`Explanation` enum tracks why a fact has its value: `Constant`, `Writable`, `Operation`, `Dependency`, `NotAttachedToGraph`. Used for auditability of computed tax values.

## Cross-Platform Structure

```
shared/src/main/scala/   — Common code (all core logic)
shared/src/test/scala/   — Common tests
jvm/src/main/scala/      — JVM-specific code
js/src/main/scala/       — JavaScript-specific code (ES modules, .mjs output)
```

JS API surface uses `@JSExport` annotations on `Graph` methods.

## Testing

Tests use **ScalaTest** (`AnyFunSpec` style) with `describe`/`it` blocks. Test dictionaries use `FactDictionaryForTests` which allows setting derived facts. Test output configured with `-oI` flag.

## Key Patterns

- **Scala 3 `given`/`using`** passes `Factual` context implicitly through evaluation
- **Factory registry** (`CompNode.factories`, `Limit.factories`) enables runtime node creation from XML without hardcoding
- **Cache invalidation is coarse-grained** — all caches cleared on `save()`, not incrementally
- **`Placeholder` vs `Incomplete`** — Placeholder means "computed but upstream may be incomplete"; Incomplete means "no data available"