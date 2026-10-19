# A 5G and 4G/NSA test bed with UERANSIM, srsRAN and Open5GS, and GRISM's MEC breakout under test

## Goal

Run a core (Open5GS: both the 5GC and the EPC) and base-station/handset simulators (UERANSIM for 5G SA, srsRAN 4G for LTE and NSA) on one Linux VM, with N2/S1-MME (NGAP and S1AP over SCTP) and N3/S1-U (GTP-U) **really crossing a wire** through a GRISM inline chain, then add an edge host and verify MEC breakout: when a UE in a chosen cell reaches the edge service, GRISM strips the GTP and hands the packet to the edge directly, wrapping the reply back towards the base station; everything else goes on to the core, which never sees the broken-out part.

The machines: a GRISM G8S-MEC (P4 on the base-station side, P5 on the core side, P3 to the edge host) and an Ubuntu 24.04 VM (8 cores, 16 GB, four NICs).

## Layout

```
VM                                                        GRISM G8S-MEC
┌─────────────────────────────────┐
│ [ran namespace]                 │
│   ens19  10.5.0.1  gNB A        │───── P4 ──┐
│          10.5.0.11 gNB B        │           │  inline chain
│          10.5.0.12 gNB C        │           │  P4 <-> P5
│          10.5.0.21 eNB (LTE+NR) │           │
│          UE1 UE2 UE3 (uesimtunN)│           │
│          UE4 (tun_srsue)        │           │
│                                 │           │
│ [host]                          │           │
│   ens20  10.5.0.2  AMF(N2) UPF(N3)│───── P5 ──┘
│          10.5.0.3  SGW-U(S1-U)  MME(S1-MME) on 10.5.0.2 too
│   ogstun 10.45.0.1/16 ─ NAT ─ ens18 ─ Internet
│                                 │
│ [edge namespace]                │
│   ens21  10.45.255.254  HTTP    │───── P3 (edge: where breakout leaves)
└─────────────────────────────────┘
```

| Interface | namespace | Address | Role | GRISM port |
|---|---|---|---|---|
| ens19 | `ran` | 10.5.0.1 / .11 / .12, 10.5.0.21 | N2/S1 addresses of the three gNBs and the eNB; the UEs live here too | P4 |
| ens20 | host | 10.5.0.2, 10.5.0.3 | AMF (NGAP) and MME (S1AP), UPF (GTP-U) on .2; SGW-U (GTP-U) on .3 | P5 |
| ens21 | `edge` | 10.45.255.254/16 | the edge host (an HTTP service) | P3 |
| ens18 | host | 192.168.x.x | management, and the NAT exit for UE traffic | — |

Two things decide the layout:

- **The gNB side has to sit in its own network namespace.** Two NICs of one host in the same namespace never put a packet on the wire for each other -- the kernel hands it over locally. Separated, every packet from a gNB to the AMF leaves ens19, crosses GRISM and comes back in on ens20.
- **So does the edge host.** The host already holds the UE range 10.45.0.0/16 on the UPF's `ogstun`, and the edge host's address has to be inside that range (see the MEC configuration below); on the host, its replies to UEs would go into `ogstun` instead of out of the wire.

## Installation

```bash
apt install -y software-properties-common git make gcc g++ cmake libsctp-dev lksctp-tools docker.io iptables

# MongoDB, which Open5GS needs: 5.0 and later require AVX, which QEMU's default CPU model lacks, so 4.4 in Docker
systemctl enable --now docker
docker run -d --name open5gs-mongo --restart unless-stopped -p 127.0.0.1:27017:27017 -v open5gs-mongo:/data/db mongo:4.4

# Open5GS 2.8
add-apt-repository -y ppa:open5gs/latest && apt install -y open5gs

# UERANSIM
git clone --depth 1 https://github.com/aligungr/UERANSIM /opt/UERANSIM && make -C /opt/UERANSIM -j8
```

The PPA does not ship `open5gs-dbctl`; take it from the Open5GS source (`misc/db/open5gs-dbctl`). It calls `mongosh`, which a wrapper hands to the container:

```sh
#!/bin/sh
exec docker exec -i open5gs-mongo mongo "$@"      # saved as /usr/local/bin/mongosh
```

## Open5GS configuration

Only two addresses change: the AMF's N2 and the UPF's N3 bind to ens20; every other interface stays on loopback. The three gNBs use different TACs, so `tai` lists them all.

```yaml
# /etc/open5gs/amf.yaml
amf:
  ngap:
    server:
      - address: 10.5.0.2
  tai:
    - plmn_id: { mcc: 999, mnc: 70 }
      tac: [1, 2, 3]
# /etc/open5gs/upf.yaml
upf:
  gtpu:
    server:
      - address: 10.5.0.2
```

Subscribers with UERANSIM's sample keys:

```bash
for i in 1 2 3; do open5gs-dbctl add 99970000000000$i 465B5CE8B199B49FAA5F0A2EE238A6BC E8ED289DEBA952E4283B54E88E6183CA; done
```

UE traffic leaves the UPF's `ogstun` and has to be NATed out. Docker sets the FORWARD policy to DROP, so allow the UE range explicitly:

```bash
sysctl -w net.ipv4.ip_forward=1
iptables -I FORWARD -s 10.45.0.0/16 -j ACCEPT
iptables -I FORWARD -d 10.45.0.0/16 -j ACCEPT
iptables -t nat -A POSTROUTING -s 10.45.0.0/16 -o ens18 -j MASQUERADE
```

## Wiring: namespaces and addresses

```bash
ip netns add ran;  ip link set ens19 netns ran
ip -n ran addr add 10.5.0.1/24 dev ens19; ip -n ran addr add 10.5.0.11/24 dev ens19; ip -n ran addr add 10.5.0.12/24 dev ens19
ip -n ran link set lo up; ip -n ran link set ens19 up
ip addr add 10.5.0.2/24 dev ens20
ip netns add edge; ip link set ens21 netns edge
ip -n edge addr add 10.45.255.254/16 dev ens21; ip -n edge link set lo up; ip -n edge link set ens21 up
mkdir -p /etc/netns/ran && echo "nameserver 8.8.8.8" > /etc/netns/ran/resolv.conf
```

Check the wire first: `ping 10.5.0.2` from inside `ran` while watching the P4/P5 counters on GRISM's traffic page. Both directions have to pass.

## UERANSIM configuration

Three gNBs on one PLMN with different cells and TACs; each UE names its gNB in `gnbSearchList`. Start from `config/open5gs-gnb.yaml` and `config/open5gs-ue.yaml`.

| gNB | `nci` | `tac` | `linkIp` / `ngapIp` / `gtpIp` | `amfConfigs` | UE |
|---|---|---|---|---|---|
| A | `0x001000000` | 1 | 10.5.0.1 | 10.5.0.2 | imsi-999700000000001 |
| B | `0x002000000` | 2 | 10.5.0.11 | 10.5.0.2 | imsi-999700000000002 |
| C | `0x003000000` | 3 | 10.5.0.12 | 10.5.0.2 | imsi-999700000000003 |

Everything runs inside the `ran` namespace:

```bash
ip netns exec ran /opt/UERANSIM/build/nr-gnb -c gnb-A.yaml      # likewise B and C
ip netns exec ran /opt/UERANSIM/build/nr-ue  -c ue-1.yaml       # likewise 2 and 3
```

A registered UE gets a `uesimtun0..2` (10.45.0.x) in `ran`. To make curl and ping in `ran` "use the phone", point the namespace's default route at `uesimtun0` -- but **pin the core's address to the wire first** (`ip -n ran route add 10.5.0.2/32 dev ens19`), or the gNB's own N2/N3 ends up routed into a UE tunnel.

The edge host is one line of Python:

```bash
ip netns exec edge python3 -m http.server 80 --bind 10.45.255.254 --directory /srv/edge
```

## 4G / NSA: an srsRAN 4G eNB with an NR cell

UERANSIM does 5G SA only. For LTE and NSA (EN-DC) the bed uses [srsRAN 4G](https://github.com/srsran/srsRAN_4G)'s srsENB and srsUE with ZMQ in place of radios -- no SDR needed; NSA is srsENB carrying an extra NR cell and srsUE with its NR RAT on, as in srsRAN's "5G NSA over ZMQ" application note.

Building release 23.11 on this VM needs two options: GCC 13's `array-bounds` false positives are errors unless `-DENABLE_WERROR=OFF`, and srsRAN defaults to `-march=skylake`, whose binary **dies with SIGILL at start** on a CPU without AVX, so `-DGCC_ARCH=x86-64-v2` (this CPU has SSE4.2/POPCNT, no AVX).

```bash
cmake -DGCC_ARCH=x86-64-v2 -DENABLE_WERROR=OFF -DENABLE_AVX=OFF -DENABLE_AVX2=OFF -DENABLE_FMA=OFF -DENABLE_ZEROMQ=ON ..
make -j8 srsenb srsue
```

The configs start from `srsenb/*.conf.example` and `srsue/ue.conf.example`; everything runs inside the `ran` namespace:

| File | Changes |
|---|---|
| `enb.conf` | `mcc=999 mnc=70`, `mme_addr=10.5.0.2`, `gtp_bind_addr=s1c_bind_addr=10.5.0.21`, `n_prb=50`; `[rf] device_name=zmq`, `device_args=fail_on_disconnect=true,tx_port0=tcp://*:2000,rx_port0=tcp://localhost:2001,tx_port1=tcp://*:2100,rx_port1=tcp://localhost:2101,id=enb,base_srate=23.04e6` (port 0 is LTE, port 1 NR) |
| `rr.conf` | the LTE cell as in the example (`cell_id=0x01`, `tac=0x0007`, `dl_earfcn=3350`); one NR cell in `nr_cell_list`: `rf_port=1; cell_id=0x02; tac=0x0007; pci=500; root_seq_idx=204; dl_arfcn=368500; band=3;` |
| `ue.conf` | the matching two ZMQ port pairs in `[rf]`; `[rat.eutra] dl_earfcn=3350 nof_carriers=1`; `[rat.nr] bands=3,78 nof_carriers=1`; `[rrc] release=15`; `[usim] imsi=999700000000004` with UERANSIM's sample keys |

Two things on the core side: `mme.yaml` binds `s1ap` to 10.5.0.2 and lists TAC 7 in `tai`; `sgwu.yaml` binds `gtpu` to 10.5.0.3 (an extra address on ens20, since the UPF already holds 10.5.0.2:2152). The HSS reads the same MongoDB, so a subscriber from `open5gs-dbctl add` serves 4G and 5G alike.

Two lessons: **start the eNB before the UE** -- a UE restarted on its own hangs at "Attaching UE..." (ZMQ connection state) and is fine when the eNB restarts with it; and `apt install` of libraries makes needrestart restart Open5GS daemons, after which every PDU session is gone (the SMF forgets them, the UPF answers the old TEIDs with GTP-U Error Indication): every UE drops at once, and re-registering base stations and UEs after any apt run is the cure.

Once attached, the UE holds `tun_srsue` (10.45.0.x, from the same pool as the 5G UEs); "RRC NR reconfiguration successful" and a random access on the NR cell on its console are EN-DC in place.

## GRISM configuration

### 1. Turn the MEC features on

Settings → the MEC card switches `s1cCorrelation`, `in-tunnels/GTP` and `flowExtensionGtpTunnelhdr` on together. `s1cCorrelation` makes the engine parse NGAP: it takes each UE's cell from the InitialUEMessage and both directions' TEIDs from the PDU session setup; GTP decapsulation and the flow extension let user-plane packets be matched back to those sessions.

### 2. The breakout chain

Start from the Studio's built-in MEC sample (Pipeline → MEC → open the sample) and put in the real ports and addresses: P4 is the gNB, P5 the core, P3 the edge.

```xml
<run>
  <filter id="1" name="UE addresses" sessionBase="no">
    <or><find name="ip.src" relation="==" content="10.45.0.0/16" /></or>
  </filter>
  <filter id="2" name="ARP for a UE" sessionBase="no">
    <and>
      <find name="arp.request.target.ip" relation="==" content="10.45.0.0/16" />
      <find name="arp.request.target.ip" relation="!=" content="10.45.255.254" />
    </and>
  </filter>
  <filter id="3" name="to the edge server" sessionBase="no">
    <or><find name="ip.dst" relation="==" content="10.45.255.254" /></or>
  </filter>
  <filter id="4" name="cell A only" sessionBase="no">
    <or><find name="gtp.data.by.s1ap.CellIdentity" relation="==" content="16384" /></or>
  </filter>
  <output id="1" name="to edge, GTP removed" arp_dstip_mac="yes">
    <port>P3</port><stripping>gtp</stripping><gateway>10.45.255.254</gateway>
  </output>
  <output id="2" name="back to gNB in GTP"><port>P4</port><tagging>gtp2</tagging></output>
  <output id="3" name="ARP reply"><port>P3</port><arp_reply_default_mac/></output>
  <chain><in>P5</in><out>P4</out></chain>
  <chain>
    <in>P4</in>
    <fid type="and">F1,F3,F4</fid>
    <out>O1</out>
    <next type="notmatch"><out>P5</out></next>
  </chain>
  <chain>
    <in>P3</in>
    <fid>F2</fid>
    <out>O3</out>
    <next type="notmatch"><out>O2</out></next>
  </chain>
</run>
```

What each part does:

- Core → gNB (P5 → P4) passes untouched.
- gNB → core (P4): a packet from a UE address (F1) **and** for the edge host (F3) **and** from cell A (F4) has its GTP stripped and leaves on P3 with the MAC that ARP resolves for `gateway` (O1); everything else goes on to P5.
- Edge → (P3): the edge host's ARP requests for UE addresses are answered by GRISM (F2 → O3); every other packet is wrapped in GTP again, using the tunnel learned earlier, and sent back to the gNB (O2). This is why the edge host's address has to be inside the UE range: the edge side ARPs for UEs as if they were hosts on its own LAN.

### 3. Working out the cell value

`gtp.data.by.s1ap.CellIdentity` compares the **top N bits** of the 36-bit NR Cell Identity from NGAP; N is 26 unless `nrCellIdentityBits` on the MEC card says otherwise (1–32). gNB A's NCI `0x001000000` at 26 bits is `0x001000000 >> 10 = 16384`; B and C are 32768 and 49152. At 32 bits the value is the 32-bit gNB ID (A = 1048576).

4G comes through S1AP: the top 20 bits of the 28-bit ECI, which is the eNB ID. srsENB's `enb_id=0x19B` gives a filter value of 411. 4G and 5G sessions share one table, so one configuration can name cells of both kinds.

The sessions the engine has learned are visible in the Studio's MEC table or from `GET /grism/task/get_s1ap_items`: each UE's address, its gNB, `cellid` and both TEIDs. Check the cell values there before writing a cell policy.

## Results

**Plain traffic** (no cell restriction, three UEs browsing): through its tunnel each UE did HTTP 204, HTTPS 200 (85 KB, 0.12 s), DNS and pings to 8.8.8.8 at about 8.6 ms, all successful. A six-second capture on GRISM's P4 held 164 GTP-U packets and the NGAP SCTP heartbeats: the N2/N3 going through the inline chain.

**MEC breakout** (no cell restriction): all three UEs fetched the edge page with `curl http://10.45.255.254/` (HTTP 200; 1.07 s the first time, with the ARP, 5 ms after that). On ens21 the traffic arrives as plain IP (`10.45.0.3 → 10.45.255.254:80`), the edge host's ARP for 10.45.0.3 is answered by GRISM, and the core's `ogstun` saw 0 packets for the edge address. Filter counters: F1 198/201, F3 16/198, F2 3/19.

**Cell policy** (three UEs on one PLMN and one address range, differing only in their cell):

| Policy | UE1 (cell A) | UE2 (cell B) | UE3 (cell C) |
|---|---|---|---|
| F4 = 16384 (cell A only) | edge HTTP 200 | timeout | timeout |
| F4 = 32768 (cell B only) | timeout | edge HTTP 200 | timeout |

Under both policies all three UEs still reach the internet, and F4 only counts hits on the allowed cell's packets. With `nrCellIdentityBits` set to 32 and F4 to 1048576 the outcome is the same.

**4G / NSA**: srsUE attached (the MME logged "Attach complete"), EN-DC came up (three NR reconfigurations and three NR random accesses), and the UE browsed through S1-U. GRISM's table gained a 4G row: `UE 10.45.0.7 | base station 10.5.0.21 | MME-UE-S1AP-ID 7 | eNB-UE-S1AP-ID 2 | cellid 411`; an eNB numbering its UEs 1, 2 next to gNBs numbering theirs 1, 2 stays correct, because the table is keyed by the id together with the base station's address. An S1AP capture showed InitialUEMessage, InitialContextSetupRequest (SGW-U 10.5.0.3, TEID 0x2d75) and the ICS Response bundled with an UplinkNASTransport as three DATA chunks of one SCTP packet, all parsed, nothing fragmented.

| Policy | LTE UE (cell 411) | UE1 (cell A) | UE2 (cell B) |
|---|---|---|---|
| F4 = 411 (the LTE cell only) | edge HTTP 200 | timeout | timeout |
| F4 = 16384 (cell A only) | timeout | edge HTTP 200 | timeout |

## Handover: Xn and N2 with synthesised NGAP

Neither UERANSIM nor srsRAN 4G moves a UE from one base station to another, so handover cannot be tested with real UEs. `/opt/5g/load/mecho.py` stands in: it encodes every message of a handover from pycrate's NGAP definitions (`mecho.py check` decodes each one back and checks the procedure codes and IE ids against pycrate's NGAP-Constants), wraps them in SCTP, IP and Ethernet and sends them through a raw socket on ens19 inside the `ran` namespace, so they enter GRISM on P4 exactly as a real gNB's signalling does. No core has to exist for them: the synthetic gNBs live in 10.6.0.0/16, the AMF and UPF in 10.7.0.0/24, both blackholed on the host; the messages are there for GRISM to see and nothing else.

```
VM [ran namespace]                                   GRISM G8S-MEC
  ens19 ── raw socket:                                 P4 ──┐ inline chain
    gNB g   10.6.x.y   cell 65536+g   (source)               │ P4 <-> P5
    gNB g+1 10.6.x.y+1 cell 65537+g   (target)               │
    AMF/UPF 10.7.0.250 (exist only inside the packets) P5 ──┘
  host: blackhole 10.6.0.0/16, 10.7.0.0/24; reads get_s1ap_items over the management network
```

Each session is first set up as a registration would be (InitialUEMessage → PDUSessionResourceSetupRequest/Response → one uplink G-PDU), then handed over from its gNB g to gNB g+1 -- another address, another cell, and a target that numbers its UEs on its own:

| Procedure | Messages (direction) | What GRISM has to do |
|---|---|---|
| Xn handover (path switch) | PathSwitchRequest (target gNB → AMF: new RAN id, the source AMF id, the new cell in UserLocationInformation, the new downlink tunnel) → PathSwitchRequestAcknowledge (AMF → target gNB: a new AMF id, the uplink tunnel) → an uplink G-PDU from the target gNB | both messages go into the path-switch staging table and, once paired, become the real node |
| N2 handover (through the core) | HandoverRequest (AMF → target gNB: AMF id, uplink tunnel) → HandoverRequestAcknowledge (target → AMF: new RAN id, downlink tunnel) → HandoverNotify (target → AMF: the new cell) → the source gNB's UEContextReleaseComplete → an uplink G-PDU from the target gNB | the handover staging table pairs and makes the real node, HandoverNotify sets the cell, the source node is released |

`--verify` reads `get_s1ap_items?ue-ipv4=<UE>` before and after; a session passes when exactly one node is left and its base station address (`ran-ipv4`), `cellid` and RAN id are the target gNB's.

```bash
python3 /opt/5g/load/mecho.py check                       # encoder self-check, sends nothing
python3 /opt/5g/load/mecho.py xn --count 200 --verify     # Xn handover for 200 sessions
python3 /opt/5g/load/mecho.py n2 --count 200 --verify     # N2 handover for the same batch
```

Run it on the host, not under `ip netns exec ran`: it enters the `ran` namespace only to open the raw socket (a socket keeps the namespace it was created in) and reads GRISM's table from the host, where the management route is. It needs the `ran` namespace and the two blackhole routes, nothing else of the bed.

**Results.** Before the fix both kinds scored 0/200: the handed-over node sat under "RAN id + base station address 0" with no cell, so after a handover the UE no longer matched the cell policy (F4) and lost its breakout. With the fix (engine 2.11.23 / 2.9.26 on) Xn and N2 score 200/200 each; the same frames written to a pcap and replayed from a LOOP port through a `replayPcap` input in `submit_instant` score 100/100 on an HL1 that is nowhere near the bed, so any box can be tested this way. Three things were fixed: the base station's address did not follow the session through the handover staging tables; 5G did not parse HandoverNotify at all, nor the UserLocationInformation in PathSwitchRequest, so only 4G updated the cell after a handover; and the NGAP dispatcher also ran a plain add on a PathSwitchRequest, leaving a node with the source AMF id and the target RAN id.

**To see what GRISM did with each MEC message**, set `s1apItemsUpdateLog` under `<args>` to `True` with `submit_config` and give one syslog target the `<type>` `mec` (the settings page does not list that type; export the current logging configuration with `get_config` and change one target): every add, pairing, handover and release of the table goes to that target in order. For a handover problem it is much faster than a capture.

**Not simulated:** 4G X2/S1 handover (S1AP's PathSwitchRequest and HandoverNotify share the tables and 4G has always parsed them, but no synthetic run has covered them), and 5G PDUSessionResourceModify / ModifyIndication, which are not parsed, so a modification that changes a tunnel is not followed.

## A UE with two PDU sessions / bearers

The table holds one node per UE (AMF/MME UE id + RAN UE id), but a VoLTE phone has two PDU sessions at once, internet and IMS, plus a dedicated bearer during a call; each has an uplink TEID of its own, and each PDU session its own UE address. The data path finds the node by one key in each direction: the TEID for an uplink G-PDU, the UE address for what comes back from the edge. So the keys of the session that is broken out must keep leading to the node, or breakout stops without a word.

**4G and 5G differ.** The S1AP parser takes E-RAB id 5 only (InitialContextSetup, E-RAB Setup and every modify and handover message alike); other bearers never reach the node, so the IMS and call bearers cannot disturb the internet bearer's breakout. The price is the assumption that the first PDN's default bearer is the internet one: a network whose first PDN is IMS would have IMS learned instead. A ten-minute S1AP capture from a live 4G network (4,471 UE contexts) bears the assumption out there: every one of 3,469 InitialContextSetups carries E-RAB 5 at QCI 7/9; 196 UEs had two bearers at once (the other E-RAB 6, QCI 5, IMS) and 4 had three (plus QCI 1, voice). 5G has no fixed id like E-RAB 5, so what follows is about 5G.

Before 2026-10-14 the index was append-only: the first session's entry stayed, and a lookup compared keys without checking the node, so both were found -- a side effect of never deleting, not a design. The new index (6.6.261013.6 / 7.6.261013.5 on) has two rules that would each have dropped the first session: an entry whose key "moved to another session" is deleted, and a lookup hands out a node only after checking that the node still carries that key (the condition that lets every core read without a lock). So the node keeps one "previous" set (`teid_prev`, `pdn_ip_prev`): when the second session is set up the earlier TEID and address move there, their index entries stay, a lookup accepts either, and a release or eviction clears both; only a third session pushes the oldest out.

**Two 5G problems and the fix (engine 2.11.24 / 2.9.27, not yet released).** A node has one set of tunnels, and the data path learns the UE's address, renews the ARP entries and keeps the idle time from the "current" uplink alone. Before the fix two cases cost the internet session its breakout; both were reproduced on a box:

- **Internet first, IMS later** (what a VoNR-capable phone does right after registering): IMS became the current session, and every PDUSessionResourceSetupResponse overwrote the downlink tunnel, so what the edge sent back to the internet address went down the IMS session's gNB tunnel. "A second response within 5 s does not overwrite it", as this section said before, holds for InitialContextSetupResponse only.
- **A UE coming back from idle**: the InitialContextSetupRequest brings all its sessions back at once, and only the last of the list was recorded. With IMS last, the internet session was not in the table at all and its return traffic was dropped.

With the fix the node remembers which PDU session id it follows (one byte more a node). A new node follows the last session of the list that is not IMS; IMS is told by a QoS flow of 5QI 5 (GSMA NG.114), since the DNN is in the ciphered NAS. An IMS session does not take the place of one that is not; every other session's uplink TEID is kept as "previous", so its uplink still finds the node and its cell; and a response takes only the followed session's downlink tunnel. A single session, two sessions neither of them IMS, and IMS set up first all end as before. 4G and the handovers are untouched.

Verified 2026-10-19 with `tools/test/mec/pdu2.py` (seven scenarios, one UE each):

| Test | Before | After |
|---|---|---|
| G8S-MEC through its ports: the table, plus the downlink GTP captured on the base-station side | the internet return went down the wrong tunnel in 1 scenario and was dropped in 1; 2 more lost the second session | 7/7; every downlink wrapped in the followed session's gNB tunnel |
| The same box under load: 30,000 single-session and 2,100 multi-session UEs, then 15,000 released | -- | 2,100/2,100 right; the three index counts as computed, each 15,000 lower after the release; 0 drops, the engine never restarted |
| HL1, replayed from a LOOP port | as above | 7/7; 50 copies, 350 UEs, 350/350 |

Still open: a node made by a handover (Xn / N2) does not know which session it follows and behaves as before the fix; a handover message carrying several sessions still leaves the staging table with the last one.

## Breakout at GRISM, compared with an edge UPF

3GPP's way is a UPF at the edge: in 5G an UL-CL / branching point steered by the SMF's N4 rules, in 4G a local S-GW/P-GW or LIPA/SIPTO. GRISM's way is an inline packet-processing box between the base stations and the core that learns each UE's cell and tunnels from the signalling and, by configuration, decapsulates chosen traffic and hands it to the edge. How they differ:

| | Edge UPF | GRISM MEC breakout |
|---|---|---|
| What the core must do | SMF/PCF support for UL-CL, DNAI and steering rules, and a new UPF on N4 -- usually an integration project with the operator | **nothing**: no N4, no SMF that knows about it, any vendor's core |
| 4G / 5G / NSA | different mechanisms per generation (local UPF vs. local S/P-GW), and NSA depends on where the anchor is | one box speaks S1AP and NGAP; one configuration for 4G, 5G and NSA |
| Steering criteria | DNN, slice and UE address, with cell-level steering needing the PCF and location reports | packet-level conditions combined freely: cell, base station, UE address range, destination, ANDed or ORed with any other filter (L4–L7, IMSI, ...) |
| Placement and latency | needs a UPF site; traffic is GTP-processed and routed again inside it | the GTP comes off at the aggregation point and the packet is delivered at L2 to the ARP-resolved edge MAC: one hop fewer |
| On failure | a UPF outage is a service outage | an inline box can fall back to its bypass relays (G8S, T12S and G8 have them), or the chain's default leg keeps the core path |
| Visibility | separate probes | the same box gives per-UE-session uplink/downlink packets and bytes (`get_s1ap_items`), filter counters, live capture and syslog |
| Time to deploy | weeks to months | this whole bed is a forty-line XML |

The honest limits: broken-out traffic never reaches the core, so the core's charging and lawful intercept do not see it (usually the point in an enterprise or campus network, something to agree on with an operator); GRISM has to see both planes -- N2 and N3, S1-MME and S1-U -- on the same box; and a UE moving between base stations is followed by GRISM's path-switch and handover tables rather than by a standard session anchor. The UE's address is learned from the user plane and the cell from NGAP/S1AP IEs, so NAS ciphering makes no difference.

## Operating notes

- The whole bed is systemd units: wiring (oneshot) → `open5gs-amfd/upfd` (drop-ins make them wait for the wiring, or they cannot bind 10.5.0.2) → `ueransim-gnb@A/B/C` → `ueransim-ue@1/2/3`, `srsenb` → `srsue` → a traffic generator. UERANSIM's gNB tries the AMF once at start and never again, so a timer that restarts any gNB without an NG Setup after a minute keeps the bed self-healing; srsUE has to start after the eNB.
- `journalctl -u 'ueransim-*' -f` shows registrations and PDU sessions; `get_s1ap_items` what GRISM has learned; the traffic page the P3/P4/P5 counters.
- Switching policy is just another run.xml (`submitxml`): the engine reloads in about a second and the UE sessions are untouched.
- Handover can only be tested with synthesised signalling (`mecho.py xn|n2 --verify`, above); filling the table is `mecfill.py` (InitialUEMessage + PDU session setup + one G-PDU, thousands of sessions a second). Both need only the `ran` namespace and the blackhole routes.
