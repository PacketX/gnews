# JA4 fingerprints: how GRISM computes them, how they are checked, what went wrong along the way

## What JA4 is

JA4 is FoxIO's TLS client fingerprint, computed from the structure of the ClientHello -- not the certificate, not the content -- so one program (a browser, a piece of malware, a library) gives the same value whatever it connects to, which is what makes it identify the client. It has three parts:

```
t13d1517h2_8daaf6152771_cb7bf5808d99
│ │ ││ │ └ first and last character of the ALPN
│ │ ││ └── number of extensions (17)
│ │ │└──── number of ciphers (15)
│ │ └───── d = SNI present, i = absent
│ └─────── TLS version (13 = 1.3)
└───────── protocol: t = TCP, q = QUIC
          ┌ JA4_b: SHA-256 of the sorted cipher list, first 12 hex digits
          └ JA4_c: sorted extensions (SNI and ALPN removed) + signature_algorithms in sent order, SHA-256, first 12
```

JA4S is the server-side counterpart (ServerHello): `t130200_1301_a56c5b993250` -- protocol, version, extension count, ALPN, the chosen cipher, the extension list hash.

## What GRISM offers

| Filter field | What it holds |
|---|---|
| `tls.handshake.ja4` | the whole JA4 |
| `tls.handshake.ja4_a` / `ja4_b` / `ja4_c` | each part on its own -- `ja4_b` alone is "every client with this cipher set" |
| `tls.handshake.ja4s`, `ja4s_a/b/c` | the server side |
| `ssl.ja3_digest`, `ssl.ja3s_digest` | the older JA3, still there |

The DPI SSL syslog carries `ja4=` and `ja4s=` for every connection, so the fingerprint distribution of a whole network can go to a SIEM without a single filter.

## Checking it: a reference that shares no code

A firmware's JA4 cannot be checked against the firmware. `tools/test/ja4.py` is a reference calculator written from scratch: it parses the pcap, Ethernet/VLAN/IP/TCP, reassembles each TCP direction, parses the TLS records and the ClientHello, follows nothing but FoxIO's specification, shares not a line with the device, and checks itself against the specification's example. It prints every ClientHello's JA4 and its **JA4_r** (the unhashed lists), so a mismatch can be traced to a field rather than guessed at.

The procedure: replay a browser capture through a LOOP port with a `replayPcap` input, point the DPI SSL syslog at the test host, and compare the syslog's `ja4=` connection by connection with `ja4.py`'s answers. `tools/test/mkhello.py` synthesises ClientHellos that differ only in their ALPN (binary values, over-long names, `--ccs` for a leading ChangeCipherSpec) to check the rules one at a time.

## 2026-10-03: three deviations

Compared that way, the implementation differed from the specification in three places, each of which made one class of client **always** wrong:

| Deviation | Effect | Fix |
|---|---|---|
| Only the first TLS record of a TCP segment was looked at. TLS 1.3's middlebox-compatibility mode (RFC 8446 D.4) sends the ClientHello after a HelloRetryRequest behind a six-byte ChangeCipherSpec record | such a hello was never fingerprinted | step over leading CCS records to the handshake record; an Alert or Application Data record in front means there is no hello behind it |
| JA4_c kept GREASE values from signature_algorithms, and Chrome puts a GREASE value **first** in that list (`dada,0904,0905,…`) | every Chrome-family client's `ja4_c` matched neither the specification, Wireshark, nor any published database | the specification ignores GREASE everywhere; skip `0x?a?a` there too |
| The two ALPN characters came from a table of known names (h2, dot, http/x, spdy/x); everything else was `00` | any ALPN not in the table was wrong | the specification's rule: the first and last byte of the first ALPN value, or the first and last hex digit when either is not alphanumeric; the first ALPN is kept with its length in a 256-byte buffer, so binary values and names longer than 23 bytes no longer fall to `00` |

Verified on the Q16 by replay: three CCS-prefixed hellos, GREASE sigalgs and nine synthetic ALPN cases all match the reference.

## 2026-10-06: wrong on a different connection every time

After that, the arm64 models still got about one Chrome hello in a hundred wrong -- but **a different connection each replay**, which is not a parsing error but a race. The cipher and extension lists were split with `strtok()`, whose "where am I" pointer is one per process, and the DPDK work cores are threads of one process: two cores fingerprinting at once cut each other's lists short. With `strtok_r()` a replay of 1365 hellos had zero errors (3 wrong in 546 before). The full analysis is in [strtok on many cores](/en/tech-strtok.md).

## QUIC: the ClientHello is encrypted

HTTP/3 runs over QUIC, where the TLS handshake sits in CRYPTO frames inside Initial packets, and the whole Initial is AEAD-protected. The good news is that anyone can derive the keys: they come by HKDF from nothing but the Destination Connection ID in the packet (RFC 9001 §5.2) -- the protection is against middleboxes altering the handshake, not for secrecy. So to compute a QUIC JA4 the device has to remove header protection (§5.4), derive the Initial keys, AEAD-decrypt (§5.3), and reassemble the CRYPTO frames by offset (RFC 9000 §19.6).

Reassembly is not optional: **every browser's hello spans 2–3 datagrams and arrives scrambled**. Post-quantum key exchange made the ClientHello bigger than a datagram; Chrome splits one hello into a dozen out-of-order CRYPTO frames (even the four-byte handshake header is cut up), Firefox sends the tail first. Firefox also pads the datagram with zeros after the Initial's Length -- legal, not malformed.

What GRISM does (setting `quicInitialReassemble`, off by default, next to the other reassembly switches): decrypt the client's Initials, collect the CRYPTO frames by offset until the hello is whole, then hand it to the same code the TCP path uses. So `tls.handshake.ja4*`, `ssl.server_name`, `quic.tag == CHLO` and the SSL syslog all see QUIC, with the JA4's `q` prefix. The crypto is portable C (`quic_initial.h`) with three backends: the OCTEON core's AES/GFM hardware on the MIPS models, OpenSSL (which uses the ARMv8 Crypto Extensions) on the arm64 models, and a software reference for host-side tests. QUIC v1 and v2 (RFC 9369) are both handled.

**A browser's QUIC JA4 is not its TCP JA4**; do not filter QUIC with the TCP value. QUIC is TLS 1.3 only, so browsers offer 3 ciphers (`ja4_b` = `55b375c5d22e`) instead of 15 (`8daaf6152771`):

| Browser | QUIC JA4 |
|---|---|
| Firefox | `q13d0315h3_55b375c5d22e_dc5437974b47` |
| Chrome (fresh) | `q13d0312h3_55b375c5d22e_178839b6cec1` |
| Chrome (resumption) | `q13d0314h3_55b375c5d22e_79cc91d6b50c` |

The same Chrome over TCP is `t13d1517h2_8daaf6152771_cb7bf5808d99`: protocol, cipher count, extension count, ALPN (h2 against h3) and all three hashes differ.

Checking it again rests on an independent implementation: `tools/test/quic_ja4.py` decrypts every connection's Initials in a pcap with Python's `cryptography`, reassembles, computes the JA4, and can emit step-by-step test vectors (keys, each datagram's plaintext and frames). `quic_initial_test.c` runs those on the build host together with the sample packets of RFC 9001/9369 and the published FIPS/HKDF vectors. On the devices: a Q16 replaying 89 Firefox and 25 Chrome connections matched the oracle 100%; a G8S gets every value right too, but a replay through a LOOP port loses a few percent of hellos -- a connection's Initials have to meet on one core, which the MIPS models arrange by descheduling a QUIC long header to the flow's core as they do a TCP segment, and a LOOP port cannot; real ports have no such loss.

## The tools

| Tool | Use |
|---|---|
| `tools/test/ja4.py <pcap>` | reference JA4 / JA4_r of TCP ClientHellos |
| `tools/test/quic_ja4.py <pcap> [--json out]` | the QUIC reference; `--vectors` writes C test vectors, `--selftest` checks itself |
| `tools/test/mkhello.py out.pcap h2 http/1.1 hex:0102 none [--ccs]` | synthetic hellos differing only in ALPN |
| `tools/test/quic_initial_test.c` | host-side decryption test (`gcc … sha256.c`; add `-DQUIC_CRYPTO_OPENSSL -lcrypto` for the OpenSSL backend) |

The two references deliberately share no code with the device, and only the JA4 string assembly with each other -- that is what lets them be the judge.
