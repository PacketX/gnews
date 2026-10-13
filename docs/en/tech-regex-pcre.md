# The regex filter: PCRE syntax support on the two product lines

## Two different engines

`<find name="regex">` is two entirely different implementations on the two product lines, and the MIPS side comes in two compiler generations; one rule is not guaranteed the same result on all of them:

| | arm64 (DPDK) models, 7.6.x | MIPS models, 6.6.x |
|---|---|---|
| Engine | PCRE2 10.42 (software) | Cavium HFA (hardware-assisted) |
| How rules reach it | each rule is compiled on its own | every regex of the configuration, one per line, is compiled into **one** graph |
| Compiler | built into the engine | **Gen4** on CN73/78 (T12S, the T20 family); **Gen3** on the rest (G8S, G8, G4…) |
| Defaults | case-sensitive; `.` stops at a newline; `^` `$` are the ends of the payload | case-sensitive; `.` stops at a newline; `^` `$` do not mean what they mean in PCRE, nor the same on the two generations (below) |

All three match **bytes** (there is no UTF-8 mode), search the L7 payload by default, and search each packet on its own, never across packets.

```xml
<filter id="10" name="sql">
  <or>
    <find name="regex" relation="==" content="(?i)union\s+select" />
  </or>
</filter>
```

## How it was tested

137 rules in 14 groups (literals, escapes, anchors, quantifiers, character classes, POSIX classes, shorthands, back-references, groups, assertions, inline options, Cavium extensions, rules seen in real use…), each with one to five payloads, some that should match and some that should not. Every (rule, payload) pair was replayed through a loopback port of the device and the device's verdict read from the filter counter, then compared with what **the PCRE2 10.42 library itself** answers for the same pair.

| | Accepted by the compiler | Comparable with PCRE2 | Same result | Different result |
|---|---|---|---|---|
| arm64 (HL1) | 135 | 135 | 135 | 0 |
| MIPS Gen4 (T12S) | 126 | 123 | 107 | 16 |
| MIPS Gen3 (G8) | 125 | 123 | 98 | 25 |

Two of the 137 are not PCRE at all (`\x{4142}` and `**`); PCRE2 itself refuses them. Six of each "different" count are Cavium macros, which mean something else in PCRE to begin with; three more of Gen3's 25 are right when used alone and were swayed by other rules of the same batch (see "Gen3: similar rules affect each other").

## Syntax support

✔ = accepted, with PCRE2's result.

| Syntax | arm64 | MIPS Gen4 | MIPS Gen3 |
|---|---|---|---|
| literals, escapes such as `\.` | ✔ | ✔ | ✔ |
| `\xhh` `\x{hh}` `\ddd` (octal) `\t` `\r` `\n` `\a` `\e` `\f` `\cA` `\Q…\E`; NUL and bytes from 0x80 up | ✔ | ✔ | ✔ |
| `.`, `a\|b`, groups `( )`, non-capturing `(?:…)` | ✔ | ✔ | ✔ |
| `*` `+` `?` `{n}` `{m,n}` `{m,}` and the lazy forms `*?` `+?` `??` `{m,n}?` | ✔ | ✔ | ✔ ※ |
| classes `[abc]` `[^abc]` `[a-f0-9]` `[\x00-\x1f]` `[\d\s]` | ✔ | ✔ | ✔ ※ |
| POSIX classes `[[:alnum:]]` … `[[:xdigit:]]` (all 14), negated `[[:^digit:]]` | ✔ | ✔ | ✔ ※ |
| `\d` `\D` `\w` `\W` `\S` | ✔ | ✔ | ✔ |
| `\h` `\H` | ✔ | ✔ | **differs**: `\h` includes the newline, `\H` excludes it |
| back-references `\1` `\g1` `\g{1}` `\g{-1}`; named groups `(?<n>…)` `(?'n'…)` with `\k<n>` `\k'n'` `\k{n}` | ✔ | ✔ | ✔ |
| `\b` `\B` | ✔ | ✔ | ✔ (except `\b` next to a quantified capturing group, below) |
| `\A` `\z` `\Z` | ✔ | ✔ | **differs**: accepted and ignored, as if not written |
| lookaround `(?=…)` `(?!…)` `(?<=…)` `(?<!…)` | ✔ | ✔ | ✔ |
| inline options `(?i)` `(?i:…)` `(?-i)` `(?s)` `(?x)`, comments `(?#…)` | ✔ | ✔ | ✔ |
| `^` `$` | the ends of the payload; every newline too only with `(?m)` | **differs**: the ends of the payload **and** every newline, as if `(?m)` were always on | **differs**: newlines **only** -- `^` is after a newline, `$` before one; the start and the end of the payload do not count, `(?m)` or not |
| possessive quantifiers `*+` `++` `?+` `{m,n}+` | ✔ | **differs**: accepted, but they behave like the ordinary (greedy) ones | as Gen4 |
| `\v` `\s` `\V` | ✔ | **differs**: `\v` is the one character VT (0x0B), so `\s` lacks VT and `\V` matches a newline | as Gen4 |
| `\N` `\o{…}` | ✔ | **differs**: accepted, and never match | as Gen4 |
| `\G` | ✔ | **differs**: accepted, and never matches | **differs**: accepted and ignored, as if not written |
| `\K` | ✔ | accepted and ignored (whether the rule matches is unaffected) | as Gen4 |
| Python-style named groups `(?P<n>…)` `(?P=n)` | ✔ | ✘ | ✘ |
| atomic groups `(?>…)`, branch reset `(?\|…)`, conditionals `(?(1)…)`, recursion and subroutine calls `(?1)` | ✔ | ✘ | ✘ |
| `(?U)`, `(*…)` start-of-pattern items, callouts `(?C)` | ✔ | ✘ | ✘ |
| `\R`, Unicode properties `\p{…}` | ✔ | ✘ | ✘ |
| Cavium macros `{i}` `{s}` `{x}` `{t}` `{@n}` (below) | read as literal text | ✔ | ✔ |
| multi-byte hex `\x{4142}` | ✘ | ✔ | ✔ |
| `**` | ✘ | ✔ | ✘ |

※ Right when used alone; may answer wrongly when loaded beside a similar rule, see "Gen3: similar rules affect each other".

The HFA syntax specification describes what Gen3 does; Gen4 measured differently in three places: the specification says `^` is replaced by `\n`, yet the start of the payload matches too; it says `\h` includes the newline, yet it does not; and `(?:…)`, which it does not list, works.

## Accepted, with a different answer

These are the easy ones to get wrong, because nobody complains -- the answers just differ:

| Rule | Payload | arm64 | Gen4 | Gen3 |
|---|---|---|---|---|
| `^abc` | `abcdef` | match | match | **no match** |
| `^abc` | `zz⏎abcdef` | no match | **match** | **match** |
| `abc$` | `zzabc` | match | match | **no match** |
| `abc$` | `abc⏎zz` | no match | **match** | **match** |
| `\Aabc` | `zzabcdef` | no match | no match | **match** |
| `abc\z` | `abczz` | no match | no match | **match** |
| `\Gabc` | `abcx` | match | **no match** | match |
| `\Gabc` | `xabc` | no match | no match | **match** |
| `xa++ay` | `xaaay` | no match | **match** | **match** |
| `x\sy` | `x`, VT, `y` | match | **no match** | **no match** |
| `x\vy` | `x⏎y` | match | **no match** | **no match** |
| `x\Vy` | `x⏎y` | no match | **match** | **match** |
| `x\hy` | `x⏎y` | no match | no match | **match** |
| `a\Nb` | `axb` | match | **no match** | **no match** |
| `x\o{101}y` | `xAy` | match | **no match** | **no match** |
| `\b(\d{1,3}\.){3}\d{1,3}\b` | `src=10.20.30.40;` | match | match | **no match** |

(⏎ is the newline, 0x0A; bold marks an answer that is not PCRE2's.)

The last row is Gen3's trouble with `\b` next to a quantified capturing group: drop the `\b`, or make the group non-capturing -- `(?:\d{1,3}\.){3}` -- and it answers as the other two do.

## Gen3: similar rules affect each other

On the Gen3 models (G8S, G8, G4), two rules that begin and end alike and differ only in the repeated middle can, **when loaded together**, have one report a match on behalf of the other. Either rule loaded alone is right; arm64 and Gen4 were given the same combinations and do not show it.

| The two rules, loaded together | Payload | Should match | Reported as matching on Gen3 |
|---|---|---|---|
| `x[a-z]+y`, `x[0-9]+y` | `xabcy` | the first only | both |
| `a.*c`, `a.+c` | `ac` | the first only | both |
| `a.*?c`, `a.+?c` | `ac` | the first only | both |
| `x[a-z]{1,200}y`, `x[a-f0-9]+y` | `xdeagy` | the first only | both |

Combinations that did not show it in the same run: fixed counts (`x[a-z]{3}y` with `x[a-f]{3}y`), literals with a common prefix (`hello1` with `hello2`), one class inside the other (`x[a-z]+y` with `x[a-f]+y`).

On a Gen3 model, where a configuration holds such a set -- same beginning and end, a `+` or `*` repeat between -- check each rule's result with real payloads, or fold the set into one rule.

## Cavium macros (MIPS only)

An HFA rule may begin with a `{…}` block, its attributes separated by commas, as in `{i,s}a.c`:

| Macro | Meaning | In PCRE |
|---|---|---|
| `{i}` | case-insensitive | `(?i)` (taken by all three) |
| `{s}` | `.` matches a newline too | `(?s)` (taken by all three) |
| `{x}` | white space in the rule is ignored | `(?x)` (taken by all three) |
| `{@n}` | match only n bytes from the start; `{@0}` is the start | `\A` for the start (not honoured by Gen3) |
| `{t}` | report where the match begins (does not change whether it matches) | — |

On arm64 the block is **literal text, not a macro**: `{i}hello` is a case-insensitive hello on MIPS and the eight characters `{i}hello` on arm64. Neither side reports anything.

Also, on MIPS a rule that begins with `#` is a comment line to the compiler; to match something that starts with `#`, write `[#]` or `\x23`.

## Writing rules that mean the same on all three

- Keep to the rows where all three columns say ✔. The rules seen in real use (e-mail, IPv4, an HTTP request line, a Host header, card numbers, URLs, hex digests, `(?i)union\s+select`, a file's magic bytes) gave the same result on all three.
- Set options with `(?i)` `(?s)` `(?x)`, not with `{i}` and its kin.
- **No anchor is common to all three.** The start of the payload is `\A` on arm64 and Gen4, and only the macro `{@0}` on Gen3. The end of the payload is `\z` on arm64 and Gen4, and has no counterpart on Gen3. Write a rule that must be anchored to either end once per kind of model.
- For "start of a line" and "end of a line", put the newline in the rule -- `\nHost: `, `\r\n` -- which all three read alike; a bare `^` or `$` has three meanings on three engines.
- Do not reach for possessive quantifiers or atomic groups to stop backtracking: MIPS either refuses them or does not honour them.
- Spell white space out: `[ \t\r\n\f\x0b]` where VT counts, `[ \t]` rather than `\h` for horizontal space; `(?:\r\n|\n|\r)` rather than `\R` for any line ending; `[^\n]` rather than `\N` for "not a newline".
- Where a group is not referred back to, write it as `(?:…)`; name groups as `(?<n>…)`, not `(?P<n>…)`.
- All three match bytes: write non-ASCII content byte by byte with `\xhh`, not with `\p{…}`.

## A rule the engine will not take

A filter holding a regex its engine does not accept **fails alone**: that filter matches nothing (as `F`, as `!F`, with `!=`, inside an `<or>` -- all the same), and every other filter and chain loads as usual. The log names the filter, the rule and the engine's own reason, and the filter counter shows it being tried and never matching (the first line below is MIPS, the second arm64):

```
[ERROR] filter 7002: regex "x(?>a+)ay" is not accepted by the HFA compiler (unexpected char after (?), so the filter matches nothing
[ERROR] filter 7002: regex "ab**c" is not accepted by PCRE2 (quantifier does not follow a repeatable item, at offset 3), so the filter matches nothing
```

It is made to match nothing, rather than to carry on one condition short, because a filter that has lost a condition no longer means what was written -- and one left with no condition at all matches every packet.

**6.6.261008.4 / 7.6.261008.1 and earlier have no such protection**, and one rule that is not accepted affects the whole configuration: on arm64 the filters and chains that come after it are not loaded; on MIPS, where all rules share one graph, no graph is built at all, regex matching cannot work, and the packet engine may stop. On those versions make sure every rule is within the supported syntax before applying it.

To check a rule beforehand:

- **arm64**: any PCRE2 tool (`pcre2test`, or an online tester set to PCRE2), with no flags.
- **MIPS**: compile that one rule on the device with its own compiler, which does not touch the running configuration; `-target` takes the value of `args.hfaTarget` in `GRISM.config.json`. No `/tmp/g` afterwards means the rule is not accepted, and the reason is printed. A rule that compiles is only a rule whose syntax is accepted: whether its result is PCRE's is what the tables above are for.

  ```bash
  printf '%s\r\n' 'the rule to check' > /tmp/p
  /usr/local/hfa/hfacg4 -target cn73xx -input /tmp/p -out /tmp/g -rc -force                # Gen4: T12S (cn73xx), the T20 family (cn78xx)
  /usr/local/hfa/hfac   -target cn70xx -input /tmp/p -out /tmp/g -warn /tmp/w -rc -force   # Gen3: G8S, G8, G4 (cn70xx)
  ```
