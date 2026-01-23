# sql_optimizer_rs

Rust-based SQL query analyzer / optimizer (MVP).

## Build

```bash
cargo build -p sqlopt
```

## CLI usage

```bash
# Suggest an index
cargo run -p sqlopt -- analyze "SELECT * FROM users WHERE age > 18"

# Detect repeated query templates (common N+1 symptom)
# (flags are counts-per-window-of-N-queries, not timestamps)
cargo run -p sqlopt -- detect-n1 examples/queries.log --threshold 5 --window 50

# Heuristic rewrite (JOIN -> IN subquery) for simple filter joins
cargo run -p sqlopt -- rewrite "SELECT o.* FROM orders o JOIN users u ON o.user_id = u.id WHERE u.active = true"
```

## Examples

`examples/queries.log` contains 847 `SELECT * FROM posts WHERE user_id = ...` lines to exercise `detect-n1`.

## Sample output

These are copied from real runs of the commands above.

```text
$ cargo run -p sqlopt -- analyze "SELECT * FROM users WHERE age > 18"
SUGGESTION: CREATE INDEX CONCURRENTLY idx_users_age ON users(age);
ESTIMATED: 1164ms -> 8ms (99%)
```

```text
$ cargo run -p sqlopt -- detect-n1 examples/queries.log --threshold 5 --window 50
CRITICAL: Detected repeated query templates (threshold=5, window=50).
COUNT=50 TOTAL=847 TEMPLATE=select * from posts where user_id = ?
```

```text
$ cargo run -p sqlopt -- rewrite "SELECT o.* FROM orders o JOIN users u ON o.user_id = u.id WHERE u.active = true"
ORIGINAL: SELECT o.* FROM orders o JOIN users u ON o.user_id = u.id WHERE u.active = true
WARNING: avoid wildcard projections (SELECT *); select only needed columns
WARNING: heuristic rewrite: verify projection if you relied on JOINed columns
OPTIMIZED: SELECT o.* FROM orders o WHERE o.user_id IN (SELECT id FROM users WHERE active = true)
```

## Comparison (rough)

Notes:
- The `sqlopt` speed is an example from the `index_suggestion` benchmark. Results vary by machine/runner; see CI artifacts in the [CI workflow](https://github.com/sqlopt-rs/sql_optimizer_rs/actions/workflows/ci.yml) for latest data.
- `sqlopt`'s N+1 detection is log-based (repeated query templates), not ORM-aware instrumentation.
- `sqlparser-rs` is a parser library; a speed comparison is not applicable.

| Tool | Language | Speed (1M queries) | N+1 Detect | Index Suggest | CLI |
| --- | --- | ------------------: | :---: | :---: | :---: |
| [sqlopt](https://github.com/sqlopt-rs/sql_optimizer_rs) | Rust | ~8.9s | Yes (heuristic) | Yes (heuristic) | Yes |
| [sqlparser-rs](https://github.com/sqlparser-rs/sqlparser-rs) | Rust | N/A | No | No | No |
| [Bullet](https://github.com/flyerhzm/bullet) | Ruby | N/A | Yes | No | No |
| [Prosopite](https://github.com/charkost/prosopite) | Ruby | N/A | Yes | No | No |

## Benchmarks

Run the index suggestion benchmark:

```bash
cargo bench --bench index_suggestion -- 1_000_000
```

Result (this machine):

| Workload | Iterations | Elapsed (s) | Throughput (queries/s) |
| --- | ---: | ---: | ---: |
| index_suggestion | 1,000,000 | 8.864s | 112,814 queries/s |

At 1,000 queries/s, processing 1,000,000 queries takes ~16.7 minutes.

See `DESIGN.md` for the MVP boundaries and testing approach.
