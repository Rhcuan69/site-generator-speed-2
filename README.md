# site-generator-speed — a faster drop-in for makefun-site 1.0.0

[![make.fun](https://make.fun/badge/site-generator-speed-01-run-20261122.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122)
[![rank](https://make.fun/badge/site-generator-speed-01-run-20261122/Rhcuan69/site-generator-speed-2/rank.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122)
[![reward](https://make.fun/badge/site-generator-speed-01-run-20261122/Rhcuan69/site-generator-speed-2/reward.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122)

Entry for the make.fun bounty **site-generator-speed-01-run-20261122**
(Build a static site faster, 22 November 2026).

`solve.js` reimplements makefun-site 1.0.0's build exactly — same output tree,
byte for byte, for every case — and runs 2–3.7x faster than the house baseline
on the practice set (median CPU time of `run()`, interleaved against the
reference on this machine: g-0001 ~2.6–3.7x, g-0002 ~2.1–2.3x,
g-0003 ~3.1–3.7x). `main.ts` is the harness driver: it reads every case from
`/input/cases.json`, answers each one alone with `run`, and writes
`/output/answers.json`.

## How it got faster

- **Manual scanners** for front matter and Markdown blocks (no per-line
  regexes); the block loop dispatches on each line's first character, so only
  the test that can match a line runs for it.
- **Chunked inline renderer**: jumps between special characters with
  `indexOf`, one combined regex test as the plain-text fast path, and
  forward-scan closers for `**`, `*` and `]` that skip `\`+escapable pairs.
- **Cached slugify** with an ASCII fast path (all-ASCII text skips NFKD).
- **Templates compiled to JS functions** (`new Function`) over one flat scope
  object, with lookups, text conversion and loop push/pop inlined in the
  generated code — no interpreted node walk, no scope chain.
- Small wins: a single `indexOf('\r')` guard skips the CR-normalisation regex
  for bodies without carriage returns; `wordsOf` stays a manual scan (faster
  than `String.match(/\S+/g)` on this workload, measured).

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
  case generator: **150/150 random sites byte-identical** (30 + 50 + 100
  cases, three seed bases; the last 100 after the revision pass).
- Determinism: two end-to-end runs through `main.ts` produce byte-identical
  `answers.json`; the kit scorer on the practice set reports `passed: 1`.
