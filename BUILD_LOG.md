# Build log — site-generator-speed-01-run-20261122 (v2 rebuild)

Fresh rebuild in this session (started after the bounty opened 2026-11-22
20:00 UTC), in a NEW directory. The previous build
(`~/workspace/makefun/site-generator-speed/`) is void for submission because
its session and repo predate opening; nothing was copied from it — `solve.js`
below was written fresh in this session, in two substantive passes
(initial write, then a revision/optimization pass on the Markdown renderer
and inline helpers).

## Files produced (~/workspace/makefun/site-generator-speed-v2/)

- `solve.js` — `export function run(input)`, dependency-free ES module.
- `main.ts` — harness driver, unchanged from the house baseline.
- `README.md` — entry description.
- `LICENSE` — MIT (copyright line matches the `makefun init`-generated
  LICENSE of the previous staging repo; `init` commits a LICENSE of its own,
  so drop this file first if `init` refuses a repo that already holds one).
- `BUILD_LOG.md` — this file.

Repo total is ~45 KB, far under the 4 MiB cap. Test scaffolding lives in
`/tmp/sgs-v2/` (bench/fuzz/practice scripts) and must NOT go into the repo.

## Practice correctness

- Kit scorer on the practice set, end to end through `main.ts`:
  **passed: 1, cases_passed: 3, cases_total: 3**.
- Direct comparison vs `practice/reference/expected.json`: **3/3 cases
  byte-identical**.
- Determinism: two end-to-end runs through `main.ts` produce byte-identical
  `answers.json`.
- Differential fuzzing against makefun-site's own `build`, using the kit's
  case generator (`program/generator/subject/sites.mjs`): **150/150 random
  sites byte-identical after the revision pass** (50 + 100 cases, two seed
  bases), plus 30/30 before it. Failing cases: none.

## Timing vs baseline (makefun-site 1.0.0)

CPU time of `run()` only, interleaved mine-vs-reference per repetition
(median, `process.cpuUsage`), this VM:

| case | mine | reference | speedup |
|------|------|-----------|---------|
| g-0001 | ~51–62 ms | ~144–203 ms | ~2.6–3.7x |
| g-0002 | ~97–119 ms | ~222–246 ms | ~2.1–2.3x |
| g-0003 | ~80–87 ms | ~262–303 ms | ~3.1–3.7x |

Ranges are across repeated bench runs (9 and 21 repetitions); the machine is
throttled, so absolute times move run to run, but the entry beats the
baseline on every case in every run. For comparison the void previous build
measured 2.84x / 2.22x / 3.04x on the same bench.

## How it got faster (revisions)

1. **Pass 1** — faithful reimplementation: manual scanners instead of
   per-line regexes, chunked inline renderer (indexOf jumps), cached
   slugify with an ASCII fast path, templates compiled to JS functions with
   a flat scope. Practice 3/3, fuzz 30/30, ~1.9–3.2x.
2. **Pass 2 (revision)** — profiled the build (Markdown ~45% of `run()`,
   template render ~18%, `wordsOf` ~10%, front matter and template
   compile ~2% each) and refactored the hot section:
   - Markdown block loop now dispatches on each line's first character, so
     only the test that can match a line runs for it; the list collector
     factored out into `collectList`.
   - `endsParagraph` dispatches on first character too (only a blank line,
     or one opening on `` ` ``, `#`, `-`, `*`, can end a paragraph).
   - `inline` fast path is a single combined regex test
     (`/[\\`*[\]&<>"']/`) instead of four `indexOf` calls plus
     `escapeHtml`'s own test — measured ~3x faster on plain lines.
   - `findUnescaped` / `findSingleStar` rewritten as forward scans that
     skip `\`+escapable pairs (the reference's own rule, read forward),
     replacing the backslash-counting backscan — same results, no
     quadratic rescan on backslash-heavy text.
   - CR normalisation: a single `indexOf('\r')` guard skips the regex for
     bodies without carriage returns.
   - `wordsOf` kept as a manual scan: micro-benchmarked faster than
     `String.match(/\S+/g)` (848 ms vs 3157 ms over 50 passes of the
     practice bodies) and identical in count.
   After pass 2: practice 3/3, fuzz 150/150, ~2.1–3.7x.

## Judgment calls / deviations from the reference build

- The code is a fresh write, not a copy of the previous build's file; the
  architecture is the same (it is the proven fast design for this spec) and
  the semantics were checked against the vendor code in the kit
  (`baseline/makefun-site/vendor/makefun-site/`), which is the authority
  where the README differs.
- `main.ts` is the house baseline driver, copied unchanged — it is the
  harness driver, not entry code.
- Templates use `new Function` for compilation at runtime — plain
  dependency-free JavaScript, no install, no network, no WASM.
- No state is kept between cases; output is deterministic.
- Only ES2020–ES2022 features are used, so the code runs on node 22/26
  alike (this machine runs node v24.22.1; the grader runs node 22.23.4).

## Notes for the parent (repo + bundle + submit)

- The entry is code only: `solve.js`, `main.ts`, `README.md`, plus a MIT
  LICENSE and `.makefun/owner.json` with wallet
  `2tY3mKb5dRYq77XkpbobPFvVhMqBwdPyCRyzo3mVNzLh` (per the watch rules).
  Do NOT add anything from `/tmp/sgs-v2/` (bench/fuzz scripts) or from
  `~/workspace/makefun/sgs-test/` (it holds the 2.7 MB practice
  `cases.json`).
- Create the repo FRESH (after opening) — do not reuse the old repo/dir.
- Entries close 2026-11-24 20:00 UTC / 03:00 WIB 25 Nov.
