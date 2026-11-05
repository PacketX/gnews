# How many entries a T12S takes in a big filter list

How many `ip.addr` (single addresses or CIDR), `ipv6.addr` or `dns.qry.name` entries can one filter hold on a T12S? This page is what a T12S was measured to take on firmware 6.6.261104 with the shipped flow tables. The G8S is [here](/en/perf-list-limits.md) and the Q16 [here](/en/perf-list-limits-q16.md).

## Where the memory is

The T12S is the same product line as the G8S (MIPS) and keeps its lists in the same place -- hash tables and trees in bootmem -- only with far more of it:

- **32 GB of DRAM**, of which Linux gets 4 GB and the packet engine the rest as bootmem. The shipped flow tables (IPv4 `flowCacheBaseSize` 150000, IPv6 `flowv6TableSize` 500000) take 4.6 GB; **about 13.8 GB was measured to be available for lists** (from the largest pair of IPv6 trees that fit at once). Every table and tree needs one contiguous block.
- **Uploaded lists go to `/data`**, a 2 GB tmpfs (Linux RAM as well): the upload, the copy in running-config and the converted GDP file all sit there. Linux has about 2.7 GB free after a boot.

## Test set-up

| Item | Setting |
|---|---|
| Model | **T12S** (MEC off) |
| Firmware | 6.6.261104.3 |
| IPv4 flow table | shipped default (`flowCacheBaseSize` 150000, ×16 = 2,400,000) |
| IPv6 flow table | shipped default (`flowv6TableSize` 500000) |
| `tryRunXmltoGdp` | on (live loading; the default from this release) |
| Everything else | `run.xml` empty, no traffic |

## Method

As on the G8S page: one kind of entry per `run1.xml`, one filter, one `<find>` a line -- random public IPv4 addresses, random /24s, random /128s under `2001:db8::/32`, host-name-like names of about 24 characters; the new file copied over the old one by scp into `ftproot` with its `.ok` (exactly what the web page's submitxml does), **without clearing the previous one first**. A size passes when three lists of that size and different content, uploaded one after the other, each take effect with the engine running on and nothing in the log, and a full reload (an empty instant configuration) keeps the list. **A reboot with the list was verified up to 4,800,000 names** (the boot converts the list to GDP again, ~90 s, and the filter count is whole afterwards): this test box's management port (a USB NIC) sometimes fails to come up at boot and needs someone at the box to power-cycle it, so not every size was rebooted as on the G8S page; the boot loads a list through the same path as a reload, the size only changes the time. The tools are `tools/test/gdp_tree/limits.py` and `limits_search.py` in the firmware tree (x1.5 up until a failure, then halving to within 15%; single addresses were stopped at a cap of 10,000,000). A T12S reboots in about 2.5 minutes, plus the time the boot takes to convert the list again (below).

## One kind at a time

| Kind | Limit (three updates and a full reload pass) | Fails at | XML size | What runs out |
|---|---|---|---|---|
| `ip.addr`, single addresses | **at least 10,000,000** (measured no further) | — | 611 MB | nothing: the table is 107 MB, /data at 583 MB |
| `ip.addr`, CIDR | **at least 12,000,000** (measured no further) | — | 752 MB | nothing: a ~2.9 GB tree, /data at 717 MB |
| `ipv6.addr` | **1,600,000** | 1,800,000 | 140 MB | bootmem: two 7.7 GB trees do not fit in 13.8 GB |
| `dns.qry.name` | **11,000,000** | 12,000,000 | 837 MB | /data: two copies of the XML plus the converted GDP file do not fit in 2 GB (`writing run1.gdp failed: No space left on device`) |

What an entry costs:

| Kind | Where | Per entry | Example |
|---|---|---|---|
| `ip.addr`, single | bootmem hash table | 8 B × 4/3 | 10,000,000 entries ≈ 107 MB |
| `dns.qry.name` | bootmem hash table | 260 B × 4/3 ≈ 347 B | 6,000,000 entries 1.9 GB; 11,000,000 entries 3.6 GB |
| `ip.addr`, CIDR | bootmem tree | random /24 about 242 B | 4,500,000 entries 1.0 GB; 12,000,000 ≈ 2.9 GB |
| `ipv6.addr` /128 | bootmem tree | random /128 about 4.3 KB (a little less the more there are: 4.4 KB at 500,000, 4.28 KB at 2,400,000) | 1,600,000 entries 6.9 GB; 1,800,000 7.7 GB; 2,400,000 10.3 GB |

- **The bootmem rule is the G8S's: every hash table and tree in use, plus a second copy of the one being updated, must fit in about 13.8 GB at once.** Two 6.9 GB IPv6 trees (1,600,000 addresses) do; at 1,800,000 the second 7.7 GB tree is not granted (`failed to alloc from bootmem named FCDATA_TREE_9101_1 size: 7747469312`), the log says `GDP reload failed: staying on hash table N`, the old list stays in force and the box is fine. The other three kinds are nowhere near it at the sizes measured.
- **The `/data` rule: the XML being uploaded, the previous XML in running-config and the converted GDP file must fit in 2 GB at once.** This is what stops dns: 11,000,000 names (837 MB of XML, a GDP file of about 350 MB) pass; at 12,000,000 (913 MB) the second upload cannot write its GDP file, the log says `writing /data/ftproot/run1.gdp failed: No space left on device -- the list in use is kept, the new one is dropped` and the old list stays in force; at 14,000,000 (1.07 GB) not even the second XML fits and the scp itself fails. The XML of single addresses and CIDR is smaller, with room to spare at the sizes measured.
- With several kinds loaded together both rules apply at once; the tool below works it out.

## Several lists at once

The tool uses the per-entry costs measured on this page: the hash tables (dns, single addresses; H together) and the trees (CIDR, IPv6; T together) are all bootmem, so a hash-list update needs 2H + T, a tree update 2H + T + that tree, and a full reload or boot H + T, each within 13.8 GB. Move the four counts: each row shows the most that kind can hold with the others fixed, the table lists the headroom of every operation, and the bar shows the four tables plus the hatched live-update copy against the budget. Only single lists were measured on the T12S; the combinations are the model's. Tree sizes depend on how the prefixes are spread, so change the per-entry figures in "Model parameters" for your own lists.

<iframe src="tools/list-limits-66.html?model=t12s&lang=en" data-autoheight style="width:100%;height:900px;border:1px solid #e5e8eb;border-radius:6px" title="Limits with several lists loaded (T12S)"></iframe>

## What happens when it fails

As on the G8S:

1. **No room in bootmem.** The log says `failed to alloc from bootmem named ...` and `GDP reload failed: staying on hash table N`; the new list is not in effect, **the old one stays**, the box is fine; the alert rule "Filter list not applied" fires.
2. **No room in RAM (while converting the XML).** The log says `run1.xml: writing /data/ftproot/run1.gdp failed ...` (ERROR); the new list is dropped, **the old one stays**.
3. **No room at a boot or a full reload.** Every list is loaded from its GDP file again; one that does not fit logs `run1.xml.gdp load fail ... not in effect` and **that filter is empty**; the other lists are unaffected.

## How long a load takes

| Action | Time |
|---|---|
| `dns.qry.name` 11,000,000 entries, live | 40 s upload + 270 s to take effect (6,000,000: 22 s + 148 s) |
| `ip.addr` CIDR 12,000,000 entries, live | 36 s upload + 330 s to take effect (8,000,000: 24 s + 219 s) |
| `ipv6.addr` 1,600,000 entries, live | 7 s upload + 72 s to take effect |
| `ip.addr` 10,000,000 entries, live | 29 s upload + 196 s to take effect |
| Full reload with 11,000,000 `dns.qry.name` entries loaded | 244 s (6,000,000: 139 s; 12,000,000 CIDR: 311 s) |
| Reboot | about 2.5 minutes, plus the boot's conversion of the list to GDP and its load: about 90 s for 4,800,000 names, with the CPU at 100% (the packet engine is not forwarding yet) |

## Other notes

- Copy big lists by scp to `/data/ftproot/run1.xml` and `touch run1.xml.ok`; the web page's submitxml sends the whole XML as a form field, which blew pywww up at 11 MB on a G8S -- a T12S has more Linux memory but is no place for files of hundreds of MB either.
- The lists are kept in `/usr/local/grism/etc/running-config/` and converted and loaded again at every boot, so the boot takes longer with every list.
