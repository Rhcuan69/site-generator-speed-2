# site-generator-speed — a faster drop-in for makefun-site 1.0.0

[![make.fun](https://make.fun/badge/site-generator-speed-01-run-20261122.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122)
[![rank](https://make.fun/badge/site-generator-speed-01-run-20261122/Rhcuan69/site-generator-speed-2/rank.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122)
[![reward](https://make.fun/badge/site-generator-speed-01-run-20261122/Rhcuan69/site-generator-speed-2/reward.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122)

Entry for the make.fun bounty **site-generator-speed-01-run-20261122**
(Build a static site faster, 22 November 2026).

`solve.js` reimplements makefun-site 1.0.0's build exactly — same output tree,
byte for byte, for every case — and runs 3.1–4.7x faster than the house baseline
on the practice set (median CPU time of `run()`, interleaved against the
reference on this machine: g-0001 ~3.7–4.2x, g-0002 ~3.1–3.8x,
g-0003 ~3.9–4.7x). It is also ~1.2–1.4x faster than the previous entry
(site-generator-speed-2, itself 2.1–3.7x over baseline). `main.ts` is the
harness driver: it reads every case from `/input/cases.json`, answers each
one alone with `run`, and writes `/output/answers.json`.

## How it got faster

- **Manual scanners** for front matter and Markdown blocks (no per-line
  regexes); the block loop dispatches on each line's first character, so only
  the test that can match a line runs for it.
- **Single-scan inline renderer**: one native regex scan finds the next
  special character, and HTML escaping is done inline as the scan goes — no
  separate escape pass, no per-chunk `indexOf` fan-out, output built with
  `+=`. Forward-scan closers for `**`, `*` and `]` skip `\`+escapable pairs.
- **Regex slugify** (same mapping as the per-character scan, verified):
  lowercase, collapse non-alphanumeric runs to one `-`, trim, cap at 64.
- **Templates compiled to JS functions** (`new Function`) that build one
  string with `+=` and return it — no output array, no join — with the
  escape/text/truthy helpers as direct parameters. A `for` loop keeps no save
  stack and allocates no loop object per iteration: one shared mutable loop
  object per build, fields saved/restored at the loop's edges, and the loop
  variable saved/restored in a local.
- Small wins: `escapeHtml` is a regex test plus a native replace; a single
  `indexOf('\r')` guard skips the CR-normalisation regex for bodies without
  carriage returns; `wordsOf` stays a manual scan (faster than
  `String.match(/\S+/g)` on this workload, measured); Markdown blocks are
  pushed with their trailing newline already attached.

No dependencies, no network, no WASM or native code. Deterministic: two runs
produce byte-identical output. No state is kept between cases.

## Files

- `solve.js` — the entry: `export function run(input)`.
- `main.ts` — the harness driver (unchanged from the house baseline).
- `README.md` — this file.
- `LICENSE` — MIT.

## Verification (this build, 2026-11-23)

- Practice set: **3/3 cases byte-identical** to the kit's reference answers.
- Differential fuzzing against makefun-site's own `build`, using the kit's
  case generator: **150/150 random sites byte-identical** (50 + 100 cases,
  two seed bases; the last 100 after the final revision).
- Determinism: two end-to-end runs through `main.ts` produce byte-identical
  `answers.json`; the kit scorer on the practice set, run on node v22.23.4,
  reports `passed: 1`.
