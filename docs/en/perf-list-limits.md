# How many entries a G8S takes in a big filter list

How many `ip.addr` (single addresses or CIDR), `ipv6.addr` or `dns.qry.name` entries can one filter hold? This page is what a G8S was measured to take on firmware 6.6.261103. The older firmware's results (6.5.260910) are kept on [the old page](/en/perf-list-limits-6.5.md); both the limits and the ways things fail differ, so do not mix the two.

## What changed in this firmware

- **IPv4 CIDR and IPv6 lists load live too.** Before, only single IPv4 addresses and domain names were turned into GDP and loaded live; CIDR and IPv6 lists took a full reload, which built a tree once per core in Linux memory (four cores on a G8S). Now these two are GDP as well, and every core reads one tree in bootmem.
- **A full reload or a boot no longer parses the lists once per core.** Editing `run.xml`, submitting an instant configuration or rebooting reloads everything; each `runN.xml` used to be parsed on every core, and every list that can load live now comes from its GDP file instead.
- **Converting a big file takes one copy less of memory.** The XML-to-GDP conversion used to hand the whole file to the parser, which copies it into a buffer of its own; it is now fed a megabyte at a time.
- **Running out of room is logged, not silent.** A conversion that cannot be written logs an ERROR and keeps the list in use; the file is no longer moved by `system("mv")`, which failed without a word when memory was short and let a full reload bring the previous list back (2,100,000 `ip.addr` entries did that on 6.6.261103.2 and pass since the fix).

## Test set-up

| Item | Setting |
|---|---|
| Model | **G8S** (MEC off) |
| Firmware | 6.6.261103.2 and .3 (the failure handling is in .3, and so is the `ip.addr` figure) |
| IPv4 flow table | **160,000** (`flowCacheBaseSize` 10000, ×16) |
| IPv6 flow table | **50,000** (`flowv6TableSize` 50000) |
| `tryRunXmltoGdp` | on (live loading) |
| Linux memory free after boot | about 190 MiB (about 340 MiB on the old firmware: every image carries Python and nginx now, and pywww takes about 80 MiB more) |
| Everything else | `run.xml` empty, no traffic |

## Method

- **One kind of entry per file.** Each kind goes into `run1.xml` as one filter whose `<find>`s are all of that kind, one per line.
- **Content.** `ip.addr` single addresses are random public IPv4s; CIDR entries random public /24s; `ipv6.addr` random /128s under `2001:db8::/32`; `dns.qry.name` names that look like real host names, about 24 characters on average.
- **How a list is updated.** The new file is copied over the old `run1.xml` by scp (into `ftproot`, then the `.ok` -- exactly what the web page's submitxml does), **without clearing the previous one first**.
- **What counts as a pass.** Three lists of the same size and different content uploaded one after the other, and every time: the filter's count becomes the new list's; the engine's pids stay and none is a `Z`; the CPU does not sit at 100%; the log says nothing of out of memory, parse error or fail; the web API keeps answering. **Then a full reload (an empty instant configuration) and a reboot** must both keep the list. The old page did not test those two.
- **Clean between sizes.** `run1.xml` is removed and the box rebooted before the next size. The tools are `tools/test/gdp_tree/limits.py` (one size) and `limits_search.py` (bisection) in the firmware tree.

## One kind at a time

| Kind | Limit (three updates, a full reload and a reboot all pass) | Fails at | XML size | What runs out |
|---|---|---|---|---|
| `ip.addr`, single addresses | **2,100,000** | 2,200,000 | 2,100,000 entries ≈ 128 MB | Linux memory: two copies of the XML in RAM; pywww gives out first |
| `ip.addr`, CIDR | **1,700,000** | 1,800,000 | 1,700,000 entries ≈ 106 MB | bootmem: two trees |
| `ipv6.addr` | **92,000** | 96,000 | 92,000 entries ≈ 8 MB | bootmem: two trees |
| `dns.qry.name` | **1,300,000** | 1,400,000 | 1,300,000 entries ≈ 99 MB | bootmem: two hash tables |

What an entry costs:

| Kind | Where | Per entry | Example |
|---|---|---|---|
| `ip.addr`, single | bootmem hash table | 8 B × 4/3 | 2,100,000 entries ≈ 22 MB |
| `dns.qry.name` | bootmem hash table | 260 B × 4/3 | 1,300,000 entries 450 MB; 1,400,000 entries 485 MB |
| `ip.addr`, CIDR | bootmem tree | 56 B for every bit a prefix does not share with the one before it; random /24s about 237 B | 2,100,000 random /24s 475 MB |
| `ipv6.addr` /128 | bootmem tree | as above; a random /128 about 5 KB | 110,000 random /128s 475 MB; 92,000 about 400 MB |

- **The bootmem rule: the new list's table plus every table in use must fit.** A live load builds "the lists in use plus the new one" in the other index before switching, so while a big list is being updated its old and new tables exist at once. With the flow tables at 10000/50000 this G8S updates a list whose table is about 450 MB (dns 1,300,000) and cannot place the second copy of a 485 MB one (dns 1,400,000); the CIDR trees sit in the same range. Every table needs one contiguous block.
- **The Linux memory rule: two copies of the XML in RAM.** `/data` is a tmpfs, so the upload and the old file in running-config both take RAM, plus the GDP file while converting. A hash table of single addresses is small, so that kind runs into this rule: 128 MB of XML (2,100,000 entries) pass; at 134 MB pywww hangs during the second upload and the web API stops answering for minutes (it recovered by itself), and sshd is gone for good -- the engine and the lists are fine, but ssh needs a reboot from the web page.
- With several kinds loaded together both rules apply at once; the tool below works it out.

## Several lists at once

The binding constraint is the bootmem rule above: every table in use, added up, plus a second copy of the one being updated, has to fit at once; a full reload or a boot instead needs the sum of one copy of each. dns, CIDR and IPv6 each take a slab; the single-address table is tiny and barely touches bootmem, so that kind is bounded instead by the two copies of its XML in the `/data` tmpfs. The tool works these rules from the per-entry costs measured on this page: move the four counts, each row shows the most that kind can hold with the others fixed, the table below shows the headroom left for every operation, and the bar shows the four tables plus the hatched live-update copy of the largest against the bootmem budget. Tree sizes depend on how the prefixes are spread -- whole networks written as CIDR, or addresses clustered in a few networks, cost far less than the same count of random /24s or /128s, so change the per-entry figures in "Model parameters" for your own lists. This is a model fitted to the four single-list limits; the combinations were not each re-measured as the old page did.

<iframe src="tools/list-limits-66.html?lang=en" data-autoheight style="width:100%;height:900px;border:1px solid #e5e8eb;border-radius:6px" title="Limits with several lists loaded"></iframe>

## What happens when it fails

1. **No room in bootmem.** The log says `failed to alloc from bootmem named ...` and `GDP reload failed: staying on hash table N`; the new list is not in effect, **the old one stays**, the box is fine.
2. **No room in RAM (while converting the XML).** The log says `run1.xml: writing /data/ftproot/run1.gdp failed ...` (ERROR); the new list is dropped, **the old one stays**. Up to 6.6.261103.2 this left no trace: the half-written GDP file loaded as a success, the old list stayed, all three updates looked fine, and only a full reload showed the first list still there.
3. **No room at a boot or a full reload.** Every list is loaded from its GDP file again; one that does not fit logs `run1.xml.gdp load fail ... not in effect` and **that filter is empty**; the other lists are unaffected. A box with big lists should have its filter counts checked after a configuration change.

## How long a load takes

| Action | Time |
|---|---|
| `ip.addr` 2,100,000 entries, live | 12 s upload + 63 s to take effect |
| `dns.qry.name` 1,300,000 entries, live | 9 s upload + 52 s to take effect |
| `ip.addr` CIDR 1,700,000 entries, live | 10 s upload + 75 s to take effect |
| `ipv6.addr` 92,000 entries, live | 2 s upload + 11 s to take effect |
| Full reload with 2,100,000 `ip.addr` entries loaded | 65 s |
| Reboot | about 100 s; the lists load from their GDP files at boot |

## Other notes

- **A big upload through the web page kills pywww.** An 11 MB list sent to submitxml on a G8S took pywww to 322 MB and the OOM killer took it; nothing restarts it on a MIPS box. Copy big lists by scp to `/data/ftproot/run1.xml` and `touch run1.xml.ok`, which is all submitxml does.
- **A single IPv6 address is expensive.** A random /128 takes about 5 KB of tree, where consecutive addresses in one prefix are cheap (300 /64s with 100 addresses each, 30,000 entries, took 4.8 MB). Write whole prefixes as CIDR.
