# spreadsheet-engine

A minimal spreadsheet engine CLI written **100% in Memelang** — the classic
graduate assignment: dependency tracking, topological evaluation, and cycle
detection. Cell values are integers only; persistence is JSON in `data/`.

Part of the Memelang beta demonstration suite (see also `ledger`, `payroll`,
`library-system` in the compiler repo).

## CLI

    sheet set <cell> <formula>     # cell like A1; formula is one of:
                                   #   <int>                    literal
                                   #   <cellref>                e.g. B2
                                   #   <cellref> +|-|* <cellref|int>
                                   #   SUM(A1:A5)  AVG(A1:A5)  MIN/MAX(A1:A5)
    sheet get <cell>               # evaluated value
    sheet display                  # grid dump; rows/columns labeled by cell refs
    sheet deps <cell>              # direct dependencies, sorted
    sheet dependents <cell>        # reverse deps, sorted
    sheet eval-order               # topological order of all populated cells

Unknown commands and missing args print the usage text to stderr and exit 1;
a command given wrong arity prints a one-line usage hint to stderr and exits 1.
Every error is exactly one stderr line plus exit code 1. Success output
goes to stdout with exit code 0.

### Formula grammar

```
formula   := int | cellref | operand ("+" | "-" | "*") operand
           | FN "(" cellref ":" cellref ")"
FN        := "SUM" | "AVG" | "MIN" | "MAX"
operand   := cellref | int
cellref   := uppercase column letters (1..2, A..ZZ) + row digits (1..999)
int       := optional "-" followed by digits
```

Whitespace is allowed around tokens (`B2*C3` and `B2 * C3` are the same
formula). Lowercase references are rejected. The binary-operand form is a
documented superset of the plan grammar: an integer is accepted on either
side (`3 - A1` is valid), not only on the right. Shell users must quote
formulas containing spaces or operators:

- `sheet set A1 42`
- `sheet set B1 "A1 + 3"`
- `sheet set C1 "B1 * 2"`
- `sheet set D1 "SUM(A1:A5)"`

## Engine rules

- **Pull-based evaluation with correct invalidation.** Every command loads
  the sheet from disk and evaluates on demand, so a formula change is
  always reflected in every transitive dependent — nothing is ever stale.
- **Cycle detection at set time.** Setting `A1 = B2 + 1` then
  `B2 = A1 + 1` is rejected on the second `set` with
  `Error: cycle detected at B2` (exit 1). The check runs BEFORE any state
  is applied or saved, so a rejected set leaves the sheet byte-identical.
  Self-reference (`A1 = A1 + 1`) is a cycle. Indirect cycles through
  arbitrary chains are detected by reachability over the current
  dependency edges.
- **Sparse sheet.** A reference to an unpopulated (or unknown) cell
  evaluates as 0. Range functions treat empty cells as 0 (`AVG` counts
  them in the denominator; `MIN`/`MAX` see 0).
- **Runtime cycle guard.** A hand-corrupted data file can contain a cycle
  that never passed set-time rejection; evaluation visits each cell once
  per query and unwinds deterministically instead of hanging.
- **Cell names.** Column letter(s) A..ZZ (1–2 uppercase letters) and row
  1..999, parsed with plain string operations.
- **Deterministic ordering.** `deps`, `dependents`, and `eval-order` are
  sorted in spreadsheet order (A..Z, then AA, AB, ...). `eval-order` is a
  Kahn-style topological emission with spreadsheet-order tie-breaking; a
  dependency on an unpopulated cell counts as satisfied, and corrupt
  records with unparseable names are skipped.
- **Persistence.** The sheet is a JSON array of `{name, formula}` records
  at `data/sheet.json`. Missing or corrupt files load as an empty sheet —
  never a traceback (Optional record reader with `or else nothing`).
- **Integer-only values.** All cell values are 64-bit Ints; `AVG` uses
  truncating integer division identical across every backend.

## Layout

    main.meme      thin argv router (usage text, command dispatch, error lines)
    cells.meme     cell-name math: parsing, column <-> letters, sort keys, ranges
    formula.meme   formula parser + dependency extraction
    sheet.meme     engine: persistence, evaluation, cycle checks, topo order, display
    tests/meme/    test suites run by `meme test` (module copies included)

Errors are written to stderr with `append ... to file "/dev/stderr"` (the
language has no stderr print; verified to produce real stderr bytes on all
six backends).

## Tests

Six suites under `tests/meme/`, run in the foreground:

    uv run --project /mnt/ssd/projects/meme meme test tests/meme --provider heuristic --target python

Status: **42 cases, 0 failed** on each of all six backends
(`--target python rust cpp go c ts`).

Coverage: literals; direct refs; 3-deep arithmetic chains; mixed binops;
SUM/AVG/MIN/MAX over sparse ranges (including negative-total AVG
truncation-toward-zero parity); transitive re-evaluation (change A1,
assert C1 via B1); two-cycle, self-cycle, and indirect cycle rejection;
rejected set leaves the sheet unchanged (reload + re-assert); a valid set
still applies after a rejection; eval-order is a valid topological order;
corrupt/unparseable formula tolerance; corrupt cyclic data terminates;
JSON roundtrips; missing/corrupt/wrong-shape file tolerance; display grid;
corrupt-name tolerance in ordering and eval-order.

## Building and demo

    uv run --project /mnt/ssd/projects/meme meme build main.meme --provider heuristic -o sheet_bin
    ./sheet_bin set B2 100 && ./sheet_bin set B3 50 && ./sheet_bin set B4 25
    ./sheet_bin set B5 "SUM(B2:B4)" && ./sheet_bin set B6 300 && ./sheet_bin set B7 "B6 - B5"
    ./sheet_bin display            # B5=175, B7=125
    ./sheet_bin set B2 80          # change an input
    ./sheet_bin display            # B5=155, B7=145 (transitively re-evaluated)
    ./sheet_bin set B2 "B5 + 1"    # Error: cycle detected at B2 (exit 1)

## Notes

- Everything parses with the offline `--provider heuristic` provider only.
- The Rust backend does not clone `Set`-valued function arguments (Vec
  arguments are cloned), so the evaluation path guard uses a
  `List<String>` instead of a `Set<String>`.
