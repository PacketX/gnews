# Packets per second with 64-byte frames

How many of the smallest frames can a box forward a second **without losing one**? This page is what a G8S, a T12S and a Q16 measured with the flow table on (the factory settings) and off (an optimised set) (2026-11-06 and 11-09; firmware 6.6.261104 on the G8S and T12S, 7.6.261106 on the Q16). Packets per second is the limit that small-packet traffic runs into: the same pps carries more throughput as packets grow, so the 64-byte pps times the average packet size of real traffic is the box's ceiling for that traffic (0.92 Mpps × 600 bytes ≈ 4.4 Gbit/s, say).

## Method

- **The box under test generates nothing itself.** Another GRISM's traffic-gen input sends the frames over real cables into the box under test, which forwards them through one chain (one filter, `ip.src == 11.0.0.0/8`, matching everything) to another port, cabled back to the generating box, which counts them. No LOOP ports.
- **Frames**: UDP, `packet_size` 60, which is exactly 64 bytes on the wire with the FCS (traffic-gen's `packet_size` excludes the FCS; 64 would make 68-byte frames). Each generator cycles through a contiguous range of source addresses (`src_ip_min` / `src_ip_max` / `src_ip_inc`), so every frame is a different flow.
- **Flows**: with the flow table on, the generators' source addresses together come to about **75%** of the table's capacity (`flowCacheBaseSize × 16`, times the number of work cores on the DPDK line): 480,000 on the G8S, 1.8 million on the T12S, 6.35 million on the Q16. Too few is wrong -- a single flow is held on one core by its atomic tag on OCTEON, which measures one core rather than the box (0.49 Mpps for a single flow on the G8S), and a few tens of thousands leave the table nearly empty and the figures high; too many is wrong too -- a nearly full table pulls them down (see the notes after the results). With the flow table off the flow count does not matter.
- **Rate**: the first trial is unpaced (line rate, or the generator's limit); if that drops, traffic-gen's `speed` is bisected, each trial 15 s of warm-up and a 60 s window, seven trials (about 1% resolution).
- **Pass**: over the 60 s the box under test has 0 `inDrops`, `inErrors` and `outDrops` on **every port** (a Q16's NIC drops are counted on the engine's uplink ports P0-P2, not on the V ports), and its out count equals its in count (within a tenth of a second of traffic: the in count is the hardware's, the out count a sum of per-core software counters, and they are not read at one instant). The difference between what the generating box sent and got back (inPackets + inDrops, since its own engine drops at the port what it cannot absorb) only fails a trial past 1%, as a guard against loss nobody counts (a switch, a cable).
- **Flow on** (factory) = the settings after "Reset to factory settings (management address kept)", nothing changed (MEC on for the MEC models). **Flow off** (optimised) = the flow / flowv6 tables, L7 parsing, tunnel inner parsing, SCTP / TCP / QUIC reassembly, IP fragment correlation, deduplication, MEC (s1cCorrelation and the GTP tunnel header), GRE / GTP / VXLAN correlation and coreSession all off, measured after a reboot; everything else as shipped.
- "1 in 1 out" is one generator and one chain; "2 in 2 out" two generators and two chains (the two 10G cables each used in both directions).

## Cabling

| Box under test | Generating box | In | Out |
|---|---|---|---|
| G8S (1G ports) | T12S | T12S P6 → G8S P2; 2 in 2 out adds T12S P5 → G8S P6 | G8S P3 → T12S P4; G8S P7 → T12S P7 |
| T12S (10G ports) | Q16 | Q16 V7 → T12S P1; 2 in 2 out adds Q16 V6 → T12S P0; 4 in 4 out adds Q16 V4 → T12S P2 and Q16 V5 → T12S P3 | T12S P0 → Q16 V6; T12S P1 → Q16 V7; T12S P3 → Q16 V5; T12S P2 → Q16 V4 |
| Q16 (10G cages) | T12S (two to four generator cores a port) | T12S P1 → Q16 V7; 2 in 2 out adds T12S P0 → Q16 V6; 4 in 4 out adds T12S P2 → Q16 V4 and T12S P3 → Q16 V5 | Q16 V6 → T12S P0; Q16 V7 → T12S P1; Q16 V5 → T12S P3; Q16 V4 → T12S P2 |

## Results (64 bytes; Mpps = million packets a second)

| Model | Settings | In/out | **Loss-free** | Drops begin at | Forwarded when flooded (offered) | CPU at the loss-free point |
|---|---|---|---|---|---|---|
| G8S | flow on (480k flows) | 1 in 1 out | **0.43** | 0.44 | 0.51 (1.49, 1G line rate) | 94% |
| G8S | flow on (480k flows) | 2 in 2 out | **0.40** (total) | 0.41 | 0.50 (2.97) | 90% |
| G8S | flow off | 1 in 1 out | **0.92** | 0.93 | 1.06 (1.49) | 100% |
| G8S | flow off | 2 in 2 out | **0.84** (total) | 0.85 | 1.06 (2.98) | 77% |
| T12S | flow on (1.8M flows) | 1 in 1 out | **2.97** | 3.0 | 3.12 (5.24) | 97% |
| T12S | flow on (1.8M flows) | 2 in 2 out | **2.94** (total) | 2.96 | 3.11 (9.80) | 98% |
| T12S | flow on (1.8M flows) | 4 in 4 out | **3.07** (total) | 3.10 | 3.22 (18.1) | 100% |
| T12S | flow off | 1 in 1 out | **≥ 4.34** (one generator core's limit, 0 drops) | — | — | 75% |
| T12S | flow off | 2 in 2 out | **6.19** (total) | 6.24 | 6.45 (8.67) | 89% |
| T12S | flow off | 4 in 4 out | **6.27** (total) | 6.33 | 6.34 (17.6) | 97% |
| Q16 | flow on (6.35M flows) | 1 in 1 out | **≥ 6.68** (two generator cores' limit, 0 drops) | — | — | 65% |
| Q16 | flow on (6.35M flows) | 2 in 2 out, 2 generators a port | **≥ 13.4** (the generators' limit, 0 drops) | — | — | 99% |
| Q16 | flow on (6.35M flows) | 2 in 2 out, 4 generators a port | **15.2** (total) | 15.3 | 17.8 (26.9) | 100% |
| Q16 | flow on (6.35M flows) | 4 in 4 out, 2 generators a port | **15.2** (total) | 15.3 | 17.9 (27.0) | 100% |
| Q16 | flow on (6.35M flows) | 4 in 4 out, 3 generators a port | **16.1** (total) | 16.3 | 17.7 (37.1) | 100% |
| Q16 | flow off | 1 in 1 out | **≥ 6.68** (the generators' limit, 0 drops) | — | — | 50% |
| Q16 | flow off | 2 in 2 out, 2 generators a port | **≥ 13.4** (the generators' limit, 0 drops) | — | — | 85% |
| Q16 | flow off | 2 in 2 out, 4 generators a port | **≥ 26.7** (the generators' limit, 0 drops; 90% of two 10G line rates) | — | — | 100% |
| Q16 | flow off | 4 in 4 out, 2 generators a port | **≥ 26.9** (the generators' limit, 0 drops) | — | — | 100% |
| Q16 | flow off | 4 in 4 out, 3 generators a port | **27.1** (total) | 27.2 (one frame lost in 60 s) | 34.5 (37.0) | 100% |

- 64-byte line rate is 1.488 Mpps on a 1G port and 14.88 Mpps on a 10G port.
- **G8S**: about 0.4 Mpps with the flow table on, a limit of the whole engine rather than of a port -- 2 in 2 out totals slightly less (one more port to serve); about twice that with the flow tables, L7 parsing and the rest off. Flooded at line rate it still forwards about 0.5 / 1.06 Mpps, the rest is dropped at the in port and the out port never drops.
- **T12S**: about 3 Mpps with the flow table on, the same for 1 in 1 out, 2 in 2 out and 4 in 4 out -- again the whole box's limit; 6.2-6.3 Mpps with it off, whatever the port count.
- **Q16**: 15-16 Mpps with the flow table on (2 in 2 out and 4 in 4 out alike); with it off **27 Mpps without a loss** (4 in 4 out; flooded with 37 Mpps from twelve T12S generator cores it forwards 34.5, the rest counted as inDrops on the uplink ports P0-P2). Four 10G ports carry 59.5 Mpps of 64-byte frames at line rate, so 27 Mpps is about 45% of it.
- **How full the flow table is changes the flow-on figures**, which is why this page uses 75%. The same T12S with the flow table on: 65,000 flows (3% of the table) took one generator core's 4.26 Mpps 1 in 1 out without a drop and 3.98 2 in 2 out; at 75% 2.97 / 2.94; at 87% 2.73. G8S: 65,000 flows 0.52 / 0.49, 75% 0.43 / 0.40, 90% 0.35 / 0.29.
- **A Q16's drops are counted on the engine's uplink ports.** Flooded with 26.9 Mpps with the flow table on, the V ports still showed in = out (17.8 = 17.8) and 0 inDrops; the 540 million frames dropped were counted as inDrops on P0-P2, the 50G ports between the switch and the engine. Look at P0-P2 to see whether a Q16 drops.
- The CPU column is the system status page's figure; below saturation it says how much is left, and 100% is the step before drops.

## The generators' own limits

- One traffic-gen input takes one core. A G8S generates only about 0.33 Mpps a port, so it is never the generator; one T12S core fills a 1G port and does about 3.4-4.3 Mpps on a 10G port, twelve cores (three on each of four ports) 37 Mpps together; one Q16 lcore does 4.3-5.2 Mpps on one port (all 64 bytes).
- Two generators on one port: fine on the MIPS line (the PKO queue is locked, each input on its own core); on the DPDK line only safe when the port has a tx queue per core, so a generating Q16 gets one input a port.
- What `speed` means: packets a second ≈ 131072 × speed ÷ packet_size (about 2184 × speed at 64 bytes); 0 is unpaced.
