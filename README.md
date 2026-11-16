# Fast marked — a faster drop-in for marked 18.1.3

[![make.fun · markdown-render-speed-02](https://make.fun/badge/markdown-render-speed-02.svg)](https://make.fun/bounty/markdown-render-speed-02?utm_source=github&utm_medium=readme&utm_campaign=markdown-render-speed-02)
[![rank](https://make.fun/badge/markdown-render-speed-02/Sapriyudin/markdown-render-speed-02/rank.svg)](https://make.fun/bounty/markdown-render-speed-02/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=markdown-render-speed-02)
[![reward](https://make.fun/badge/markdown-render-speed-02/Sapriyudin/markdown-render-speed-02/reward.svg)](https://make.fun/bounty/markdown-render-speed-02?utm_source=github&utm_medium=readme&utm_campaign=markdown-render-speed-02)
[![Node](https://img.shields.io/badge/node-22-green)](https://nodejs.org/)

A faster drop-in replacement for [marked](https://github.com/markedjs/marked)
18.1.3: `parse(markdown)` returns byte-for-byte the HTML that marked renders
with its default options (GFM on, breaks off, pedantic off, async off), for
every input — and renders it faster.

Built for the make.fun bounty
[Make Markdown rendering faster](https://make.fun/bounty/markdown-render-speed-02)
($250, top 20).

## Results (practice set, local)

- **720/720 cases byte-identical** to marked 18.1.3 (`grade-local`:
  `{"cases_passed":720,"cases_total":720,"passed":1}`).
- **~1.5x faster than marked** on the 40 generated README documents
  (local, Node 24, interleaved runs; ~1.3x on the 680 spec examples).

## How it works

This is a fork of marked 18.1.3 (MIT, see `LICENSE` and the notice in
`parse.js`), optimized without changing its semantics, so byte-exactness
holds by construction:

- `parse()` bypasses marked's `Marked` wrapper and lexes + parses directly
  with the shared defaults object — no per-call option spreads or hooks
  plumbing.
- The inline loop only tries a tokenizer when the first character can start
  it (`\` for escapes, `<` for tags/autolinks, `[`/`!` for links, `*`/`_`
  for emphasis, `` ` `` for code spans, `~` for strikethrough, …).
- The block loop does the same for each block tokenizer, and only attempts
  the expensive setext-heading regex when a `=`/`-` underline line actually
  exists before the next blank line.
- The list tokenizer guards its per-line continuation regexes by the first
  non-space character, skips tab-expansion work when a line has no tab, and
  replaces `split`/`search` with `indexOf`/manual scans.
- The emphasis/delimiter closer search (`fe`) walks backwards from the end
  with native `lastIndexOf` instead of scanning the whole inline source
  character by character (only the last qualifying run per delimiter
  matters), and delimiter-run lengths avoid spread allocations.
- `lex()` skips the carriage-return normalization when the input has no
  `\r`, and the emphasis-masking replaces are skipped when their trigger
  characters are absent.

Nothing is cached between calls: `parse()` computes its output from the
markdown it is given, keeps no state from an earlier call, and makes no
network access.

## Layout

- `parse.js` — the parser (dependency-free ES module, exports `parse`).
- `main.ts` — the driver: reads `/input/cases.json`, renders each case
  alone with `parse()`, writes `/output/html.json`.
- `package.json` — `{"type": "module"}` so Node loads the entry as ESM
  without re-parsing.

## License

MIT. marked 18.1.3 is Copyright (c) 2018–2026 MarkedJS (MIT); this fork
keeps its license and notice.
