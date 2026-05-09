# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

BD2-Proyecto-1 is an academic mini DBMS implemented in Python. It manages data through structured indexing techniques on paginated secondary memory (disk). There is no external database—all indexing (B+ Tree, Sequential File, Extendible Hashing, R-Tree) is implemented from scratch using raw file I/O with fixed 4096-byte pages.

## Build & Run Commands

### Setup (Nix)
```bash
nix develop   # auto-creates venv and installs requirements.txt
```

### Setup (Manual)
```bash
python3.11 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### Run API
```bash
uvicorn main:app --reload          # dev with auto-reload (port 8000)
```

### Docker
```bash
docker-compose up --build          # API at http://localhost:8000
```

### Run Tests
```bash
python tests/test_dbengine.py        # DB engine integration tests (110 tests)
python tests/test_bplus.py           # B+ Tree unit tests (38 tests)
python tests/test_rtree.py           # R-Tree unit tests (34 tests)
python tests/test_concurrency.py     # Concurrency/lock tests (31 tests)
python src/parser/run_all_inputs.py  # Parser tests (8 SQL inputs)
```

All tests are standalone scripts (no pytest), run individually with `python <file>`.

## Architecture

```
SQL string → Parser (scanner→parser→AST) → DBVisitor → DataBase → Storage+Indices → Disk
```

### Directory Structure

```
src/
├── api/            # Orchestrator between parser and indexes
│   └── dbengine.py     # DataBase class + execute_sql()
├── storage/        # Page management and physical file I/O
│   ├── pagemanager.py  # Low-level 4096B page read/write
│   ├── heapfile.py     # Record serialization, soft delete, free slots
│   ├── schema.py       # JSON schema metadata persistence
│   └── external_sort.py # Two-pass multiway merge sort (TPMMS)
├── structures/     # Index data structures (all disk-backed)
│   ├── bplus.py        # B+ Tree (ordered, range queries, linked leaves)
│   ├── sequentialfile.py # Sequential File: clustered (PK) or secondary index
│   ├── Extendible_Hashing.py # Dynamic hashing with expandable directory
│   └── rtree.py        # R-Tree for 2D spatial queries (radius, k-NN)
├── parser/         # SQL grammar and parser
│   ├── lexer_token.py  # Token types enumeration
│   ├── scanner.py      # Lexical analysis (tokenizer)
│   ├── ast_nodes.py    # AST node class definitions
│   ├── parser.py       # Recursive descent parser
│   ├── visitor.py      # Abstract visitor + TraceVisitor (debug/traza)
│   ├── db_visitor.py   # Concrete visitor for DB execution
│   └── main.py         # Parser entry point
└── concurrency/    # Transaction simulation and locks
    └── concurrency.py  # PageLockManager, 2PL, deadlock detection

tests/              # All test scripts
frontend/           # UI (React, Tkinter, etc.)
data/               # Runtime heap files (.bin)
docs/               # Technical documentation
main.py             # FastAPI REST API entry point
```

### Import Convention

All imports use `src.` prefix for absolute imports:
- `from src.storage.pagemanager import PageManager`
- `from src.structures.bplus import BPlusTree`
- `from src.api.dbengine import DataBase`
- Parser internal modules use relative imports (e.g., `from .scanner import *`)

### Layers

1. **API Layer** (`main.py`): FastAPI REST endpoints. POST `/query` executes SQL, POST `/csv/data` uploads CSVs.

2. **Orchestrator** (`src/api/dbengine.py`): `DataBase` class is the central facade. Manages schema persistence (JSON), index lifecycle, record CRUD, and I/O metric tracking (`disk_reads`/`disk_writes`).

3. **Parser** (`src/parser/`): Recursive descent parser converts SQL to AST nodes. Uses the Visitor pattern—`db_visitor.py` walks the AST and calls `DataBase` methods. Grammar defined in `src/parser/ebnf.md`.

4. **Storage** (`src/storage/`): PageManager for low-level 4096B page I/O, HeapFile for record serialization with soft delete, SchemaManager for JSON metadata, ExternalSort for TPMMS.

5. **Structures** (`src/structures/`): B+ Tree, Sequential File, Extendible Hashing, R-Tree. All use PageManager for disk-backed pages.

6. **Concurrency** (`src/concurrency/`): PageLockManager with shared/exclusive locks, wait-for graph deadlock detection, ConcurrentBPlusTree, Transaction class with strict 2PL.

### Storage Modes: Clustered Sequential vs HeapFile

The system supports two storage modes determined by `pk_index_type`:

**Clustered Sequential (pk_index_type="sequential")**:
- The SequentialFile stores FULL RECORDS sorted by the primary key
- No HeapFile is created — the SF IS the table storage
- Secondary indexes (B+, Hash, RTree) point to (page, slot) in the SF
- After reconstruction or delete, secondary indexes are rebuilt automatically
- `DataBase("t", schema, primary_key="id", pk_index_type="sequential")`

**HeapFile (pk_index_type="bplus", default)**:
- HeapFile stores records unordered
- ALL indexes (including PK) are secondary, storing (key, RID) pointing to HeapFile
- `DataBase("t", schema, primary_key="id")` — default B+ Tree on PK

SQL convention: `INDEX SEQUENTIAL` on a column makes it the clustered PK.

### File Layout on Disk (Runtime)
- `data/*.bin` — Heap pages (only when NOT using clustered sequential)
- `indexes/*.idx` — Index pages (and clustered data when using sequential PK)
- `schemas/*.json` — Table schema metadata

### Data Types
- INT (4B), FLOAT (8B), VARCHAR(N) (N bytes), POINT (two FLOATs, spatial)

### Index Types
- `"bplus"` — B+ Tree, non-clustered (equality + range)
- `"sequential"` — Sequential File, clustered for PK (records sorted by key)
- `"hash"` — Extendible Hashing, non-clustered (equality only)
- `"rtree"` — R-Tree, non-clustered (spatial, requires POINT column)

## Key Design Decisions

- All structures use fixed 4096-byte pages for fair I/O comparison
- PageManager tracks `disk_reads`/`disk_writes` for benchmarking
- B+ Tree leaves are linked for efficient range traversal
- Sequential File reconstructs (merges aux into main) when overflow exceeds threshold; triggers secondary index rebuild via `on_reconstruct` callback
- HeapFile uses soft delete (deleted flag) with stable RID slots
- SequentialFile clustered mode uses soft delete (flag byte per slot), keeping RIDs stable. Secondary indexes are updated individually on delete, not rebuilt. Deleted entries are compacted during reconstruction.
- No JOINs, aggregates, UPDATE, or query optimizer—scope is single-table operations

## SQL Dialect

```sql
CREATE TABLE t (col1 INT INDEX SEQUENTIAL, col2 VARCHAR(50) INDEX BTREE, ...) [FROM FILE 'path.csv']
SELECT * FROM t WHERE col = val | col BETWEEN a AND b | col IN (POINT(x,y), RADIUS r | K k)
INSERT INTO t VALUES (v1, v2, ...)
DELETE FROM t WHERE col = val
```
