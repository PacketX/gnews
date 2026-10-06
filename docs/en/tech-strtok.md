# strtok on many cores: the string is private, its state is not

## What we saw

Replaying the same browser capture through an arm64 (DPDK) model, nearly every TLS ClientHello got the right JA4, but about 1–2% came out wrong on every run — and **a different connection each time**:

| Correct JA4 | What came out now and then |
|---|---|
| `t13d1517h2_8daaf6152771_cb7bf5808d99` | `t13d0317h2_b0e173700538_cb7bf5808d99` |
| | `t13d1017h2_24938229c549_cb7bf5808d99` |
| | `t13d1507h2_8daaf6152771_43af166abec8` |

The damage differed: sometimes the cipher count dropped and JA4_b was wrong while JA4_c was right; sometimes the extension count dropped and JA4_c was wrong. The same capture never went wrong on a MIPS model.

"A different connection each run" is the clue. A parsing bug fails on the same packet every time; a random failure is a **timing** problem.

## Why

JA4 splits the cipher and extension lists (strings such as `4865-4866-4867-...`), sorts them and hashes them. The split used the C library's `strtok()`.

The string being split was each call's own `strdup()` copy, so it looked private. The problem is not the string but **the pointer `strtok()` keeps of where it has got to**. The first call passes the string, later calls pass `NULL` to mean "carry on from last time", and that position lives in a static variable inside libc:

```c
char *strtok(char *s, const char *delim)
{
    static char *olds;              /* one for the whole process */
    return __strtok_r(s, delim, &olds);
}
```

The Linux man page marks `strtok()` as `MT-Unsafe race:strtok`.

- **arm64 (DPDK)**: the work cores are **threads of one process**. They share one address space, and so one `olds`.
- **MIPS (OCTEON)**: each core is **a process of its own**, with its own `olds`.

"Private" only means that nothing else is meant to use the copy. Every thread of a process can read and write its whole heap, so once another core points `olds` at its own copy, this core goes on splitting someone else's string.

## The sequence

Taking `t13d0317h2_…`:

| Time | Core A (splitting its cipher list) | Core B (another ClientHello) | `olds` points at |
|---|---|---|---|
| 1 | `strtok(dupA, "-")` → 1st | | dupA |
| 2 | `strtok(NULL, "-")` → 2nd | | dupA |
| 3 | `strtok(NULL, "-")` → 3rd | | dupA |
| 4 | | `strtok(dupB, "-")` starts | **dupB** |
| 5 | | splits to the end of dupB | end of dupB |
| 6 | `strtok(NULL, "-")` → carries on from the end of dupB → `NULL` | | |

Core A takes the list to be over after 3 ciphers: JA4_a becomes `t13d03…` and JA4_b is the hash of those 3. The extensions are split later, undisturbed, so JA4_c is still right. When it is the extension list that gets cut short, the result is the `t13d1507h2_…` kind. It can also go the other way: core A carries on splitting core B's string, or writes `\0` into core B's copy and spoils both results.

It takes two cores splitting **at the same moment**, which is why the failures are random and land on a different connection each run.

## The fix

Use the re-entrant `strtok_r()`, which keeps the position in a variable the caller owns, on its own stack: one per call, one per core.

```c
char *save = NULL;
char *tok = strtok_r(dup, "-", &save);
while (tok) {
    /* ... */
    tok = strtok_r(NULL, "-", &save);
}
```

A single `strtok(s, ":;")` used only to cut a string short writes the shared `olds` too, and can break off another core's split mid-way. It was replaced with code that touches no shared state:

```c
size_t lead = strspn(s, ":;");
s[lead + strcspn(s + lead, ":;")] = '\0';
```

## Verification

One arm64 unit, one capture, one source tree differing only by this fix. Every JA4 the device reported was checked against an independent reference implementation:

| Firmware | Replays | TCP ClientHellos | Wrong |
|---|---|---|---|
| without the fix | 2 | 546 | 3 (different connections each run) |
| with the fix | 5 | 1365 | 0 |

## Takeaways

- Code that several cores run at once must not call functions with hidden static state. Besides `strtok()`, the usual ones are `strerror()`, `localtime()`, `gmtime()`, `asctime()`, `ctime()`, `rand()` and `inet_ntoa()`. Use the `_r` versions (`strtok_r`, `localtime_r`, `strerror_r`, …) or code that keeps no shared state.
- "The data is private" does not make a call thread-safe. What matters is whether the function keeps shared state inside.
- Code that is correct with one process per core can break with one thread per core. Check for this when porting between the two.
- Random results that fail somewhere different every time are usually a race. Replaying a fixed input and checking every result against a reference implementation is the quickest way to confirm one.
