# Build log — site-generator-speed-01-run-20261122 (v3 rebuild)

Fresh rebuild in this session (started 2026-11-23, after the bounty opened
2026-11-22 20:00 UTC), in a NEW directory. The previous entry
(`~/workspace/makefun/site-generator-speed-v2-gh/`, submission 01a206ff,
already submitted and being graded) was read for understanding, but `solve.js`
below was written fresh in this session, in three substantive passes
(initial write of the full file, then a revision pass on the inline
renderer, then the slugify revision) plus two small edits. The other files
(main.ts, README.md, LICENSE, .makefun/owner.json) were copied from the v2
directory unchanged except README.md, which was updated for this version.

## Files produced (~/workspace/makefun/site-generator-speed-v3/)

- `solve.js` — `export function run(input)`, dependency-free ES module.
- `main.ts` — harness driver, unchanged from the house baseline.
- `README.md` — entry description.
- `LICENSE` — MIT (copyright line matches the `makefun init`-generated
  LICENSE of the previous staging repo; `init` commits a LICENSE of its own,
  so drop this file first if `init` refuses a repo that already holds one).
- `BUILD_LOG.md` — this file.

Repo total is ~46 KB, far under the 4 MiB cap. Test scaffolding lives in
`/tmp/sgs-v3/` (bench/fuzz/practice/micro scripts) and must NOT go into the
repo.

## Practice correctness

- Kit scorer on the practice set, end to end through `main.ts` on node
  v22.23.4 (the grader's version): **passed: 1, cases_passed: 3,
  cases_total: 3**.
- Direct comparison vs `practice/reference/expected.json`: **3/3 cases
  byte-identical**.
- Determinism: two end-to-end runs through `main.ts` produce byte-identical
  `answers.json`.
- Differential fuzzing against makefun-site's own `build`, using the kit's
  case generator (`program/generator/subject/sites.mjs`): **150/150 random
  sites byte-identical** (50 + 100 cases, two seed bases). Failing cases: none.

## Timing vs baseline (makefun-site 1.0.0) and vs v2

CPU time of `run()` only, interleaved v2-vs-v3-vs-reference per repetition
(median, `process.cpuUsage`), this VM. Five bench rounds of 21 repetitions
each, after the final revision:

| case | v3 | v2 | ref | v3/v2 | v3/ref |
|------|----|----|-----|-------|--------|
| g-0001 | ~39–44 ms | ~45–55 ms | ~158–174 ms | ~1.14–1.30x | ~3.7–4.2x |
| g-0002 | ~65–77 ms | ~95–105 ms | ~233–245 ms | ~1.32–1.51x | ~3.1–3.8x |
| g-0003 | ~64–73 ms | ~83–91 ms | ~282–308 ms | ~1.15–1.30x | ~3.9–4.7x |

Per-case medians across the five rounds: v3/v2 ≈ 1.21x / 1.38x / 1.29x
(overall ≈ 1.3x faster than v2); v3/ref ≈ 3.9x overall (v2 was 2.1–3.7x).
The machine is throttled, so absolute times move run to run, but v3 beat v2
on every case in every one of the 15 round-measurements (range 1.14–1.51x;
one 0.90x reading in an early round, before the inline `+=` revision,
was noise — the revision was re-measured in five clean rounds after it).

## How it got faster (revisions vs v2)

Profiled v2 first (inline ~20%, template render ~22%, cardOf ~11%,
renderMarkdown ~12%, GC ~5%), then rewrote:

1. **Pass 1 — full rewrite.** Same pipeline and semantics, new code paths:
   - `inline` is one native regex scan (`/[\\`*[\]&<>"']/g`, shared regex
     with `lastIndex` set explicitly before every `exec`, so the recursion
     into emphasis/link text cannot disturb the caller's scan) that also
     escapes HTML as it goes — no separate escape pass, no per-chunk
     `indexOf` fan-out. Microbenchmark on 10,562 real paragraphs: 55 ms
     (v2 chunked) → 36 ms. Verified 0 diffs vs v2's inline on that corpus.
   - `escapeHtml` is a regex test plus a native `replace` (measured ~2x
     faster than the manual loop on real lines).
   - Templates compile to functions that build one string with `+=` and
     return it (microbenchmark: ~10x faster than array push + join for a
     page-shaped render), helpers as direct parameters, no output array.
   - `for` loops keep no save stack and allocate no loop object per
     iteration: one shared mutable loop object per build, its four fields
     saved/restored at the loop's edges, the loop variable saved/restored
     in a local. Restoring a shadowed name by assigning its old value
     (undefined when unbound) reads exactly as unbound in every lookup.
   - Markdown blocks are pushed with their trailing newline attached
     (no second join pass); `page.toc` passes the heading list through
     instead of copying it.
2. **Pass 2 — inline output via `+=` instead of array + join.** Microbenchmark
   on the same paragraph corpus: 37 ms → 28 ms (~24% faster).
3. **Pass 3 — regex slugify.** Same mapping as the per-character scan
   (lowercase, collapse non-alphanumeric runs to one `-`, trim, cap 64),
   verified 0 diffs on 1,949 real titles/headings/tags plus adversarial
   Unicode cases, and 150/150 fuzz after the change. Microbenchmark:
   3.35 ms → 2.40 ms per 1,949 texts.

## Ideas measured and rejected

- Hybrid inline fast path (escape-only lines via one replace): slower overall
  (25.9 ms vs 23.6 ms) — only ~7% of paragraphs are escape-only, and the
  extra regex test per call costs more than it saves.
- `wordsOf` variants (regex exec, `match`, ASCII fast-path branches): the
  manual table-driven scan stays fastest (~17 ms over all practice bodies);
  a stub A/B was too noisy to trust, the microbenchmark stands.
- Counting words inside `renderMarkdown` instead of a separate pass: same
  total character work, no saving.

## Judgment calls / deviations from the reference build

- The code is a fresh write in this session, not a copy of v2's file; the
  architecture is the same proven design (it is the fast design for this
  spec) and the semantics were checked against the vendor code in the kit
  (`baseline/makefun-site/vendor/makefun-site/`), which is the authority
  where the README differs.
- `main.ts` is the house baseline driver, copied unchanged — it is the
  harness driver, not entry code.
- Templates use `new Function` for compilation at runtime — plain
  dependency-free JavaScript, no install, no network, no WASM.
- No state is kept between cases; output is deterministic.
- Only ES2020–ES2022 features are used, so the code runs on node 22/26
  alike (this machine runs node v24.22.1; the grader runs node 22.23.4;
  the scorer run above used the kit's node v22.23.4 binary).

## Notes for the parent (repo + bundle + submit)

- The entry is code only: `solve.js`, `main.ts`, `README.md`, plus a MIT
  LICENSE and `.makefun/owner.json` with wallet
  `2tY3mKb5dRYq77XkpbobPFvVhMqBwdPyCRyzo3mVNzLh` (per the watch rules).
  Do NOT add anything from `/tmp/sgs-v3/` (bench/fuzz scripts) or from
  `~/workspace/makefun/sgs-test/` (it holds the 2.7 MB practice
  `cases.json`).
- Create the repo FRESH (after opening) — do not reuse the old repo/dir.
  Suggested name following the v2 pattern: `site-generator-speed-3`.
- Entries close 2026-11-24 20:00 UTC / 03:00 WIB 25 Nov. The live entry
  (submission 01a206ff, claim AF-40A0X) is already being graded; only
  resubmit if this ~1.3x margin over v2 is worth it — it is a real, repeated
  margin, not noise (15/15 round-measurements in v3's favour after the
  final revision).
