# DuckDB Internals Guide

A technical walkthrough of DuckDB's core subsystems for database-engineering learners.
Each section identifies the key C++ classes, primary source files, and specific function
signatures or line numbers that anchor the implementation.

---

## Table of Contents

1. [Parser / Binder — SQL → LogicalPlan](#1-parser--binder--sql--logicalplan)
2. [Optimizer — Stratified Optimization](#2-optimizer--stratified-optimization)
3. [Execution Engine — Push-Based Vectorized Model](#3-execution-engine--push-based-vectorized-model)
4. [Storage — PAX Format & Buffer Manager](#4-storage--pax-format--buffer-manager)
5. [Data Flow — `SELECT * FROM table` End-to-End](#5-data-flow--select--from-table-end-to-end)

---

## 1. Parser / Binder — SQL → LogicalPlan

### Overview

Turning a SQL string into an executable plan requires two distinct passes:

| Pass | What it produces | Primary classes |
|------|-----------------|-----------------|
| **Parsing** | A tree of unresolved `ParsedExpression` / `SQLStatement` nodes | `Parser`, `Transformer` |
| **Binding** | A `LogicalOperator` tree with fully-resolved column references | `Planner`, `Binder` |

---

### 1.1 Parsing

**Files:** `src/parser/parser.cpp`, `src/parser/transformer.hpp`

```
Parser::ParseQuery(const string &query)          // parser.cpp – public entry point
  └─ PostgresParser::Parse(query)                // third_party postgres parser
  └─ Transformer::TransformParseTree(tree, stmts)// transformer.hpp:61
       └─ Transformer::TransformExpression(node) // transformer.hpp:240
```

`Parser::ParseQuery` (defined at `src/parser/parser.cpp`) delegates to the embedded
PostgreSQL parser (`third_party/libpg_query`) which produces a `PGList` parse tree.
`Transformer::TransformParseTree` then walks that tree and converts each node into a
DuckDB `SQLStatement` subclass:

| Postgres node type | DuckDB class | File |
|--------------------|--------------|------|
| `PGSelectStmt` | `SelectStatement` / `SelectNode` | `src/parser/query_node/select_node.cpp` |
| Expressions | `ParsedExpression` subclasses | `src/parser/expression/` |
| Table references | `TableRef` subclasses | `src/parser/tableref/` |

#### Key classes

| Class | File | Role |
|-------|------|------|
| `Parser` | `src/parser/parser.cpp` | Entry point; calls PostgresParser & Transformer |
| `Transformer` | `src/parser/transformer.hpp` | Walks PG parse tree → DuckDB AST |
| `SelectStatement` | `src/parser/statement/select_statement.cpp` | Root parse node for SELECT |
| `ParsedExpression` | `src/parser/expression/` (many files) | Base for unresolved expressions |

---

### 1.2 Binding

**Files:** `src/planner/planner.cpp`, `src/planner/binder.cpp`,
`src/planner/binder/statement/bind_select.cpp`,
`src/planner/binder/query_node/bind_select_node.cpp`

```
Planner::CreatePlan(SQLStatement &statement)    // planner.cpp
  └─ Binder::Bind(statement)                   // binder.cpp:80
       └─ Binder::Bind(SelectStatement &stmt)  // bind_select.cpp:7
            └─ Binder::Bind(QueryNode &node)   // bind_select_node.cpp
                 ├─ Resolve column references against the catalog
                 ├─ Resolve function overloads
                 └─ Returns BoundStatement { plan, names, types }
```

`Binder::Bind` dispatches on the statement type (line 80 of `src/planner/binder.cpp`) to
the appropriate handler. For a `SELECT`, control reaches
`bind_select_node.cpp`, which:

1. **Binds FROM clause** – resolves table names in the catalog; creates
   `LogicalGet` operators.
2. **Binds WHERE clause** – resolves column references (qualified and unqualified)
   producing `BoundExpression` trees.
3. **Binds SELECT list** – handles star-expansion, aliases, aggregates and window
   functions.
4. **Produces a `LogicalOperator` tree** (e.g. `LogicalGet → LogicalFilter →
   LogicalProjection`).

#### Key classes

| Class | File | Role |
|-------|------|------|
| `Planner` | `src/planner/planner.cpp` | Orchestrates binding + verification |
| `Binder` | `src/planner/binder.cpp` | Resolves names against catalog |
| `BindContext` | `src/planner/bind_context.cpp` | Tracks in-scope columns |
| `LogicalOperator` | `src/include/duckdb/planner/logical_operator.hpp` | Base of logical plan tree |
| `BoundExpression` | `src/planner/expression/` | Fully resolved expression |

After binding, `Planner::CreatePlan` also calls `FlattenDependentJoins::DecorrelateIndependent`
(`src/planner/subquery/flatten_dependent_join.hpp`) to decorrelate subqueries and returns a
single-rooted `LogicalOperator` tree.

---

## 2. Optimizer — Stratified Optimization

**File:** `src/optimizer/optimizer.cpp`

### Entry point

```cpp
// optimizer.cpp
unique_ptr<LogicalOperator> Optimizer::Optimize(unique_ptr<LogicalOperator> plan_p)
```

`Optimizer::Optimize` is the single entry point called from `ClientContext` after
binding.  It runs `RunBuiltInOptimizers()` (defined in the same file), which applies
passes in a fixed order using the helper:

```cpp
void Optimizer::RunOptimizer(OptimizerType type, const std::function<void()> &callback)
```

Each pass checks `OptimizerDisabled(type)` (so individual passes can be disabled via
`SET disabled_optimizers = ...`) and is profiled by `QueryProfiler`.

### Optimizer passes — fixed execution order

| # | `OptimizerType` | Class / file | Effect |
|---|-----------------|--------------|--------|
| 1 | `EXPRESSION_REWRITER` | `ExpressionRewriter` (`src/optimizer/rule/`) | Constant folding, arithmetic simplification, LIKE optimizations, etc. |
| 2 | `CTE_INLINING` | `CTEInlining` (`src/optimizer/cte_inlining.cpp`) | Inlines non-recursive CTEs |
| 3 | `SUM_REWRITER` | `SumRewriterOptimizer` (`src/optimizer/sum_rewriter.cpp`) | Rewrites `SUM(x + C)` → `SUM(x) + C * COUNT(x)` |
| 4 | `FILTER_PULLUP` | `FilterPullup` (`src/optimizer/filter_pullup.cpp`) | Pulls filters up through joins |
| 5 | `FILTER_PUSHDOWN` | `FilterPushdown` (`src/optimizer/filter_pushdown.cpp`) | Pushes filters as close to scans as possible |
| 6 | `CTE_FILTER_PUSHER` | `CTEFilterPusher` (`src/optimizer/cte_filter_pusher.cpp`) | Derives and pushes filters into materialized CTEs |
| 7 | `REGEX_RANGE` | `RegexRangeFilter` (`src/optimizer/regex_range_filter.cpp`) | Converts `LIKE 'prefix%'` to range predicates |
| 8 | `IN_CLAUSE` | `InClauseRewriter` (`src/optimizer/in_clause_rewriter.cpp`) | Converts large IN lists to joins |
| 9 | `DELIMINATOR` | `Deliminator` (`src/optimizer/deliminator.cpp`) | Removes redundant `DelimGet`/`DelimJoin` |
| 10 | `CTE_INLINING` | (second pass) | Re-evaluates after filter pushdown |
| 11 | `EMPTY_RESULT_PULLUP` | `EmptyResultPullup` (`src/optimizer/empty_result_pullup.cpp`) | Short-circuits queries returning no rows |
| 12 | `WINDOW_SELF_JOIN` | `WindowSelfJoinOptimizer` | Replaces some window functions with self-joins |
| 13 | `PROJECTION_PULLUP` | `ProjectionPullup` | Pulls projections above joins |
| 14 | `JOIN_ORDER` | `JoinOrderOptimizer` (`src/optimizer/join_order/join_order_optimizer.hpp`) | DPhyp join-order enumeration; also converts cross products + filters to joins |
| 15 | `JOIN_ELIMINATION` | `JoinElimination` (`src/optimizer/join_elimination.cpp`) | Removes redundant joins |
| 16 | `UNNEST_REWRITER` | `UnnestRewriter` | Moves UNNESTs to projections |
| 17 | `UNUSED_COLUMNS` | `RemoveUnusedColumns` | Prunes columns not referenced downstream |
| 18 | `DUPLICATE_GROUPS` | `RemoveDuplicateGroups` | Deduplicates GROUP BY keys |
| 19 | `COMMON_SUBEXPRESSIONS` | `CommonSubExpressionOptimizer` | Extracts common sub-expressions |
| 20 | `COLUMN_LIFETIME` | `ColumnLifetimeAnalyzer` | Builds projection maps to drop columns early |
| 21 | `BUILD_SIDE_PROBE_SIDE` | `BuildProbeSideOptimizer` | Chooses build vs. probe side for hash joins |
| 22 | `COMMON_SUBPLAN` | `CommonSubplanOptimizer` | Converts repeated sub-trees to materialized CTEs |
| 23 | `LIMIT_PUSHDOWN` | `LimitPushdown` | Pushes LIMIT below PROJECTION |
| 24 | `ROW_GROUP_PRUNER` | `RowGroupPruner` | Prunes row groups using statistics / zone maps |
| 25 | `SAMPLING_PUSHDOWN` | `SamplingPushdown` | Pushes table sampling into scans |
| 26 | `TOP_N` | `TopN` (`src/optimizer/topn_optimizer.cpp`) | Converts `ORDER BY … LIMIT n` to `TopN` |
| 27 | `LATE_MATERIALIZATION` | `LateMaterialization` | Defers fetching wide columns |
| 28 | `STATISTICS_PROPAGATION` | `StatisticsPropagator` (`src/optimizer/statistics/`) | Propagates column statistics |
| 29 | `TOP_N_WINDOW_ELIMINATION` | `TopNWindowElimination` | Rewrites `ROW_NUMBER() … LIMIT n` |
| 30 | `COMMON_AGGREGATE` | `CommonAggregateOptimizer` | Deduplicates identical aggregates |
| 31 | `COLUMN_LIFETIME` | (second pass) | Final column lifetime analysis |
| 32 | `REORDER_FILTER` | `ExpressionHeuristics` | Simple expression-level filter reordering |
| 33 | `JOIN_FILTER_PUSHDOWN` | `JoinFilterPushdownOptimizer` | Pushes join filters after all other passes |

Extensions can also inject pre-/post-optimization passes via `OptimizerExtension`.

---

## 3. Execution Engine — Push-Based Vectorized Model

### Overview

DuckDB uses a **push-based pipeline model**: data flows from *source* operators
(which produce `DataChunk`s) through a chain of *pipeline operators* (which transform
chunks in place) to a *sink* operator (which accumulates results).  All data movement
uses fixed-size `DataChunk`s of up to `STANDARD_VECTOR_SIZE` (2048) rows.

**Files of interest:**
- `src/include/duckdb/execution/physical_operator.hpp` — `PhysicalOperator` base class
- `src/execution/operator/` — concrete physical operators
- `src/include/duckdb/common/types/data_chunk.hpp`
- `src/include/duckdb/common/types/vector.hpp`
- `src/include/duckdb/common/vector_size.hpp`

---

### 3.1 `DataChunk` and `Vector`

#### `Vector` — `src/include/duckdb/common/types/vector.hpp`

`Vector` is the fundamental columnar data container. Each vector stores one column
worth of data for up to `STANDARD_VECTOR_SIZE` rows.

Key members and concepts:

| Member / concept | Description |
|-----------------|-------------|
| `VectorType` enum | `FLAT_VECTOR`, `CONSTANT_VECTOR`, `DICTIONARY_VECTOR`, `SEQUENCE_VECTOR` |
| `data_ptr_t data` | Pointer to the underlying typed array (via `VectorBuffer`) |
| `ValidityMask validity` | Bit-mask for NULL values; bit `i` = 1 means row `i` is valid |
| `SelectionVector` | Indirection layer used by `DICTIONARY_VECTOR` to avoid copying |
| `UnifiedVectorFormat` | Flattened view (sel + data + validity) used by operators |

Accessors like `FlatVector::GetData<T>(vec)` and `ConstantVector::GetData<T>(vec)`
provide typed, zero-copy access to the underlying array.

#### `DataChunk` — `src/include/duckdb/common/types/data_chunk.hpp`

`DataChunk` is a horizontal slice of a relation: a fixed-length array of `Vector`s,
all sharing the same cardinality (`count`).

```cpp
class DataChunk {
    vector<Vector> data;        // one Vector per column
    idx_t count;                // number of valid rows (≤ capacity)
    idx_t capacity;             // max rows (STANDARD_VECTOR_SIZE by default)
    ...
};
```

Key operations:

| Method | Purpose |
|--------|---------|
| `Initialize(allocator, types)` | Allocates one `VectorCache` per column |
| `SetCardinality(n)` | Sets the row count; O(1) |
| `Slice(sel, count)` | Creates a dictionary view without copying |
| `Flatten()` | Materialises all column vectors as flat arrays |
| `Reset()` | Resets count to 0 and reclaims vector caches |

#### Vector size constant

```cpp
// src/include/duckdb/common/vector_size.hpp
#define DEFAULT_STANDARD_VECTOR_SIZE 2048U
```

This value is the maximum batch size processed per operator invocation. It is sized to
fit comfortably in L1/L2 cache while amortising per-row overhead.

---

### 3.2 `PhysicalOperator` — the operator interface

**File:** `src/include/duckdb/execution/physical_operator.hpp`

Every physical operator inherits from `PhysicalOperator` and implements one or more of
three interfaces:

#### Source interface (data producers)

```cpp
// Returns a batch of rows into `chunk`
SourceResultType GetData(ExecutionContext &context, DataChunk &chunk,
                         OperatorSourceInput &input) const;

// Thread-local / global state factories
unique_ptr<LocalSourceState>  GetLocalSourceState(ExecutionContext &, GlobalSourceState &) const;
unique_ptr<GlobalSourceState> GetGlobalSourceState(ClientContext &) const;
```

Example: `PhysicalTableScan` (`src/execution/operator/scan/physical_table_scan.cpp`)
implements `GetDataInternal` (line 160) to call the table function, which reads from
`RowGroup::Scan` in the storage layer.

#### Pipeline operator interface (transformers)

```cpp
// Transforms input into output, both provided as DataChunk
OperatorResultType Execute(ExecutionContext &context, DataChunk &input,
                           DataChunk &chunk,
                           GlobalOperatorState &gstate,
                           OperatorState &state) const;
```

Example: `PhysicalFilter` (`src/execution/operator/filter/physical_filter.cpp`, line 43)
evaluates predicates on the input chunk and writes matching rows to the output chunk.

Example: `PhysicalProjection` (`src/execution/operator/projection/physical_projection.cpp`, line 28)
executes scalar expressions via `ExpressionExecutor::Execute` and writes results to the
output chunk.

#### Sink interface (data consumers)

```cpp
// Accumulates data from input into global/local sink state
SinkResultType Sink(ExecutionContext &context, DataChunk &chunk,
                    OperatorSinkInput &input) const;

// Merges thread-local state into the global state
SinkCombineResultType Combine(ExecutionContext &context,
                              OperatorSinkCombineInput &input) const;

// Called once after all Sink() calls; finalizes the operator
SinkFinalizeType Finalize(Pipeline &pipeline, Event &event,
                          ClientContext &context,
                          OperatorSinkFinalizeInput &input) const;
```

Example: `PhysicalHashJoin` (`src/execution/operator/join/physical_hash_join.cpp`, line 484)
builds an in-memory hash table during `Sink`, then becomes a *source* during `GetData`,
probing the hash table with the probe side batches.

---

### 3.3 Operator directory map

```
src/execution/operator/
├── scan/         PhysicalTableScan, PhysicalColumnDataScan, ...
├── filter/       PhysicalFilter
├── projection/   PhysicalProjection, PhysicalUnnest
├── aggregate/    PhysicalHashAggregate, PhysicalUngroupedAggregate, PhysicalWindow
├── join/         PhysicalHashJoin, PhysicalNestedLoopJoin, PhysicalCrossProduct
├── order/        PhysicalOrder, PhysicalTopN
├── set/          PhysicalUnion, PhysicalIntersect, PhysicalExcept
├── persistent/   PhysicalInsert, PhysicalDelete, PhysicalUpdate
├── helper/       PhysicalLimit, PhysicalMaterializedCollector, ...
└── schema/       PhysicalCreateTable, PhysicalDropTable, ...
```

---

### 3.4 Pipeline scheduler

The `Executor` (`src/execution/executor.cpp`) builds a set of `Pipeline` objects from
the physical plan tree.  Each pipeline is a linear chain Source → [Operators] → Sink
that can be scheduled independently.  Parallel execution is achieved by running multiple
threads through different pipelines or by parallelizing the source (if
`PhysicalOperator::ParallelSource()` returns true).

---

## 4. Storage — PAX Format & Buffer Manager

### 4.1 Storage layout (PAX / column-group)

DuckDB stores data in a **PAX (Partition Attributes Across)** layout: the file is
divided into fixed-size **row groups** (default 122880 rows each), and within each row
group every column is stored as a contiguous sequence of **column segments** on separate
blocks.  This gives column-store scan performance while keeping related rows in the same
I/O region.

```
.duckdb file
└── Block 0: file header (magic, version, root meta pointer)
└── Block N: row group data
     ├── Column 0: ColumnSegment(s) – compressed data
     ├── Column 1: ColumnSegment(s) – compressed data
     └── ...
└── WAL: write-ahead log (.wal file alongside .duckdb)
```

**Files:** `src/storage/storage_manager.cpp`, `src/storage/table/row_group.cpp`,
`src/storage/table/column_data.cpp`, `src/storage/table/column_segment.cpp`

#### Key storage classes

| Class | File | Role |
|-------|------|------|
| `StorageManager` | `src/storage/storage_manager.cpp` | Top-level storage controller; owns block manager & WAL |
| `SingleFileStorageManager` | `src/storage/storage_manager.cpp` | Concrete file-backed storage manager |
| `SingleFileBlockManager` | `src/include/duckdb/storage/single_file_block_manager.hpp` | Manages `.duckdb` file blocks |
| `RowGroupCollection` | `src/storage/table/row_group_collection.cpp` | Ordered list of `RowGroup`s for one table |
| `RowGroup` | `src/storage/table/row_group.hpp` | One PAX row group (≤ row_group_size rows) |
| `ColumnData` | `src/storage/table/column_data.hpp` | Per-column storage within a row group |
| `ColumnSegment` | `src/storage/table/column_segment.cpp` | Contiguous compressed segment of one column |

#### `StorageManager::Initialize` / `LoadDatabase`

```cpp
// src/storage/storage_manager.cpp:328
void StorageManager::Initialize(QueryContext context) {
    LoadDatabase(context);          // line 335
}

// src/storage/storage_manager.cpp:370 (SingleFileStorageManager)
void SingleFileStorageManager::LoadDatabase(QueryContext context) {
    // New file:  sf_block_manager->CreateNewDatabase(context)  // line 451
    // Existing:  sf_block_manager->LoadExistingDatabase(context) // line 476
}
```

Block constants (`src/include/duckdb/storage/storage_info.hpp`):

```cpp
#define DEFAULT_BLOCK_ALLOC_SIZE  262144ULL   // 256 KiB per block
constexpr static idx_t SECTOR_SIZE = 4096U;  // alignment granularity
```

---

### 4.2 Buffer Manager

**Files:** `src/storage/buffer_manager.cpp`,
`src/include/duckdb/storage/standard_buffer_manager.hpp`,
`src/storage/buffer/block_handle.cpp`,
`src/storage/buffer/buffer_pool.cpp`

The `BufferManager` manages a pool of pinned in-memory blocks. It implements a
clock/LRU eviction policy and supports spilling evicted blocks to a temporary directory.

```
BufferManager (abstract base)
└── StandardBufferManager   – production implementation
     ├── BufferPool          – eviction queue + memory accounting
     └── BlockHandle         – reference-counted handle to one block
          └── FileBuffer      – owned raw memory for the block
```

#### Key methods on `StandardBufferManager`

```cpp
// src/include/duckdb/storage/standard_buffer_manager.hpp
BufferHandle Pin(shared_ptr<BlockHandle> &handle);     // load block into RAM, inc ref
void         Unpin(shared_ptr<BlockHandle> &handle);   // dec ref, eligible for eviction
BufferHandle Allocate(MemoryTag tag, idx_t block_size);// allocate a new in-memory block
```

`Pin` checks whether the block is already in memory (hot path: just increment the
reference count and return).  If the block is not resident, it calls the underlying
`FileBuffer` / block manager to read it from disk, potentially evicting cold blocks
first via `EvictBlocksOrThrow`.

#### `BlockHandle` lifecycle

```
BlockHandle::state:
  UNLOADED   → Pin() → LOADED (data in RAM)
  LOADED     → Unpin() → CAN_UNLOAD (eligible for eviction)
  CAN_UNLOAD → EvictBlocksOrThrow() → UNLOADED (data written to temp file if dirty)
```

---

### 4.3 Write-Ahead Log (WAL)

All mutations are first written to `<db>.wal` before the in-memory state is modified.
`StorageManager::WALStartCheckpoint` (`src/storage/storage_manager.cpp:204`) coordinates
a checkpoint: it flushes the WAL, writes a new consistent snapshot to the `.duckdb`
file, then truncates the WAL.

---

## 5. Data Flow — `SELECT * FROM table` End-to-End

The following trace shows how a simple `SELECT * FROM my_table` moves through the
system from the public API down to storage block reads.

```
ClientContext::Query("SELECT * FROM my_table")
  │
  ├─ 1. PARSE
  │    ClientContext::PendingStatementInternal()   (client_context.cpp:841)
  │    └─ Parser::ParseQuery(query)                (parser.cpp)
  │         └─ PostgresParser::Parse(query)        → PGSelectStmt tree
  │         └─ Transformer::TransformParseTree()   → SelectStatement
  │
  ├─ 2. BIND  (client_context.cpp:415)
  │    Planner::CreatePlan(SelectStatement)        (planner.cpp:50)
  │    └─ Binder::Bind(SelectStatement)            (binder.cpp:80)
  │         └─ Binder::Bind(SelectNode)            (bind_select_node.cpp)
  │              ├─ Resolves "my_table" in catalog → DuckTableEntry
  │              ├─ Creates LogicalGet(my_table)
  │              └─ Creates LogicalProjection(* expansion)
  │              → BoundStatement { plan=LogicalGet, types, names }
  │
  ├─ 3. OPTIMIZE  (client_context.cpp:443-446)
  │    Optimizer::Optimize(LogicalGet)             (optimizer.cpp)
  │    └─ RunBuiltInOptimizers()
  │         ├─ RemoveUnusedColumns  → strips unreferenced columns
  │         ├─ RowGroupPruner       → prunes row groups via zone maps
  │         └─ StatisticsPropagator → annotates plan with column stats
  │         → Optimized LogicalGet (with column projection + filters)
  │
  ├─ 4. PHYSICAL PLAN GENERATION  (client_context.cpp:457)
  │    PhysicalPlanGenerator::CreatePlan(LogicalGet)   (physical_plan/plan_get.cpp)
  │    └─ Creates PhysicalTableScan(my_table, columns)
  │         └─ table function = DataTable::ScanFunction
  │
  ├─ 5. PIPELINE BUILD + EXECUTE
  │    Executor::Initialize(PhysicalTableScan)
  │    └─ Builds one pipeline:
  │         Source: PhysicalTableScan
  │         Sink:   PhysicalMaterializedCollector (or streaming result)
  │
  │    Executor::ExecuteTask()  (executor.cpp)
  │    └─ Pipeline::Execute()
  │         └─ PhysicalTableScan::GetData()            (physical_table_scan.cpp:160)
  │              └─ TableFunction::function(...)        → calls storage scan
  │                   └─ DataTable::Scan(state, chunk)
  │                        └─ RowGroupCollection::Scan(state, chunk)
  │                             └─ RowGroup::Scan(state, chunk)  (row_group.cpp)
  │                                  └─ ColumnData::Scan(txn, vec_idx, state, result)
  │                                       └─ ColumnSegment reads from buffer:
  │
  ├─ 6. BUFFER MANAGER / STORAGE READ
  │         StandardBufferManager::Pin(block_handle)
  │         └─ If block not resident: FileBuffer::Read(block_id)
  │              └─ FileSystem::Read(fd, offset, size)
  │                   → 256 KiB block loaded into RAM
  │         Decompression (e.g. FSST, BitPacking, RLE)
  │         → Populates Vector<T> with up to 2048 values
  │
  └─ 7. RESULT DELIVERY
       DataChunk (2048 rows × N columns) bubbles back through the pipeline.
       PhysicalMaterializedCollector::Sink() appends it to a ColumnDataCollection.
       ClientContext::FetchResultInternal() → MaterializedQueryResult returned to caller.
```

### Cardinality at each stage

| Stage | Granularity |
|-------|-------------|
| SQL string | Entire query |
| `SQLStatement` | One statement |
| `LogicalOperator` tree | Full relation (lazy) |
| Physical pipeline | Unbounded stream of `DataChunk`s |
| `DataChunk` | Up to 2048 rows × N columns |
| `ColumnSegment` | ≤ row-group-size rows per column (default 122880) |
| Block | 256 KiB (default `DEFAULT_BLOCK_ALLOC_SIZE`) |

---

*This guide was generated from the source tree at commit HEAD and references files
under `src/`. Line numbers are approximate and may shift as the codebase evolves.*
