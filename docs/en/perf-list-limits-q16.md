# How many entries a Q16 takes in a big filter list

How many `ip.addr` (single addresses or CIDR), `ipv6.addr` or `dns.qry.name` entries can one filter hold on a Q16? This page is what a Q16 (the DPDK line) was measured to take on firmware 7.6.261104 with the shipped settings. The G8S is [here](/en/perf-list-limits.md) and the T12S [here](/en/perf-list-limits-t12s.md); each of the three runs into a different wall.

## Where the memory is

What sets the Q16 apart from the MIPS models is that the lists do not all live in one place:

- **The hash tables (`dns.qry.name`, single-address `ip.addr`) live in hugepages.** The box has 48 GB of RAM; 10 GB of it is reserved at boot as hugepages (20 pages of 512 MB) for the packet engine, whose own flow tables take about 4.5 GB, leaving **about 5.5 GB (11 pages)** for lists. Each hash table needs one contiguous block.
- **The trees (`ip.addr` CIDR, `ipv6.addr`) live in the engine process's ordinary memory**, not in hugepages: roughly 20 GB is usable after the hugepages and everything else.
- **Uploaded lists go to `/data`**, a 4 GB tmpfs (RAM as well): the upload, the copy in running-config and the converted GDP file all sit there.

## Test set-up

| Item | Setting |
|---|---|
| Model | **Q16** (16 1G/10G ports behind a Marvell switch) |
| Firmware | 7.6.261104.2 |
| IPv4 flow table | shipped default (`flowCacheBaseSize` 23000) |
| IPv6 flow table | shipped default (`flowv6TableSize` 30000) |
| `tryRunXmltoGdp` | on (live loading; the default from this release) |
| Everything else | shipped defaults, MEC off, `run.xml` empty, no traffic; 11 hugepages free when idle |

## Method

As on the G8S page: one kind of entry per `run1.xml`, one filter, one `<find>` a line -- random public IPv4 addresses, random /24s, random /128s under `2001:db8::/32`, host-name-like names of about 24 characters; the new file copied over the old one by scp into `ftproot` with its `.ok` (exactly what the web page's submitxml does), **without clearing the previous one first**. A size passes when three lists of that size and different content, uploaded one after the other, each take effect with the engine running on and nothing in the log, **and then a full reload (an empty instant configuration) and a reboot both keep the list**; the list is removed and the box rebooted before the next size. The tools are `tools/test/gdp_tree/limits.py` and `limits_search.py` in the firmware tree (x1.5 up until a failure, then halving to within 15%; single addresses and CIDR were stopped at a cap). A Q16 reboots in about 2 minutes.

## One kind at a time

| Kind | Limit (three updates, a full reload and a reboot all pass) | Fails at | XML size | What runs out |
|---|---|---|---|---|
| `ip.addr`, single addresses | **at least 10,000,000** (measured no further) | — | 611 MB | nothing: the table is 107 MB and /data half empty |
| `ip.addr`, CIDR | **at least 12,000,000** (measured no further) | — | 752 MB | nothing: a ~2.9 GB tree in ordinary memory |
| `ipv6.addr` | **2,200,000** | 3,000,000 | 193 MB | Linux memory: two trees of ~29 GB together -- **the box runs out of memory and its management services die** (below) |
| `dns.qry.name` | **7,600,000** | 8,400,000 | 578 MB | hugepages: two 2.9 GB tables do not fit in 5.5 GB |

What an entry costs:

| Kind | Where | Per entry | Example |
|---|---|---|---|
| `ip.addr`, single | hugepage hash table | 8 B × 4/3 | 10,000,000 entries ≈ 107 MB |
| `dns.qry.name` | hugepage hash table | 260 B × 4/3 ≈ 347 B | 7,600,000 entries 2.6 GB; 8,400,000 entries 2.9 GB; 10,000,000 entries 3.5 GB |
| `ip.addr`, CIDR | tree in ordinary memory | random /24 about 240 B | 12,000,000 entries ≈ 2.9 GB |
| `ipv6.addr` /128 | tree in ordinary memory | random /128 about 4.3 KB | 2,200,000 entries ≈ 9.4 GB; 3,000,000 ≈ 12.9 GB |

- **The hugepage rule: every hash table in use, plus a second copy of the one being updated, must fit in 5.5 GB at once.** A live load builds "the lists in use plus the new one" before switching, so one dns table can be about 2.6 GB (7,600,000 names); a 2.9 GB second copy is not granted (`failed to alloc from bootmem named FCDATA_HASH_TABLE_0_9101 size: 2912000000`), the log says `GDP reload failed: staying on hash table N`, the old list stays in force and the box is fine.
- **The ordinary-memory rule: the trees in use, plus a second copy of the one being updated, must fit in about 20 GB.** Two 9.4 GB IPv6 trees (2,200,000 addresses) do; at 3,000,000 the second upload builds a second 12.9 GB tree and **is not refused -- it uses up Linux's memory**: the OOM killer takes sshd, nginx, pywww and snmpd, the packet engine keeps forwarding, but ssh and the web page are gone and only a power cycle brings them back. This differs from the MIPS models, whose trees live in bootmem and simply fail to allocate, keeping the old list; the Q16's trees are ordinary malloc, which Linux grants first and pays for later. **From 7.6.261105 (engine 2.9.45) the tree is sized first and refused when the memory is not there (2 GB is kept back for everything else)**, with `filter N: its M-prefix tree would take X GB ... not built` and `GDP reload failed: staying on hash table N` in the log, the old list in force and the box fine (measured: the first 3,000,000-address tree of 12.05 GB built, the second was refused with 7.5 GB left). On the releases before that, **keep a list of random single IPv6 addresses under 2,000,000 entries**; whole networks written as CIDR cost far less.
- **The `/data` rule: two copies of the XML (the upload and the one in running-config) must fit in 4 GB at once**; nothing on this page comes near it.
- With several kinds loaded together all three rules apply at once; the tool below works it out.

## Several lists at once

The tool uses the per-entry costs measured on this page: the hash tables (dns, single addresses; H together) need 2H in the hugepages; the trees (CIDR, IPv6; T together) need T plus a second copy of the tree being updated in ordinary memory; a full reload or a boot needs H and T once each. Move the four counts: each row shows the most that kind can hold with the others fixed, the table lists the headroom of every operation, and the two bars show the hugepages and the ordinary memory. Only single lists were measured on the Q16; the combinations are the model's. Tree sizes depend on how the prefixes are spread, so change the per-entry figures in "Model parameters" for your own lists.

<iframe src="tools/list-limits-66.html?model=q16&lang=en" data-autoheight style="width:100%;height:900px;border:1px solid #e5e8eb;border-radius:6px" title="Limits with several lists loaded (Q16)"></iframe>

## What happens when it fails

1. **No room in the hugepages (dns, single addresses).** The log says `failed to alloc from bootmem named ...` and `GDP reload failed: staying on hash table N`; the new list is not in effect, **the old one stays**, the box is fine; the alert rule "Filter list not applied" fires.
2. **No room in ordinary memory (the CIDR and IPv6 trees).** Up to 7.6.261104 there is no safe failure: the memory is used up, the OOM killer ends the management services and the box needs a power cycle. From 7.6.261105 the tree is sized before it is built and fails like a hash table, the old list stays and the alert fires. Either way, work it out with the tool above first and keep two trees well under 20 GB together.
3. **No room at a boot or a full reload.** Every list is loaded from its GDP file again; a hash table that does not fit logs `run1.xml.gdp load fail ... not in effect` and **that filter is empty**; the other lists are unaffected.

## How long a load takes

| Action | Time |
|---|---|
| `dns.qry.name` 7,600,000 entries, live | 5 s upload + 45 s to take effect |
| `ip.addr` CIDR 12,000,000 entries, live | 7 s upload + 84 s to take effect |
| `ipv6.addr` 2,200,000 entries, live | 2 s upload + 28 s to take effect |
| `ip.addr` 10,000,000 entries, live | 6 s upload + 45 s to take effect |
| Full reload with 7,600,000 `dns.qry.name` entries loaded | 40 s |
| Reboot | about 2 minutes; the lists are rebuilt from their GDP files at boot (a 3,000,000-address IPv6 tree in 8 s) |

## Other notes

- Copy big lists by scp to `/data/ftproot/run1.xml` and `touch run1.xml.ok`; the web page's submitxml sends the whole XML as a form field, which is not meant for files of hundreds of MB.
- The lists are kept in `/usr/local/grism/etc/running-config/` and loaded again at every boot; a list that does not fit at boot is tried again at every boot.
