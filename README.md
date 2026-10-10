# db

[![CI](https://github.com/alya-lang/db/actions/workflows/ci.yml/badge.svg)](https://github.com/alya-lang/db/actions/workflows/ci.yml)
[![License](https://img.shields.io/github/license/alya-lang/db?color=blue&label=License)](LICENSE)
[![Alya](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Fdb%2Fmain%2Falya.toml&query=%24.package.alya-version&label=Alya&color=orange&prefix=%3E%3D)](https://github.com/alya-lang/alya)
[![Package Version](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Fdb%2Fmain%2Falya.toml&query=%24.package.version&label=Version&color=brightgreen)](alya.toml)

Database driver interface, connection management, and query abstraction for Alya

---

## 🌟 Features

- 🔌 **Driver Plug-in Model**: Databases plug in as plain `Driver` structs (four function hooks, no registry, no globals). Adding postgres or mysql tomorrow means constructing one more `Driver` — this package is never revised.
- 🔒 **Explicitly-Typed Params**: `?` placeholders bind from `Param` values (`param_int`, `param_float`, `param_text`, `param_null`, `param_blob`), so quoting and injection handling are structural, not stringly.
- 🧾 **Structured Errors**: Drivers throw catchable `DbError` structs (`code` + `message`) instead of raw strings.
- 🔄 **Transactions Through One Facade**: `begin`/`commit`/`rollback` run over `execute`, so every driver gets them with zero extra surface.
- 🪶 **Zero-Dependency Core**: Pure Alya, no C sources, no driver imports. Adapters live in driver packages and depend on `db`, never the reverse.
- 🔗 **Bundled SQLite Adapter**: Optional `sqlite` slice (`src/adapters/`) binds the contract to `alya-lang/sqlite` with error normalization into `DbError`.
- 🧪 **Mock-Tested Contract**: `tests/test_basic.alya` ships a complete in-memory mock driver proving every facade call, so future driver authors have a working reference.
- 🏊 **Explicit Connection Pooling**: Bounded `ConnPool` reuse without closures (`pooled_open`, `acquire`/`release`/`invalidate`, counts, drain). Needs no `pool` dependency: driver and path travel explicitly, which callback factories cannot express.

---

## 📁 Project Architecture

```text
db/
├── alya.toml               # Package manifest (no required dependencies)
├── src/
│   ├── lib.alya            # Public API facade (open/close/execute/query/scalars/transactions + Conn methods)
│   └── types.alya          # ParamKind, Param + param_* ctors, DbError, Driver, Conn
├── examples/
│   └── demo.alya           # Runnable walkthrough on a minimal in-memory driver
├── tests/
│   └── test_basic.alya     # Contract suite incl. a full mock driver (KV store with real rollback)
└── benches/
    └── bench_basic.alya    # Facade + dispatch micro-benchmarks (stub driver)
```

> [!NOTE]
> **Visibility & Modularity:** Symbols annotated with `pub` are exported to external consumers. Symbols without `pub` remain strictly internal to their declaring module.

---

## 📦 Installation

Add `db` to the `[dependencies]` section in your `alya.toml`:

```toml
[dependencies]
db = { git = "https://github.com/alya-lang/db", branch = "main" }
```

Or install it directly using the Alya package CLI:

```bash
alya add db --git https://github.com/alya-lang/db --branch main
alya install
```

### Package Features

| Feature | Default | Description |
|:---|:---:|:---|
| `sqlite` | ✅ | Bundled `sqlite` adapter (`sqlite_driver()` in `src/adapters/sqlite_adapter.alya`, with a fallback-free skip when disabled). |

```bash
# Full build (default)
alya install
alya test

# Slim build (core only, no sqlite dependency)
alya install --no-default-features
alya test --no-default-features
```

---

## 🚀 Quick Start

```alya
import "db" as pkg

# A minimal driver is four functions; real drivers do real I/O here.
function mem_open(path: string)
    return map()
end

function mem_close(handle) -> int
    return 0
end

function mem_execute(handle, sql: string, params: array) -> int
    return 1
end

function mem_query(handle, sql: string, params: array) -> array
    return [{ "v": 41 }]
end

function main()
    let driver = pkg::Driver { name: "mem", open_fn: mem_open, close_fn: mem_close, execute_fn: mem_execute, query_fn: mem_query }

    # 1. Open a connection through the driver value
    let conn = pkg::open(driver, ":memory:")

    # 2. Parameterized write (quotes need no escaping)
    pkg::execute(conn, "INSERT INTO kv (k, v) VALUES (?, ?);", [pkg::param_text("name"), pkg::param_text("O'Brien")])

    # 3. Scalar read
    let visits = pkg::query_scalar(conn, "SELECT v FROM kv WHERE k = ?;", [pkg::param_text("visits")])
    say f"Visits: {visits}"

    # 4. Close
    pkg::close(conn)
end

main()
```

---

## 📖 API Reference

### Connections

| Function | Arguments | Returns | Description |
|---|---|---|---|
| `open(driver, path)` | `driver: Driver, path: str` | `Conn` | Opens a connection through the driver's `open_fn` hook. |
| `close(conn)` | `conn: Conn` | `i32` | Closes the connection via `close_fn`; double close safely returns `0`. |
| `Conn.execute(self, sql, params)` | `sql: str, params: Array` | `i32` | Method form of `execute`. |
| `Conn.query(self, sql, params)` | `sql: str, params: Array` | `Array` | Method form of `query`. |
| `Conn.query_scalar(self, sql, params)` | `sql: str, params: Array` | `value` | Method form of `query_scalar`. |
| `Conn.begin/commit/rollback(self)` | none | `i32` | Method forms of the transaction helpers. |
| `Conn.close(self)` | none | `i32` | Method form of `close`. |

### Query Execution

| Function | Arguments | Returns | Description |
|---|---|---|---|
| `execute(conn, sql, params)` | `conn: Conn, sql: str, params: Array<Param>` | `i32` | Runs a statement with bound `?` parameters; returns affected row count. |
| `query(conn, sql, params)` | `conn: Conn, sql: str, params: Array<Param>` | `Array` | Runs a `SELECT` with bound params; returns row maps. |
| `query_scalar(conn, sql, params)` | `conn: Conn, sql: str, params: Array<Param>` | `value` | First column of the first row, or null when empty. |
| `begin(conn)` / `commit(conn)` / `rollback(conn)` | `conn: Conn` | `i32` | Transaction control over `execute`. |

### Parameters & Models

| Symbol | Kind | Description |
|---|---|---|
| `param_int(v)` / `param_float(v)` / `param_text(v)` / `param_null()` / `param_blob(v)` | `pub function` | Explicitly-typed `?` placeholder constructors. |
| `ParamKind` | `pub enum` | Placeholder kinds (`Int = 1`, `Float = 2`, `Text = 3`, `Null = 4`, `Blob = 5`). |
| `Param` | `pub struct` | Bound parameter (`kind`, `ival`, `fval`, `sval`, `bval`). |
| `DbError` | `pub struct` | Thrown error payload (`code`, `message`). |
| `db_error(code, message)` | `pub function` | Constructs a throwable `DbError`. |
| `Driver` | `pub struct` | Driver contract (`name`, `open_fn`, `close_fn`, `execute_fn`, `query_fn`). |
| `Conn` | `pub struct` | Open connection (`driver`, `handle`, `path`, `is_open`, `pool_id`). |
| `sqlite_driver()` | `pub function` (`sqlite` feature, default-on) | Builds the sqlite-backed `Driver` (see `src/adapters/sqlite_adapter.alya`). |
| `pooled_open(driver, path, max_size)` | `pub function` | Opens a bounded `ConnPool` (lazy connections, `max_size` must be positive). |
| `ConnPool` | `pub struct` | Connection pool (`driver`, `path`, `max_size`, `idle`, `active`, `next_id`, `closed`). |
| `ConnPool.acquire(self)` | `pub method` | Reuses idle or opens new below cap; null when closed/exhausted. |
| `ConnPool.release(self, conn)` | `pub method` | Shelves the loan (drops externally closed ones). |
| `ConnPool.invalidate(self, conn)` | `pub method` | Closes and discards a broken loan. |
| `ConnPool.close(self)` | `pub method` | Drains idle loans; returns drained count. |
| `ConnPool.size/idle_count/active_count(self)` | `pub method` | Pool gauges. |

> [!TIP]
> **Internal Helpers & Documentation:** Public symbols carry `##` Markdown docstrings for `alya doc`. Drivers convert `Param` to native binds inside their adapter; see the mock driver in `tests/test_basic.alya` for the reference implementation.

---

## 🧪 Running Tests & Benchmarks

Run the automated test suite using `alya test`:

```bash
alya test
```

Generate static API documentation:

```bash
alya doc . -o docs --markdown
```

Run the benchmark suite:

```bash
alya run benches/bench_basic.alya
```

Run the example demo:

```bash
alya run examples/demo.alya
```

Check code formatting:

```bash
alya fmt . --check
```

Run static code linter:

```bash
alya lint . --check
```

---

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository and clone it locally
2. Install dependencies:
   ```bash
   alya install
   ```
3. Create your feature branch (`git checkout -b feature/my-feature`)
4. Verify tests and formatting before opening a PR:
   ```bash
   alya test
   alya fmt . --check
   ```
5. Commit your changes (`git commit -m "feat: add feature"`) and open a Pull Request

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
