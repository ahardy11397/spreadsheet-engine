# PLAN — spreadsheet-engine: a Memelang graduate-level demo

Build a minimal spreadsheet engine CLI entirely in Memelang. Beta
demonstration program: the hardest classic grad assignment — dependency
tracking, topological evaluation, and cycle detection.

Language docs: /mnt/ssd/projects/meme/docs/WRITING_GUIDE.md and
docs/TUTORIAL.md (read both first). Compiler commands via
`uv run --project /mnt/ssd/projects/meme meme ...` from this repo.

## Ground rules (non-negotiable)

- Work ONLY in this repo. NO git commits.
- 100% Memelang. Cell VALUES are integers only (no floats — keeps parity
  trivial); displayed values are integers.
- JSON persistence in `data/` via Optional record readers/writers; missing/
  corrupt file = clean Optional handling, never a traceback.
- Everything must parse with `--provider heuristic`.
- Test suites run in the FOREGROUND only.

## Domain and CLI (thin argv-router main + logic modules; unknown command →
usage + exit 1; every error = one stderr line + exit 1)

    sheet set <cell> <formula>     # cell like A1; formula is one of:
                                   #   <int>                    literal
                                   #   <cellref>                e.g. B2
                                   #   <cellref> +|-|* <cellref|int>
                                   #   SUM(A1:A5)  AVG(A1:A5)  MIN/MAX(A1:A5)
    sheet get <cell>               # evaluated value
    sheet display                  # grid dump, populated cells, refs shown
    sheet deps <cell>              # direct dependencies, sorted
    sheet dependents <cell>        # reverse deps, sorted
    sheet eval-order               # topological order of all populated cells

Engine rules:
- Formula changes RE-EVALUATE all dependents transitively (pull-based
  evaluation on demand is fine; caches must be invalidated correctly).
- Cycles: setting A1 = B2 + 1 then B2 = A1 + 1 must be detected and
  REJECTED at set time with `Error: cycle detected at <cell>` (exit 1) —
  the sheet must be left unchanged by a rejected set.
- Self-reference is a cycle. Range functions over ranges containing
  unevaluated/empty cells treat empty as 0. Unknown cellref in a formula
  evaluates as 0 (sparse sheet).
- Cell names: column letter(s) A..Z, row 1..999. Parse them with string
  operations.

## Tests — tests/meme/*.meme, `meme test`

At minimum: literal; direct ref; arithmetic chains 3 deep; SUM/AVG/MIN/MAX
over ranges incl. empty cells; transitive re-evaluation (change A1, assert
C1 that depends via B1 updates); cycle rejection (2-cycle, self-cycle);
rejected set leaves sheet unchanged (assert values after); eval-order is a
valid topological order (assert every dep appears before its dependent);
display grid; corrupt data tolerance. Fixed data, unique scratch files,
cleaned up.

## Acceptance ("done means")

1. `meme test` python — 0 failed.
2. Same `--target rust cpp go c ts` — ALL SIX green.
3. `meme build main.meme -o sheet_bin` — end-to-end demo: build a small
   budget sheet with SUM, change an input, show re-evaluated display, show
   a cycle rejection. Transcript in your final summary.
4. README.md: CLI usage, formula grammar, engine rules, tests.
5. Final message: files, test count per target, demo transcript,
   deviations with reasons.
