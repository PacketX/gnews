# The regex filter: PCRE syntax support on the two product lines

## Two different engines

`<find name="regex">` is two entirely different implementations on the two product lines, and one rule is not guaranteed the same result on both:

| | arm64 (DPDK) models, 7.6.x | MIPS models, 6.6.x |
|---|---|---|
| Engine | PCRE2 10.42 (software) | Cavium HFA (hardware-assisted) |
| How rules reach it | each rule is compiled on its own | every regex of the configuration, one per line, is compiled into **one** graph |
| Compiler | built into the engine | Gen4 on CN73/78 (T12S, the T20 family); Gen3 on the rest (G8S, G8, G4…) |
| Defaults | case-sensitive; `.` stops at a newline; `^` `$` are the ends of the payload | case-sensitive; `.` stops at a newline; `^` `$` **also hold at every newline** |

Both match **bytes** (there is no UTF-8 mode), search the L7 payload by default, and search each packet on its own, never across packets.

```xml
<filter id="10" name="sql">
  <or>
    <find name="regex" relation="==" content="(?i)union\s+select" />
  </or>
</filter>
```

## How it was tested

137 rules in 14 groups (literals, escapes, anchors, quantifiers, character classes, POSIX classes, shorthands, back-references, groups, assertions, inline options, Cavium extensions, rules seen in real use…), each with one to five payloads, some that should match and some that should not. Every (rule, payload) pair was replayed through a loopback port of the device and the device's verdict read from the filter counter, then compared with what **the PCRE2 10.42 library itself** answers for the same pair.

| | Result |
|---|---|
| arm64 (HL1) | all 135 rules PCRE2 accepts gave PCRE2's answer on every payload |
| MIPS Gen4 (T12S) | the compiler accepts 126; of the 123 that can be compared with PCRE2, 107 agree on every payload and 16 differ (below) |
| MIPS Gen3 | compiler only: it accepts what Gen4 accepts, less `**` |

Two of the 137 are not PCRE at all (`\x{4142}` and `**`); PCRE2 itself refuses them.

## Syntax support

✔ = accepted, with PCRE2's result.

| Syntax | arm64 | MIPS |
|---|---|---|
| literals, escapes such as `\.` | ✔ | ✔ |
| `\xhh` `\x{hh}` `\ddd` (octal) `\t` `\r` `\n` `\a` `\e` `\f` `\cA` `\Q…\E`; NUL and bytes from 0x80 up | ✔ | ✔ |
| `.`, `a\|b`, groups `( )`, non-capturing `(?:…)` | ✔ | ✔ |
| `*` `+` `?` `{n}` `{m,n}` `{m,}` and the lazy forms `*?` `+?` `??` `{m,n}?` | ✔ | ✔ |
| classes `[abc]` `[^abc]` `[a-f0-9]` `[\x00-\x1f]` `[\d\s]` | ✔ | ✔ |
| POSIX classes `[[:alnum:]]` … `[[:xdigit:]]` (all 14), negated `[[:^digit:]]` | ✔ | ✔ |
| `\d` `\D` `\w` `\W` `\S` `\h` `\H` | ✔ | ✔ |
| back-references `\1` `\g1` `\g{1}` `\g{-1}`; named groups `(?<n>…)` `(?'n'…)` with `\k<n>` `\k'n'` `\k{n}` | ✔ | ✔ |
| `\b` `\B` `\A` `\z` `\Z` | ✔ | ✔ |
| lookaround `(?=…)` `(?!…)` `(?<=…)` `(?<!…)` | ✔ | ✔ |
| inline options `(?i)` `(?i:…)` `(?-i)` `(?s)` `(?m)` `(?x)`, comments `(?#…)` | ✔ | ✔ |
| `^` `$` | the ends of the payload | **differs**: also at every newline |
| possessive quantifiers `*+` `++` `?+` `{m,n}+` | ✔ | **differs**: accepted, but they behave like the ordinary (greedy) ones |
| `\v` `\s` `\V` | ✔ | **differs**: `\v` is the one character VT (0x0B), so `\s` lacks VT and `\V` matches a newline |
| `\N` `\o{…}` `\G` | ✔ | **differs**: accepted, and never match |
| `\K` | ✔ | accepted and ignored (whether the rule matches is unaffected) |
| Python-style named groups `(?P<n>…)` `(?P=n)` | ✔ | ✘ |
| atomic groups `(?>…)`, branch reset `(?\|…)`, conditionals `(?(1)…)`, recursion and subroutine calls `(?1)` | ✔ | ✘ |
| `(?U)`, `(*…)` start-of-pattern items, callouts `(?C)` | ✔ | ✘ |
| `\R`, Unicode properties `\p{…}` | ✔ | ✘ |
| Cavium macros `{i}` `{s}` `{x}` `{t}` `{@n}` (below) | read as literal text | ✔ |
| multi-byte hex `\x{4142}` | ✘ | ✔ |
| `**` | ✘ | Gen4 ✔, Gen3 ✘ |

The HFA syntax specification and the measurement disagree in three places (Gen4): it says `^` is replaced by `\n`, yet the start of the payload matches too; it says `\h` includes the newline, yet it does not, as in PCRE; and `(?:…)`, which it does not list, works.

## Accepted, with a different answer

These are the easy ones to get wrong, because neither side complains -- the answers just differ:

| Rule | Payload | arm64 | MIPS |
|---|---|---|---|
| `^abc` | `zz⏎abcdef` | no match | match |
| `abc$` | `abc⏎zz` | no match | match |
| `^abc$` | `x⏎abc⏎y` | no match | match |
| `xa++ay` | `xaaay` | no match | match |
| `x\sy` | `x`, VT, `y` | match | no match |
| `x\vy` | `x⏎y` | match | no match |
| `x\Vy` | `x⏎y` | no match | match |
| `a\Nb` | `axb` | match | no match |
| `x\o{101}y` | `xAy` | match | no match |
| `\Gabc` | `abcx` | match | no match |

(⏎ is the newline, 0x0A.)

## Cavium macros (MIPS only)

An HFA rule may begin with a `{…}` block, its attributes separated by commas, as in `{i,s}a.c`:

| Macro | Meaning | The form both lines take |
|---|---|---|
| `{i}` | case-insensitive | `(?i)` |
| `{s}` | `.` matches a newline too | `(?s)` |
| `{x}` | white space in the rule is ignored | `(?x)` |
| `{@n}` | match only n bytes from the start; `{@0}` is the start | `\A` for the start |
| `{t}` | report where the match begins (does not change whether it matches) | — |

On arm64 the block is **literal text, not a macro**: `{i}hello` is a case-insensitive hello on MIPS and the eight characters `{i}hello` on arm64. Neither side reports anything.

Also, on MIPS a rule that begins with `#` is a comment line to the compiler; to match something that starts with `#`, write `[#]` or `\x23`.

## Writing rules that mean the same on both

- Keep to the upper half of the table, where both columns say ✔. The rules seen in real use (e-mail, IPv4, an HTTP request line, a Host header, card numbers, URLs, hex digests, `(?i)union\s+select`, a file's magic bytes) gave the same result on both.
- Set options with `(?i)` `(?s)` `(?x)`, not with `{i}` and its kin.
- Anchor to the ends of the payload with `\A` and `\z`; where a per-line anchor is meant, say so with `(?m)^` and `(?m)$`. A bare `^` or `$` gets two different answers once the payload holds a newline.
- Do not reach for possessive quantifiers or atomic groups to stop backtracking: MIPS either refuses them or does not honour them.
- For white space that includes VT write `[ \t\r\n\f\x0b]`; for any line ending write `(?:\r\n|\n|\r)` rather than `\R`; for "not a newline" write `[^\n]` rather than `\N`.
- Name groups as `(?<n>…)`, not `(?P<n>…)`.
- Both lines match bytes: write non-ASCII content byte by byte with `\xhh`, not with `\p{…}`.

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
- **MIPS**: compile that one rule on the device with its own compiler, which does not touch the running configuration; `-target` takes the value of `args.hfaTarget` in `GRISM.config.json`. No `/tmp/g` afterwards means the rule is not accepted, and the reason is printed.

  ```bash
  printf '%s\r\n' 'the rule to check' > /tmp/p
  /usr/local/hfa/hfacg4 -target cn73xx -input /tmp/p -out /tmp/g -rc -force                # Gen4: T12S (cn73xx), the T20 family (cn78xx)
  /usr/local/hfa/hfac   -target cn70xx -input /tmp/p -out /tmp/g -warn /tmp/w -rc -force   # Gen3: G8S, G8, G4 (cn70xx)
  ```
