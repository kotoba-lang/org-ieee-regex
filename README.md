# kotoba-lang/org-ieee-regex — POSIX regular expressions as a Kotoba library

`regex.core`: POSIX ERE and BRE (IEEE Std 1003.1) compiled to a program and
matched by Thompson's NFA simulation — a library module for `grep`, `sed`
and `awk`, linked with `--source-path`.

```clojure
(ns grep.core (:require [regex.core :as rx]))
(rx/compile-pattern "^(foo|bar)+" true)   ; ERE → program string, or "!" + message
(rx/matches? prog text lo hi)             ; is there a match in the line [lo, hi)?
(rx/find prog text lo from hi)            ; leftmost-longest: start*2^32 + end, or -1
```

## Measured, 2026-09-17

Over 1,268,018 agent Bash calls: `grep -E` 9,582 patterns, shaped as
alternations of words (`W|W`, `^(W|W)`, `^W |W`), then `+` 1,294, `.` 1,225,
`*` 497, `?` 150, `{n,m}` 98, `[:space:]` 43; `sed -n '/re/p'` and `s/re/…/`
are the same grammar. That is the subset here: `|` `( )` `.` `*` `+` `?`
`{n}` `{n,}` `{n,m}`, bracket expressions with ranges, negation and the
POSIX classes, `^` `$`, `\` escapes (a literal metacharacter; `\w \W \d \D
\s \S` as classes; `\b \B` as boundaries), and the BRE spellings `\( \) \{
\} \| \+ \?`. No back-references — not regular, and none measured.

## Measured against the system utility

`test/regex_test.cljk` compiles a driver that links the module, packages
it, and compares its first match on each (pattern, line) against
`/usr/bin/grep -ob` — POSIX leftmost-longest with byte offsets — for 70
cases (each construct and its edges, `a|ab` → `ab`, `(a|ab)(c|bcd)` →
`abcd`, multi-byte lines and classes, `(a*)*b`) plus 7 patterns both sides
refuse. All byte-identical.

## How it runs, and what it cost to learn

- **A program is a string** of 9-byte records — an op byte and two four-digit
  base-93 fields (printable ASCII without `;`) — with *relative* jump
  targets computed from sub-program lengths, so it is built by concatenation
  alone. No vectors: the compiler's linearity rule refuses a vector that is
  read and written on one path, and a thread set is exactly that.
- **A thread set is a string** of `pc;` tokens, deduplicated by one
  `string-index-of`, rebuilt per character. Linear in input × program;
  `(a*)*b` cannot hang. Every visited pc is recorded, the ε ones too — the
  first cut recorded consumers only and followed the `(a*)*` cycle until the
  stack overflowed (exit 123).
- **`matches?` is one pass** (a fresh thread at pc 0 at every position);
  `find` is leftmost-longest the plain way, an anchored run from each start,
  each in its own `arena-scope` — unscoped, a 60-character line of misses
  minted 18,000 handles against a 4,096 budget.
- **Speed**: the per-character constant is host calls per live thread (~10),
  so a 5.4 MB file simulates in seconds where `/usr/bin/grep` takes 0.1 s.
  `grep` keeps a literal fast path for patterns without metacharacters and a
  *prefilter* from the top-level alternatives' literal prefixes, which makes
  the measured shapes run at BSD speed (`SIGILL|static_assert`: 0.15 s vs
  0.13 s user). Patterns that defeat the prefilter (`e+ in`, `^line [0-9]+`)
  run the simulation on every line: 2–4 s on 5.4 MB. The next lever is a
  lazily built DFA over the same program — read-only tables once built are
  what the linearity rule allows.

## What this is not

No back-references, no collating symbols (`[[.a.]]`) or equivalence classes
(`[[=a=]]`), no lazy quantifiers (`*?`, a TRE extension `/usr/bin/grep`
accepts), no `\<` `\>`. Case folding is the caller's (ASCII, before
compiling). Lines are at most 2²¹ bytes and a program 8,649 records.
