# A 5G test bed with UERANSIM and Open5GS, and GRISM's MEC breakout under test

## Goal

Run a 5G core (Open5GS) and a base-station/handset simulator (UERANSIM) on one Linux VM, with N2 (NGAP over SCTP) and N3 (GTP-U) **really crossing a wire** through a GRISM inline chain, then add an edge host and verify MEC breakout: when a UE in a chosen cell reaches the edge service, GRISM strips the GTP and hands the packet to the edge directly, wrapping the reply back towards the gNB; everything else goes on to the core, which never sees the broken-out part.

The machines: a GRISM G8S-MEC (P4 on the gNB side, P5 on the core side, P3 to the edge host) and an Ubuntu 24.04 VM (8 cores, 16 GB, four NICs).

## Layout

```
VM                                                        GRISM G8S-MEC
┌─────────────────────────────────┐
│ [ran namespace]                 │
│   ens19  10.5.0.1  gNB A        │───── P4 ──┐
│          10.5.0.11 gNB B        │           │  inline chain
│          10.5.0.12 gNB C        │           │  P4 <-> P5
│          UE1 UE2 UE3 (uesimtunN)│           │
│                                 │           │
│ [host]                          │           │
│   ens20  10.5.0.2  AMF(N2) UPF(N3)│───── P5 ──┘
│   ogstun 10.45.0.1/16 ─ NAT ─ ens18 ─ Internet
│                                 │
│ [edge namespace]                │
│   ens21  10.45.255.254  HTTP    │───── P3 (edge: where breakout leaves)
└─────────────────────────────────┘
```

| Interface | namespace | Address | Role | GRISM port |
|---|---|---|---|---|
| ens19 | `ran` | 10.5.0.1 / .11 / .12 | N2/N3 addresses of the three gNBs; the UEs live here too | P4 |
| ens20 | host | 10.5.0.2 | Open5GS AMF (NGAP) and UPF (GTP-U) | P5 |
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

## Operating notes

- The whole bed is systemd units: wiring (oneshot) → `open5gs-amfd/upfd` (drop-ins make them wait for the wiring, or they cannot bind 10.5.0.2) → `ueransim-gnb@A/B/C` → `ueransim-ue@1/2/3` → a traffic generator. UERANSIM's gNB tries the AMF once at start and never again, so a timer that restarts any gNB without an NG Setup after a minute keeps the bed self-healing.
- `journalctl -u 'ueransim-*' -f` shows registrations and PDU sessions; `get_s1ap_items` what GRISM has learned; the traffic page the P3/P4/P5 counters.
- Switching policy is just another run.xml (`submitxml`): the engine reloads in about a second and the UE sessions are untouched.
