# How many entries a G8S takes in a big filter list

How many `ipv6.addr`, `ip.addr` or `dns.qry.name` entries can one filter hold? This page is what a G8S was measured to take, and how the three kinds of list get in each other's way when they are loaded together. The tool at the bottom lets you set all three counts and see whether they fit.

## Test set-up

| Item | Setting |
|---|---|
| Model | **G8S** |
| Firmware | 6.5.260910 |
| IPv4 flow table | **160,000** (`flowCacheBaseSize` 10000, ×16) |
| IPv6 flow table | **50,000** (`flowv6TableSize` 50000) |
| Linux memory free after boot | about 340 MiB |
| Otherwise | an empty `run.xml`, no traffic |

## Method

- **One kind of condition per file.** Each list sits in a `runN.xml` of its own holding one filter, and every `<find>` in it is of the same kind.
- **Contents.** `ipv6.addr`: random /128s under `2001:db8::/32`. `ip.addr`: random IPv4 addresses. `dns.qry.name`: names that look like real ones, such as `cdn74.27dcfee4f4.io`, 24 characters on average.
- **How a list is updated.** The new file is written over the old `runN.xml` (into `ftproot`, then the `.ok` marker), **without clearing the previous one first** -- what replacing a list in service looks like.
- **What counts as success.** Three different lists of the same size uploaded one after another, and every time:
  - the filter's count becomes the new list's;
  - every other filter keeps its count;
  - the engine pids stay the same and no `grism-linux_64` is a zombie (`Z`);
  - the CPU does not sit at 100%;
  - `/grism/get_log` reports no error.
- **A clean start for every size.** The test lists are deleted and the box rebooted before the next size.

## One kind at a time

| Condition | Limit (three uploads in a row) | Sizes that failed | XML size |
|---|---|---|---|
| `ipv6.addr` | **12,000** | 12,500 and 13,000 on the 2nd upload | about 1 MB at 12,000 |
| `ip.addr` | **1,250,000** | 1,375,000 on the 3rd, after two good ones; 1,500,000 on the 2nd; 2,000,000 on the 1st | about 76 MB at 1,250,000 |
| `dns.qry.name` | **1,100,000** | 1,200,000 on the 2nd | about 84 MB at 1,100,000 |

- **`ipv6.addr` costs Linux memory.** IPv6 conditions are kept as a bit-by-bit radix tree in every core's engine process, about 18.8 KiB an entry. Loaded once, 15,000 still fit; 16,000 got an engine process OOM-killed into a zombie, and only a reboot brought it back.
- **`ip.addr` and `dns.qry.name` are limited by the size of the XML.** Both are turned into GDP and loaded live into hash tables in bootmem, and those tables are not the limit at these sizes. Linux is: the new upload and the old copy in running-config both sit in the `/data` tmpfs, which is RAM, and the watcher needs another block of memory to parse the new file.
- **About 84 MB of XML is the edge.** `ip.addr` at that size passed some times and failed others; `dns.qry.name` just passed. Keep one list's XML under about 76 MB for some headroom.

## All three together

The three lists share one pool of Linux memory, so more of one leaves room for less of the others. Each combination below started from a clean reboot and was uploaded in order, then updated in turn.

| Combination | Result |
|---|---|
| `ip.addr` 1,000,000 + `dns.qry.name` 800,000 | updated three times each, in turn: all good |
| `ip.addr` 1,200,000 + `dns.qry.name` 960,000 | the dns list went in; updating the ip list then failed (out of memory) and the old ip list stayed |
| `dns.qry.name` 1,000,000 + `ipv6.addr` 3,000 → 4,000 | every IPv6 update good |
| `dns.qry.name` 1,000,000 + `ipv6.addr` 5,000 | on the next IPv6 update **the dns filter disappeared** |
| `dns.qry.name` 1,000,000 + `ipv6.addr` 6,000 | the same: the dns filter disappeared |
| `ip.addr` 500,000 + `dns.qry.name` 500,000 + `ipv6.addr` 7,500 | updating the dns list failed (out of memory); the old one stayed |
| `ip.addr` 500,000 + `dns.qry.name` 500,000 + `ipv6.addr` 6,500 | all three updated three times each, in turn: all good (predicted by the model beforehand) |

## What a failure looks like

Three kinds, not equally bad:

1. **An ip / dns update does not fit.** The log says `parse error at line 1: out of memory`; the new list does not take effect and **the old one stays in force**. The box carries on normally.
2. **An IPv6 update does not fit.** Nothing in the log -- it still says `reload grism` -- but the new file is never moved into running-config, so **the old list stays in force**. Only the filter's count shows it.
3. **A full reload cannot parse a big list again.** Updating an IPv6 list or changing `run.xml` makes the engine parse every `runN.xml` afresh. When that does not fit, the log says `runN.xml parsing fail` and **that ip / dns filter disappears** until a later reload that fits. This is the one case where the configuration in force actually shrinks, so on a box with big lists, check the filter counts after a configuration change.

Apart from those, an IPv6 list loaded once at more than about 15,000 entries gets an engine process OOM-killed (a fourth kind), and the box needs a reboot.

## A model of how they interact

With F the memory free after boot (about 340 MiB), all three rules have to hold:

- **Updating an ip / dns list:** F − IPv6 tables − every list's XML − the new file ≥ 36 MiB + 1.8 × the new file.
- **Updating the IPv6 list:** F − IPv6 tables − every list's XML ≥ 110 MiB.
- **A full reload:** F − IPv6 tables − every list's XML ≥ 2.5 × the largest ip / dns XML.

The IPv6 tables take about 18.8 KiB an entry. An XML line is about 61 bytes for `ip.addr`, 52 + the name's length for `dns.qry.name`, and 88 for `ipv6.addr`. The rules were fitted to the 16 measured passes and failures above; the last three-way combination was predicted with the model first and then measured, and it came out as predicted.

The model holds for the set-up above. Another model, other flow-table sizes or a big `run.xml` of its own leave a different amount of memory after boot; F can be changed under the tool's "Model parameters".

## Tool: limits with all three loaded

Drag or type the three counts. Under each row is the most that list can hold with the other two as set; the table below shows how much headroom each operation has left.

<iframe src="tools/list-limits.html?lang=en" data-autoheight style="width:100%;height:900px;border:1px solid #e5e8eb;border-radius:6px" title="Limits with all three lists loaded"></iframe>

## Other things to know

- **The web upload on this firmware stops at 10 MB.** 6.5.260910's Go web app answers 502 to a `submitxml` over about 10 MB, so big lists go into `/data/ftproot` by scp, followed by the `.ok` marker.
- **A boot parses every `runN.xml` too.** Every size here was deleted and the box rebooted before the next, so booting with lists near these limits was not tested.
