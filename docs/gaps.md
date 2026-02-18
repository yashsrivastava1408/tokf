# tokf Engine Gaps

Filters that RTK implements but tokf cannot yet express in TOML.
Each gap includes: what it blocks, minimum fix, and estimated effort.

---

## Gap 1 — Per-line regex replacement

**What it is:** Apply a regex + template substitution to every output line (sed-like).
The current `extract` field only matches the _first_ matching line.

**Blocked filters:**
- `pnpm outdated` — reformat `pkg  old  wanted  new` columns to `pkg: old → new`
- `pip outdated` — same
- `wc` — strip filename column, reformat whitespace
- `docker ps` — trim alignment padding per row

**Proposed TOML syntax:**
```toml
[[replace]]
pattern = '^(\S+)\s+\S+\s+(\S+)\s+(\S+)'
output = "{1}: {2} → {3}"
```

Multiple `[[replace]]` entries run in order. Applied before `skip`/`keep`.

**Effort:** Medium — new `replace` filter step in `src/filter/replace.rs`, new
`Vec<ReplaceRule>` field on `FilterConfig`, analogous to `skip`.

---

## Gap 2 — JSON / structured-data parsing

**What it is:** Parse JSON (or NDJSON) output and extract fields from structured data.

**Blocked filters:**
- `go test -json` (NDJSON: one event object per line)
- `vitest --reporter=json`
- `eslint --format=json`
- `ruff --output-format=json`
- `pylint --output-format=json2`
- `golangci-lint --format=json`
- `gh api` responses

**Options:**
1. Add a `jq`-style extraction field to TOML (limited, hard to generalise)
2. Phase 4 Lua escape hatch — receive raw output as string, return filtered string

**Recommendation:** Defer to Phase 4 Lua (Issue #14). Lua resolves Gaps 2, 4, and 5 in one
feature rather than three separate engine extensions.

**Effort:** High (bespoke JSON field) / deferred to Phase 4 (Lua).

---

## Gap 3 — Stateful line deduplication

**What it is:** Collapse consecutive identical (or near-identical) lines into a single line
with a repeat count, e.g. `[repeated 47 times]`.

**Blocked filters:**
- `docker logs` — high-frequency repeated log lines
- `kubectl logs` — same
- Generic log file summarisation (`rtk log`)

**Proposed TOML syntax:**
```toml
dedup = true           # collapses consecutive identical lines
dedup_window = 10      # optional: compare within N-line window (default: consecutive)
```

**Effort:** Low-Medium — single stateful pass in a new `src/filter/dedup.rs`, new `dedup`
field on `FilterConfig`, independent of the rest of the pipeline.

---

## Gap 4 — Per-file error grouping (tsc full parity)

**What it is:** Extract a key from each line, group lines by that key, and render a
per-group header followed by the group's items. RTK builds `HashMap<filename, errors>` and
re-renders as a labelled section per file.

**Blocked filters:**
- `tsc` full output: `src/auth.ts (2 errors)\n  L12: TS2322 …\n  L15: TS2345 …`

**Why the current engine can't do it:**
The `parse.group` field groups by key and counts; it doesn't retain the _lines_ per group
or render them individually. Sections collect by state machine (enter/exit) not by per-line
key extraction.

**Options:**
1. Extend `parse.group` to retain matched lines and expose them as a per-group collection
2. Lua (Phase 4) — straightforward to implement with a `HashMap`

**Effort:** Medium (engine extension) / deferred to Phase 4 (Lua).

---

## Gap 5 — Per-item sub-filtering inside `each:` loops (pytest full parity)

**What it is:** When iterating over collected section items via `{items | each: "…"}`,
apply a line-level filter (e.g. keep only lines starting with `E` or `>`) to each item
_before_ rendering it.

**Blocked filters:**
- `pytest` failure context: RTK picks only the `>` (pointing) line and `E` (assertion)
  lines from each failure block; the rest of the traceback is discarded.

**Proposed TOML syntax inside `each:`:**
```toml
output = "{failure_blocks | each: \"{value | lines | keep: '^[>E] ' | join: '\\n'}\"}"
```

**Required engine additions (none of these pipes exist today):**

| Pipe | Signature | Description |
|------|-----------|-------------|
| `lines` | `Str → Collection` | Split a string on newlines into a collection of strings |
| `keep: <regex>` | `Collection → Collection` | Retain only items matching the regex |
| `where: <regex>` | `Collection → Collection` | Alias for `keep:` (filter-style naming) |

These must be added to `apply_pipe` in `src/filter/template.rs`. Implementation sketch:

```rust
// lines pipe: split string into collection
fn apply_lines(value: Value) -> Value {
    match value {
        Value::Str(s) => Value::Collection(s.lines().map(str::to_string).collect()),
        c => c,
    }
}

// keep pipe: filter collection items by regex
fn apply_keep(arg: &str, value: Value) -> Value {
    let pattern = parse_string_arg(arg);
    let Ok(re) = Regex::new(&pattern) else { return value };
    match value {
        Value::Collection(items) => Value::Collection(items.into_iter().filter(|l| re.is_match(l)).collect()),
        s => s,
    }
}
```

**Effort:** Medium — three new pipe arms in `src/filter/template.rs` (~30 lines), plus
tests. Resolves Gap 5 without Lua; Lua (Phase 4) is the alternative for the full general
case.

---

## Priority Recommendation

| Priority | Gap | Effort | Unlocks |
|---|---|---|---|
| 1 | Gap 1 (per-line replace) | Medium | pnpm/pip outdated, wc, docker ps columns |
| 2 | Gap 3 (dedup) | Low-Medium | docker/kubectl logs |
| 3 | Gap 5 (each: sub-filtering) | Medium | pytest full parity |
| 4 | Gap 2 + 4 together | High (via Lua) | all linters (eslint/ruff/pylint/golangci), go test, vitest, full tsc |
