# Blazegraph RDF/SPARQL Named Graph Database: Technical Essentials

## System Overview

**Blazegraph** (formerly BigData) is an ultra high-performance RDF/SPARQL graph database supporting up to 50 billion edges on a single machine. It powers production systems at Fortune 500 companies and serves as the backend for Wikidata Query Service.

---

## 1. Core Data Structures

### 1.1 RDF Statement Representation (SPO)

**File:** `bigdata-core/bigdata-rdf/src/java/com/bigdata/rdf/spo/SPO.java`

```java
SPO {
    s: InternalValue    // Subject (64-bit IV)
    p: InternalValue    // Predicate (64-bit IV)
    o: InternalValue    // Object (64-bit IV)
    c: InternalValue    // Context/Named Graph (quad mode)
    sid: Long           // Statement identifier (optional)
    flags: byte         // Type, modified, user flags
}
```

**Key Design:**
- All RDF terms (URIs, literals, blank nodes) encoded as 64-bit **Internal Values (IVs)**
- Compact representation: 32 bytes per quad (with context)
- Statement types: `EXPLICIT`, `INFERRED`, `AXIOM`
- Lazy-initialized statement identifiers for reification

### 1.2 Multiple Index Key Orders

**File:** `bigdata-rdf/src/java/com/bigdata/rdf/spo/SPOKeyOrder.java`

**Triple Store (3 indices):**
- `SPO` - Subject-Predicate-Object
- `POS` - Predicate-Object-Subject
- `OSP` - Object-Subject-Predicate

**Quad Store (6 indices for named graphs):**
- `SPOC`, `POCS`, `OCSP` - Extends triple indices with context
- `CSPO`, `PCSO`, `SOPC` - Context-first orderings for graph queries

**Index Selection Logic:**
```
Pattern(s, p, o, c) → Select index with longest bound prefix
  Bound: S, P      → Use SPO
  Bound: P, O      → Use POS
  Bound: O         → Use OSP
  Bound: C         → Use CSPO (quad store)
```

### 1.3 B+-Tree Implementation

**File:** `bigdata-core/bigdata/src/java/com/bigdata/btree/BTree.java`

**Architecture:**
- **Copy-on-write** B+-tree (lock-free reads)
- **No leaf linking** (avoids reference cycles)
- **Values stored exclusively in leaves**
- **Variable-size nodes** for I/O efficiency
- **Optional bloom filters** for point lookups
- **Leading key compression**

**Node Structure:**
```
AbstractNode (internal)
  ├── keys[]          // Separator keys
  ├── childAddrs[]    // Disk addresses of children
  └── branchingFactor // Variable branching

Leaf (storage)
  ├── keys[]          // Indexed keys
  ├── values[]        // Associated values
  └── version         // MVCC version
```

**Concurrency:**
- **Thread-safe for concurrent readers**
- **Single writer** (MVCC isolation)
- **Adaptive packed arrays** for mutations

### 1.4 Lexicon (Term Dictionary)

**File:** `bigdata-rdf/src/java/com/bigdata/rdf/lexicon/LexiconRelation.java`

**Purpose:** Bidirectional encoding of RDF terms ↔ Internal Values

**Indices:**
- `TERM2ID` - RDF term → IV (primary)
- `ID2TERM` - IV → RDF term (reverse lookup)
- `BLOBS` - Large object storage (long URIs/literals)
- Full-text indices (optional)

**Inline Terms:**
- Small URIs/literals stored directly in IV representation
- Reduces storage and improves cache efficiency
- Example: `xsd:int(42)` stored inline without lexicon lookup

---

## 2. Disk Storage & Persistence

### 2.1 Journal-Based Storage

**File:** `bigdata-core/bigdata/src/java/com/bigdata/journal/Journal.java`

**Structure:**
```
Journal
├── Root Block          // Metadata, commit records, UUIDs
├── Write Buffer        // Uncommitted B+-tree mutations
├── Index Segments      // Read-only materialized indices
└── Transaction Log     // Recovery log
```

**Root Block Contents:**
- Checkpoint metadata
- Index addresses (pointers to B+-tree roots)
- Commit records (transaction IDs, timestamps)
- Database UUID

**Write-Once-Read-Many (WORM):**
- New data written to end of journal
- Old versions retained for MVCC snapshots
- Copy-on-write semantics for all updates

### 2.2 Buffer Strategies

**File:** `bigdata/src/java/com/bigdata/journal/`

**Available Strategies:**
1. `DirectBufferStrategy` - Direct memory (off-heap)
2. `MappedBufferStrategy` - OS memory mapping (mmap)
3. `DiskBackedBufferStrategy` - Disk with cache
4. `TransientBufferStrategy` - In-memory only
5. `WORMStrategy` - Write-once append-only

### 2.3 Transaction Management

**ACID Guarantees:**
- **Atomicity:** All-or-nothing commits via journal
- **Consistency:** Constraint validation
- **Isolation:** Multi-Version Concurrency Control (MVCC)
- **Durability:** fsync on commit (configurable)

**Transaction Types:**
- `READ_COMMITTED` - Read snapshot at transaction start
- `UNISOLATED` - Read latest committed data
- Write transactions with optimistic locking

**Group Commit:**
- Batch multiple transactions into single fsync
- Reduces I/O overhead for high write throughput

---

## 3. Memory Management

### 3.1 Weak Reference Caching

**File:** `bigdata-cache/src/main/java/com/bigdata/cache/ConcurrentWeakValueCache.java`

**Design:**
```java
ConcurrentWeakValueCache<K, V>
  ├── ConcurrentHashMap<K, WeakReference<V>>
  ├── HardReferenceQueue<V>     // MRU protection
  └── ReferenceQueue<V>          // GC tracking
```

**Strategy:**
- **Weak references:** Automatic eviction via GC
- **Hard reference queue:** Keep N most-recently-used alive
- **Lock-free:** Concurrent access without blocking
- **Self-cleaning:** Reference queue processes GC'd entries

**Benefits:**
- Automatic memory adaptation to heap pressure
- No manual cache sizing
- Low contention for concurrent queries

### 3.2 Write Retention Queue

**Purpose:** Minimize random I/O during commits

**Strategy:**
1. Keep recently written B+-tree nodes in memory
2. Buffer mutations until commit
3. Evict in post-order traversal (children before parents)
4. Write siblings together for disk locality

**Effect:**
- Converts random writes → sequential writes
- Reduces disk seeks during commits
- Improves SSD/HDD write throughput

### 3.3 Index Segments

**Read-Only Materialized Views:**
- Snapshot immutable B+-trees to disk
- Compress and optimize for read access
- No write retention overhead
- Memory-mapped for fast access
- Background merge process (compaction)

---

## 4. Query Processing Algorithms

### 4.1 SPARQL Pipeline

**File:** `bigdata-rdf/src/java/com/bigdata/rdf/sparql/`

```
SPARQL Query String
    ↓
  Parser (Sesame/OpenRDF)
    ↓
  SPARQL AST
    ↓
  53 Optimization Passes
    ↓
  AST2BOp Conversion
    ↓
  QueryEngine (Pipelined Execution)
    ↓
  Results
```

### 4.2 Access Path Selection

**Algorithm:** Prefix-based index selection

```python
def select_access_path(pattern):
    # pattern = (s, p, o, c) with variables or constants
    bound_positions = count_bound(pattern)

    # Select index maximizing bound prefix
    if pattern.s.is_bound():
        if pattern.p.is_bound():
            return SPO_INDEX  # Longest prefix: S, P
        else:
            return SPO_INDEX  # Prefix: S
    elif pattern.p.is_bound():
        if pattern.o.is_bound():
            return POS_INDEX  # Longest prefix: P, O
        else:
            return POS_INDEX  # Prefix: P
    elif pattern.o.is_bound():
        return OSP_INDEX      # Prefix: O
    else:
        return SPO_INDEX      # Full scan (any index)
```

**Optimization:**
- Bloom filter check before range scan
- Skip index if bloom filter returns false
- Reduces I/O for non-existent patterns

### 4.3 Join Ordering Optimization

**File:** `bigdata-rdf/src/java/com/bigdata/rdf/sparql/ast/optimizers/ASTJoinGroupOrderOptimizer.java`

**Algorithm:** Cardinality-based dynamic programming

```python
def optimize_join_order(patterns):
    # 1. Estimate cardinality for each pattern
    for pattern in patterns:
        pattern.cardinality = estimate_cardinality(pattern)

    # 2. Order by selectivity (low cardinality first)
    patterns.sort(key=lambda p: p.cardinality)

    # 3. Adjust for variable binding
    ordered = []
    remaining = patterns[:]

    while remaining:
        # Prefer patterns that bind variables used later
        best = select_best_next(remaining, ordered)
        ordered.append(best)
        remaining.remove(best)

    return ordered
```

**Cardinality Estimation:**
```python
def estimate_cardinality(pattern):
    # Use B+-tree statistics
    if pattern fully bound:
        return 1  # Point lookup
    elif pattern has bound prefix:
        # Range scan: estimate from index
        index = select_index(pattern)
        return index.estimate_range_count(pattern.prefix)
    else:
        # Full scan: return total statement count
        return total_statements
```

**Cost Model:**
**File:** `bigdata/src/java/com/bigdata/bop/cost/BTreeCostModel.java`

```python
def estimate_btree_cost(n_entries, branching_factor, range_fraction):
    # Tree height (excluding root in memory)
    height = log(n_entries, branching_factor) - 1

    # Random I/O cost for tree traversal
    traversal_cost = height * RANDOM_IO_COST

    # Sequential scan cost for range
    scan_cost = (n_entries * range_fraction) * SEQUENTIAL_IO_COST

    return traversal_cost + scan_cost
```

### 4.4 Key Optimizations (53 Total)

**Join Optimizations:**
- `ASTJoinGroupOrderOptimizer` - Cardinality-based ordering
- `ASTHashJoinOptimizer` - Hash join vs. nested-loop selection
- `ASTStaticJoinOptimizer` - Static binding analysis

**Filter Optimizations:**
- Push filters close to source (early pruning)
- Evaluate cheap filters first
- Short-circuit boolean evaluation

**Subquery Optimizations:**
- `ASTSparql11SubqueryOptimizer` - Flatten or separate execution
- Hoist subqueries when possible

**Graph Pattern Optimizations:**
- `ASTFlattenJoinGroupsOptimizer` - Flatten nested groups
- `ASTGraphGroupOptimizer` - Named graph context optimization

**Range Optimizations:**
- `ASTRangeOptimizer` - Identify range queries
- `ASTRangeCountOptimizer` - COUNT(*) optimization

**Full-Text Search:**
- `ASTFulltextSearchOptimizer` - Integration with text indices

### 4.5 BOp Execution Model

**File:** `bigdata/src/java/com/bigdata/bop/engine/QueryEngine.java`

**Pipelined Execution:**
```
AccessPathOp → FilterOp → JoinOp → ProjectOp → Results
     ↓             ↓          ↓          ↓
  Chunks       Chunks     Chunks     Chunks
```

**Chunked Processing:**
- Data organized in chunks (e.g., 10,000 solutions)
- Streaming between operators
- Memory-efficient for large result sets
- Configurable chunk sizes

**Distributed Execution:**
- Scale-out support for multi-node deployment
- Data partitioning across shards
- Query routing and aggregation

---

## 5. Named Graph Support

### 5.1 Quad Store Configuration

**Enable:** `AbstractTripleStore.Options.QUADS = true`

**Index Expansion:**
```
Triple Store: 3 indices (SPO, POS, OSP)
Quad Store:   6 indices (SPOC, POCS, OCSP, CSPO, PCSO, SOPC)
```

**Context Position:**
- Fourth position in key = named graph URI (encoded as IV)
- Enables efficient graph-level operations

### 5.2 SPARQL Graph Operations

**File:** `bigdata-rdf/src/java/com/bigdata/rdf/sparql/ast/`

**Operations:**
- `CreateGraph` - Create named graph
- `DropGraph` - Delete named graph
- `ClearGraph` - Clear graph contents
- `LoadGraph` - Load RDF into graph
- `DeleteInsertGraph` - INSERT/DELETE updates
- `AddGraph`, `MoveGraph`, `CopyGraph` - Graph management

### 5.3 Query Semantics

**GRAPH Pattern:**
```sparql
SELECT ?s ?p ?o ?g WHERE {
  GRAPH ?g {
    ?s ?p ?o
  }
}
```

**Execution:**
1. Bind `?g` to graph URIs (scan CSPO with pattern `(?g, ?, ?, ?)`)
2. For each `?g`, bind `?s ?p ?o` (scan SPOC with pattern `(?, ?, ?, ?g)`)
3. Use CSPO or SPOC based on query pattern binding

**FROM/FROM NAMED:**
- Specify default graph and named graphs
- Optimizer rewrites to GRAPH patterns
- `ASTGraphGroupOptimizer` - Context optimization

### 5.4 Multi-Tenancy

**Namespace Isolation:**
- Multiple independent RDF databases per instance
- Separate index sets per namespace
- Shared or isolated lexicons (configurable)

---

## 6. Performance Characteristics

### Strengths
1. **Multiple index orderings** - O(log n) access for any pattern
2. **Copy-on-write** - Lock-free concurrent reads
3. **Bloom filters** - Fast non-existence checks (no I/O)
4. **Weak reference caching** - Automatic GC-based memory management
5. **53 optimization passes** - Sophisticated query planning
6. **MVCC** - Snapshot isolation without locks
7. **Index segments** - Read-optimized immutable snapshots

### Trade-offs
1. **Storage overhead** - 3-6x for multiple index orderings
2. **Write amplification** - Copy-on-write for all mutations
3. **Single writer** - No concurrent write transactions
4. **Rebalancing cost** - B+-tree maintenance on updates

### Scalability
- **Single machine:** 50+ billion triples
- **Scale-out:** Multi-node distributed deployment
- **Wikidata:** 12+ billion statements in production

---

## 7. Key File Locations

| Component | File Path |
|-----------|-----------|
| B+-Tree | `bigdata-core/bigdata/src/java/com/bigdata/btree/BTree.java` |
| SPO Relation | `bigdata-core/bigdata-rdf/src/java/com/bigdata/rdf/spo/SPORelation.java` |
| Index Orders | `bigdata-core/bigdata-rdf/src/java/com/bigdata/rdf/spo/SPOKeyOrder.java` |
| Lexicon | `bigdata-core/bigdata-rdf/src/java/com/bigdata/rdf/lexicon/LexiconRelation.java` |
| Journal | `bigdata-core/bigdata/src/java/com/bigdata/journal/Journal.java` |
| Cache | `bigdata-cache/src/main/java/com/bigdata/cache/ConcurrentWeakValueCache.java` |
| SPARQL AST | `bigdata-core/bigdata-rdf/src/java/com/bigdata/rdf/sparql/ast/` |
| Optimizers | `bigdata-core/bigdata-rdf/src/java/com/bigdata/rdf/sparql/ast/optimizers/` |
| Query Engine | `bigdata-core/bigdata/src/java/com/bigdata/bop/engine/QueryEngine.java` |
| SAIL API | `bigdata-core/bigdata-sails/src/java/com/bigdata/rdf/sail/BigdataSail.java` |

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│         SPARQL Query (BigdataSail API)              │
└────────────────────┬────────────────────────────────┘
                     │
        ┌────────────▼────────────┐
        │   AST + 53 Optimizers   │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │  BOp QueryEngine        │
        │  (Pipelined Execution)  │
        └────────────┬────────────┘
                     │
         ┌───────────┴───────────┐
         │                       │
┌────────▼────────┐    ┌────────▼────────┐
│  SPORelation    │    │ LexiconRelation │
│  (RDF Indices)  │    │ (Term Dict)     │
│                 │    │                 │
│  SPO POS OSP    │    │ TERM2ID ID2TERM │
│  (or SPOC...)   │    │ BLOBS FT-Index  │
└────────┬────────┘    └────────┬────────┘
         │                      │
         └──────────┬───────────┘
                    │
         ┌──────────▼──────────┐
         │   B+-Tree Indices   │
         │  (Copy-on-Write)    │
         └──────────┬──────────┘
                    │
         ┌──────────▼──────────┐
         │ Weak Reference Cache │
         └──────────┬──────────┘
                    │
         ┌──────────▼──────────┐
         │  Journal (MVCC)     │
         │  Write Buffer       │
         │  Index Segments     │
         └──────────┬──────────┘
                    │
         ┌──────────▼──────────┐
         │   RawStore (Disk)   │
         └─────────────────────┘
```

---

## Summary

Blazegraph is a production-grade RDF database built on:
- **Data Structures:** Copy-on-write B+-trees with multiple index orderings, lexicon-based term encoding
- **Disk:** Journal-based MVCC with WORM semantics, index segments for read optimization
- **Memory:** Weak reference caching with GC-based eviction, write retention queues for I/O batching
- **Algorithms:** Cardinality-based join ordering, prefix-based index selection, 53 query optimizations, pipelined execution

The system achieves high performance through sophisticated indexing, lock-free concurrency, and cost-based query optimization while maintaining full SPARQL 1.1 compliance with named graph support.
