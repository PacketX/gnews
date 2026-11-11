# GRISM ARM64 line (7.x) release notes

The ARM64 models: Q16, F3T1G4, T4G12, H2. The MIPS64 models' notes are [here](/en/release-mips.md).

## GRISM-7.6.261111.1
\- function added \-
- traffic by country can count the remote end alone: give the ports a WAN/LAN side (Settings → Interfaces) or list the local networks (Settings → Packet handling → Flow tracking) and each flow counts once, under the country of its remote end, with inbound / outbound and flow counts; with nothing set it counts both ends as before
- the country statistics can be cleared on their own, leaving the other counters alone
- a factory reset can keep the management interfaces' addresses (Settings → Backup & restore, on by default)
- the settings page explains what each of the four account levels may do

\- bug fixed \-
- fix a configuration parse failure being logged at every boot after a factory reset
- fix one failed query of the packet engine being logged at boot
- improve the memory check for very large IPv6 / CIDR lists: one that would not fit is refused and the list in use kept, instead of affecting the system

\- item changed \-
- the raw configuration XML lists each port's WAN/LAN side and the local networks
- system account and SSH component permissions tightened, against a known Linux kernel weakness


## GRISM-7.6.261104.2
\- function added \-
- live loading of big filter lists is on by default (Settings → Packet handling → Live filter updates), and IPv4 CIDR and IPv6 lists load live too instead of through a full reload; a filter taken out of a list file is removed with the file's next load
- a new alert, Filter list not applied: raised when a list fails to load, cleared by its next successful load (Settings → Alerts)
- the filters and chains pages list the filters the device runs that this configuration does not define (live-loaded lists and the like), and a chain can reference them

\- bug fixed \-
- improve the stability and memory use of loading and updating big filter lists; one that does not fit keeps the list in use, and the system log says so
- fix the GRISM ports' SNMP ifIndex shifting after a management interface re-registers
- fix the SNMP sub-agent exiting on a query that takes too long

\- item changed \-
- system components updated: OpenSSL 3.5.9, nginx 1.30.6, Python 3.11.13, OpenSSH 10.6p1, installed by every firmware update (before, only by a first install); the update file is bigger and the update takes longer
- each box uses SSH host keys of its own (the factory image shared one set): the first SSH connection after the upgrade warns that the host key changed
- the management interfaces no longer accept TCP packets carrying IP options, against a known Linux kernel weakness
- the GRISM ports' SNMP ifIndex numbers change once at the upgrade; set grismPortIfIndexBase in snmpd.conf where they must stay fixed


## GRISM-7.6.261028.1
\- function added \-
- an automation interface: API tokens, configuration validation and path tracing, a submit that rolls back unless confirmed in time, a device capability query, an event feed and webhooks (Settings → API tokens / Webhooks)
- a replay input takes a speed ratio, playing a capture at a multiple of its own pace

\- MEC \-
- the table follows the UE's internet bearer / PDU session: on 4G it is learned when the internet bearer is not E-RAB 5, and IMS and voice bearers are no longer taken for it; on 5G an IMS session (by 5QI) is passed over
- 5G PDU session modify (PDUSessionResourceModify and ModifyIndication) is parsed: when a session's tunnel moves, breakout follows the new one
- a UE with several PDU sessions keeps its internet session's tunnels and cell across a handover (Xn / N2)
- E-RAB release and PDU session release are parsed: when the internet bearer / session is released its entry goes with it, and one set up again uses its new tunnel
- SCTP fragments are reassembled per association, stream and sequence number: base stations sending fragmented messages with the same sequence number at once no longer disturb each other
- handover is more reliable under load: a request and its acknowledge arriving together are no longer lost
- fix s1apItemsAllowIp stopping the packet engine when set to one address, and /0 matching only the network address
- verified by replaying live signalling of two operator networks (about 3 hours each); 4G S1 handover verified; a T12S-MEC takes about 1 million S1AP packets a second without loss; 100,000 sessions set up and released over and over under uplink traffic, the table and its indexes reclaimed exactly


## GRISM-7.6.261016.1
\- function added \-
- Allow Local Login Hosts: while TACACS+ / RADIUS is on, sign-ins from the listed addresses use the local accounts (Settings → Login authentication)
- System log shows the older log files, and all of them can be downloaded as one zip
- an alert on the MEC table's fill, raised at 95% (Settings → Alerts)

\- bug fixed \-
- fix a regex filter with syntax the engine does not take affecting other filters, or stopping the packet engine
- fix clearing a management interface's gateway not taking effect
- fix the watcher exiting when a bypass filter is set
- fix traffic counters occasionally under-counting on several cores
- fix the packet engine stopping when the packet buffer pool ran dry
- fix a port's outDrops count and its source IP setting

\- item changed \-
- the traffic statistics API needs a sign-in
- the system log records the address a request really came from
- in verbose mode the system log follows packet capture and ARP sending (for debugging)

\- MEC \-
- the table's three indexes (uplink TEID, UE address, AMF/MME UE id) are buckets that delete: a session that is released, evicted or idle-cleared takes its index entries with it, a lookup costs the same at any fill, and a nearly full table no longer stalls packet processing (it used to cost 2.8 µs a packet above 90% and fall to a few hundred packets a second near full)
- the UE-address and TEID indexes are at least twice the table; a smaller setting is raised automatically
- a better bucket hash for the table's nodes: UEs numbered in step by the AMF and a base station no longer land in one bucket (1000 of them used to keep 100)
- the cell and UE-id mapping carries the base station's address: several base stations each numbering from 1 no longer overwrite each other
- handover (Xn path switch and N2) keeps the base station's address and the cell; 5G now parses HandoverNotify and the location in PathSwitchRequest; a 5G path switch no longer leaves cell 0
- a UE's second bearer / PDU session keeps its tunnel and address alongside the first
- new setting nrCellIdentityBits: how many high bits of the NR Cell Identity make the 5G cell id, 26 by default (Settings → MEC)
- when a downlink GTP wrap finds no session for its destination, the system log says how many and which addresses, once a minute
- with s1apItemsUpdateLog on and a syslog target of type mec, every change of the table is sent (for debugging)
- verified with 900 UEs, ~400 Mbps and the table at 90%: handover, SCTP-fragmented signalling and heavy session churn


## GRISM-7.6.261008.1
\- function added \-
- alerts on CPU, memory, disks, temperatures, fans, services and the packet engine (Settings → Alerts)
- alerts can be sent as SNMP traps and to syslog
- four account levels: view, operator, config and admin (Settings → Login authentication)

\- bug fixed \-
- fix SNMP traps not being sent

\- item changed \-
- OpenSSH updated to 10.6p1
- Settings shows the login authentication in use and when settings are being applied
- the SNMP system notification service can be switched in Settings → Services, off by default
- System status is lighter to read


## GRISM-7.6.261007.5
\- function added \-
- the MEC table shows each UE's uplink and downlink traffic, and sorts by clicking a column
- the management services can be limited to allowed IPs (Settings → Services)
- System status shows the main current settings
- the system log can be cleared
- replayPcap plays nanosecond-resolution pcaps

\- bug fixed \-
- fix occasional JA4 calculation errors
- fix a packet replay issue
- fix services not restarting after an update on some models
- improve the stability of uploading large pcap files

\- item changed \-
- a pcap upload checks the free space first and shows its progress
- replayPcap logs which file it is playing
- display improvements on the export page and the MEC table


## GRISM-7.6.261005.4
\- function added \-
- TLS filters and JA4 for QUIC connections (Settings → packet processing, off by default)
- traffic-gen can run for a set duration
- replayPcap can pick several files at once

\- bug fixed \-
- fix JA4 calculation issues
- improve the stability of loading and updating big filter lists
- fix some settings not taking effect after being changed
- fix a counter-clearing issue

\- item changed \-
- improve how ports are shown and picked
- the flow table size settings show how many sessions they hold
- usability improvements on the export and packet-processing pages


## GRISM-7.6.261001.2
\- bug fixed \-
- fix the firmware update overlay reporting "complete" about a second after it was submitted, skipping the rebooting and back-online phases; most visible on the MIPS line
- fix grism_watcher sometimes not restarting after an online update. That service is what applies firmware updates, so losing it meant every later update silently did nothing, with no error shown; only a reboot recovered it. The restart is now verified and retried, and a failure is written to the log
- fix core dumps being pruned only when the web UI listed them; they are pruned at startup as well now
- fix the overview's chain diagram stacking its output ports on top of each other
- fix a long ingress port list overflowing its node on the chain canvas
- fix several port pickers not showing the port descriptions (deduplication, SD-WAN, heartbeat, log exporters, filter conditions, simulator)
- tidy the filters page layout when no filter is defined

\- item changed \-
- the session TCP/UDP port tables now say they count ports below 1024 only
- the export page keeps 10 pre-submit snapshots instead of 5


## GRISM-7.6.260930.4
\- function added \-
- new web interface, GRISM Studio: edit the packet pipeline visually, simulate a packet's path before submitting, live traffic and system status, configuration diff and rollback; English/Chinese, light/dark
- packet capture now shows decoded packets as they are recorded
- crash debugging: core dumps can be collected and downloaded from the web UI, and the packet engine can be restarted from there too
- a firmware update now restarts the affected services instead of rebooting
- new TLS JA4/JA4S filter fields
- service management: report each service's version, start and stop services
- front switch (Q16) status and restart
- system status now lists hugepages and the flow tables separately as reserved memory, and computes usage with them subtracted
- sshd and the country database can be updated on their own, without a full firmware update
- statistics counters and sessions are kept across a configuration reload
- web idle logout raised from 10 to 30 minutes

\- bug fixed \-
- fix inner packets of VXLAN/GRE tunnels not being parsed on non-correlation ports, which silently disabled filters written against inner fields
- fix inner packets not being parsed on a LOOP ingress
- fix the packet engine possibly dying after a configuration submit
- fix the timezone used for times shown in the web UI
- fix syslog filters not accepting the new expression form
- fix the TLS version read for JA4S
- fix login account privilege handling
- fix an update check failing without saying why
- remove the sample mapping rows from the shipped configuration
