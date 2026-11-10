# Traffic by country: which end of a flow counts, and how

The "Traffic › Countries" page (`get_country_counter` in the API) adds traffic up by country. Since 6.6.261110 / 7.6.261110 (engines 2.11.44 / 2.9.47, Studio 2.269.0) it has two ways of doing so: **the default is exactly what it always did**; only once you give the ports a WAN/LAN side, or list the local networks, does it count the remote end alone and tell inbound from outbound. This page spells out both, and the order in which a flow is decided.

## What holds in both modes

- **The unit is a flow**, not a packet. A flow is counted when it leaves the flow table: idle timeout (30 s by default), active timeout (7200 s), 5 s after a TCP FIN/RST, or when the counters are cleared. The page therefore lags the traffic by one timeout.
- Each of a flow's two addresses is looked up once in the DB-IP country database (`dbip-country-YYYY-MM.mmdb`, replaceable under Settings › Services), giving a two-letter country or "not found". Private addresses, TEST-NET and unallocated space are not found.
- A flow's packets and bytes are **both directions summed**. The flow's **first packet** sets its source → destination direction, and the port it arrived on is remembered.
- IPv4 only. Flow tracking (`flow`) and `mmdb_country_counter` must be on and the database loaded.

## Mode one: both (the default)

With nothing configured this is the mode, and it is unchanged from earlier releases. For each flow:

1. If the source has a country, that country gets the whole flow (packets, bytes, flows + 1).
2. If the destination has a country and it differs from the source's, that country gets the whole flow too; the same country only once.
3. An end without a country adds nothing; a flow with neither (private to private, say) is not counted at all.
4. No direction: `in_*` / `out_*` stay 0, and so do `local` / `unknown` / `undirected`.

So a flow between two countries appears under both, the countries' sum can approach twice the traffic, and this site's own public addresses show up as their country, usually as the biggest row.

## Mode two: remote

It is on as soon as **any port** has a side (`side` = `wan` or `lan`) or `countryLocalNetworks` is not empty. Each flow is decided in this order:

1. Is each end local? An address inside one of the `countryLocalNetworks` networks, or a private one (10/8, 172.16/12, 192.168/16, 100.64/10, 169.254/16, 127/8), is local.
2. Both ends local: the whole flow goes to **local**, no country.
3. Exactly one end local: the other end is the remote end.
4. Neither end local: the side of the port the **first packet arrived on** decides. `wan` makes the source the remote end (the remote opened the flow); `lan` makes the destination the remote end (opened locally).
5. With the remote end known, if it has a country **that country alone** gets the whole flow, split by direction: `in_*` is traffic from the remote end to the local one, `out_*` the other way; a flow the remote end opened counts in `in_flows`, one opened locally in `out_flows`. A remote end without a country sends the whole flow to **unknown**.
6. A flow nothing could place (both ends public, neither listed, the port without a side) goes to **undirected** and is **also counted the both way, under both countries**. A number in that row means a port or a network is still unconfigured.

Two things easily confused:

- `in` / `out` is the **direction of the traffic**; `in_flows` / `out_flows` is **who opened the flow**. A local user downloading a file is one `out_flows` with a large `in_bytes`.
- The list wins over the port side, and a private address is local whatever port it came in on.

### A public address on the LAN side

With a side on both the WAN and the LAN port, no list is needed. A local public host connecting out has its first packet arrive on the LAN port, so the source is local; a connection from outside arrives first on the WAN port, so the source is remote. The list is for the cases where the ports cannot tell the sides apart, most often a SPAN that mirrors both directions into one port, or only one side cabled.

## Examples

The five flows used to verify it (DB-IP 2025-10 says: 8.8.8.8 and 8.8.4.4 US, 168.95.1.1 TW, 1.1.1.1 AU, 31.13.87.52 TW, 203.0.113.5 not found), under four settings:

| Flow (first packet's direction) | both | ingress port side=wan | ingress port side=lan | list 203.0.113.0/24, no side |
|---|---|---|---|---|
| 192.168.50.10 → 8.8.8.8 | US | US, opened locally | US, opened locally | US, opened locally |
| 168.95.1.1 → 1.1.1.1 | TW and AU, once each | TW, opened by the remote | AU, opened locally | undirected, plus TW and AU once each |
| 203.0.113.5 → 8.8.4.4 | US | unknown | US, opened locally | US, opened locally |
| 10.0.0.1 → 10.0.0.2 | not counted | local | local | local |
| 31.13.87.52 → 192.168.50.11 | TW | TW, opened by the remote | TW, opened by the remote | TW, opened by the remote |

Measured on a G8S-MEC and a Q16 under all four settings, the boxes matched this table exactly.

## How to set it

- **A port's side**: Settings › Interfaces has a new column per port, WAN, LAN or none. Only the ports you changed are sent; a side change reloads the engine configuration and traffic pauses for a few seconds. In the API:
  ```xml
  <configSet reboot="no">
    <interfaces><ports><find name="P0"><side>wan</side></find></ports></interfaces>
    <interfaces><ports><find name="P1"><side>lan</side></find></ports></interfaces>
  </configSet>
  ```
  `<side></side>` takes it away.
- **Local networks**: the "Local networks" field of the Flow tracking card under Settings › Packet handling: IPv4 addresses or networks, separated by spaces, commas or semicolons; an entry that does not parse is skipped. In the API:
  ```xml
  <configSet reboot="no"><args><countryLocalNetworks>203.0.113.0/24, 198.51.100.7</countryLocalNetworks></args></configSet>
  ```
  An empty element clears it.
- Both settings appear in the raw configuration XML (`get_config_xml`) and are carried across a firmware update.

## The JSON

Each entry of `country_counter[]` keeps the three fields it always had, `iso_code`, `packets`, `bytes`, meaning in both modes "both directions of every flow the country was in", so an older reader is unaffected. The new fields sit beside them:

| Field | Meaning |
|---|---|
| `flows` | flows counted under the country |
| `in_packets` / `in_bytes` | traffic from the remote end to the local one (non-zero in the remote mode only) |
| `out_packets` / `out_bytes` | traffic from the local end to the remote one |
| `in_flows` / `out_flows` | flows opened by the remote / the local end |
| top-level `mode` | `both` or `remote` |
| top-level `local` / `unknown` / `undirected` | each `{packets, bytes, flows}`, as above |

The Studio's country page says which mode is in force; in the remote mode it adds the inbound, outbound and "flows in / out" columns, and "local to local" follows the countries when it is not zero. `unknown` and `undirected` are in the JSON only; the page does not list them, so as not to raise questions it cannot answer there.

## What the mode does not change

- The `country.iso_code` filter is as before: it matches when either end is in that country, this site's own public addresses included.
- IPv6 flows get no country.
- A change of side or list reloads the engine configuration and the counters start from zero; so does "clear counters". The country page's own Clear (`clear_country_counter` in the API, operator level) zeroes the country statistics alone; port, filter and flow counters stay.
