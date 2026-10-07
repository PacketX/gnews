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


## GRISM-6.6.261007.5
\- function added \-
- the MEC table shows each UE's uplink and downlink traffic, and sorts by clicking a column
- the management services can be limited to allowed IPs (Settings → Services)
- System status shows the main current settings
- the system log can be cleared
- replayPcap plays nanosecond-resolution pcaps

\- bug fixed \-
- fix occasional JA4 calculation errors
- fix a packet replay issue
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


## GRISM-6.6.261005.6
\- function added \-
- TLS filters and JA4 for QUIC connections (Settings → packet processing, off by default)
- traffic-gen can run for a set duration
- replayPcap can pick several files at once

\- bug fixed \-
- fix JA4 calculation issues
- improve the stability of loading and updating big filter lists
- fix some settings not taking effect after being changed
- fix a packet replay issue

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

## GRISM-6.6.261001.2
\- bug fixed \-
- fix the firmware update overlay reporting "complete" about a second after it was submitted, skipping the rebooting and back-online phases; most visible on the MIPS line
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

## GRISM-6.6.260930.4
\- function added \-
- new web interface, GRISM Studio: edit the packet pipeline visually, simulate a packet's path before submitting, live traffic and system status, configuration diff and rollback; English/Chinese, light/dark
- packet capture now shows decoded packets as they are recorded
- crash debugging: core dumps can be collected and downloaded from the web UI for analysis
- filters accept nested expressions (and/or/not, several levels deep)
- new TLS JA4/JA4S filter fields
- new S1AP/NGAP cellid support
- service management: report each service's version, start and stop services
- sshd and the country database can be updated on their own, without a full firmware update
- statistics counters and sessions are kept across a configuration reload
- web idle logout raised from 10 to 30 minutes

\- bug fixed \-
- fix inner packets of VXLAN/GRE tunnels not being parsed on non-correlation ports, which silently disabled filters written against inner fields
- fix inner packets not being parsed on a LOOP ingress
- fix the packet engine possibly dying after a configuration submit
- fix the wrong VLAN counter being used for a QinQ output
- fix filter "and" condition evaluation
- fix syslog filters not accepting the new expression form
- fix login account privilege handling
- fix an update check failing without saying why

## GRISM-6.5.260715
\- function added \-
- more management interface settings
- add get dns qry name resp table dev api support

\- bug fixed \-
- fix config set issue
- fix country iso counter issue
- fix filter matchResult issue

## GRISM-6.5.260612
\- bug fixed \-
- fix dns name parsing issue

## GRISM-6.5.260601
\- function added \-
- add dns dynamic update parsing and syslog support
- add dns opcode, rcode, remove dtype_num fields syslog support
- add management link status filter for HA support
- add grism xml chain in/out vlan tagging/stripping attribute support
- add web service 10 minutes session timeout support
- add vxlan strip data to packet tail and tag back to packet head support
- add web GRISM XML delete run1-9 xml support

\- bug fixed \-
- remove ssh dsa type key generate for G8s seldomly boot very slow

## GRISM-6.5.260320
\- function added \-
- add management1 support

\- bug fixed \-
- fix packet duplication issue for modify tcp mss field
- fix G8s(KOB3410) hardware phy port init issue

## GRISM-6.5.260123
\- function added \-
- add hostname and lldpd support

\- bug fixed \-
- fix pcap replay MAX files
- fix get l2gre/vxlan correlation table failure
- fix l2gre and vxlan l2broadcast
- fix broadcast tunnel packet with vlan
- fix console login message repeating
- fix dns query type log

## GRISM-6.5.251230
\- function added \-
- add management mac address get
- add num of runX.gdp to 16

\- bug fixed \-
- fix decrypt function
- fix traffic-gen function
- fix sshd session link path error on first install

## GRISM-6.5.251107
\- bug fixed \-
- fix L3/L4 IP/TCP/UDP packet error check handle

## GRISM-6.5.251029
\- function added \-
- upgrade sshd to 10.2p1 for security issue
- update dbip country mmdb to 2025-10

## GRISM-6.4.250911
\- bug fixed \-
- fix www service instability

## GRISM-6.4.250731
\- function added \-
- add netflow counter and eps support
- add interface ifidx support for netflow and statistics

## GRISM-6.3.250714
\- function added \-
- simplify XML save load function
- enhance GRISM-A heartbeat detect and switch XML config
- add netflow packet length statistics choice to use ip header length

\- bug fixed \-
- fix google chrome (version 138) view issue

## GRISM-6.2.250502
\- function added \-
- GRISM-A heartbeat submit function support
- add gtpc log ipv6 prefix support 

\- bug fixed \-
- fix vxlan change client port issue
- fix web service security issue
- fix G8s bypass init issue

## GRISM-6.1.250327
\- function added \-
- add ja4 ja4s support
- add matched syslog blocked=yes/no string
- fix security issues

## GRISM-5.5.250317
\- function added \-
- upgrade sshd to 9.9p2

## GRISM-6.0.250310
\- function added \-
- upgrade sshd to 9.9p2
- add DNS over TCP support
- add dns client query per second (QPS) syslog support

## GRISM-5.10.250224
\- function added \-
- add syslog, dpilog, netflow and interface description setting without interrupt system
- add tacacs+ admin(15)/guest(1) priv support
- add more GTP-C syslog support
- add GRISM monitor other GRISM port support
- add hide runX.xml from UI support
- add encapsulation Encrypt Key timeout support
- add flowv6 filter support

\- bug fixed \-
- fix gtp-c v1,v2 parsing issue
- fix grism xml memory usage for hash table issue
- fix find type issue if unsupport type parsed

## GRISM-5.5.250206
\- function added \-
- upgrade sshd to 9.8p1
- fix security issues

## GRISM-5.9.250114
\- function added \-
- add G8 hardware iptables support
- update dbip country mmdb to 2024-12
- add netflow v9 gtp/gre/vxlan tunnel outer ip

\- bug fixed \-
- fix RAN-UE-NGAP-ID integer range issue
- fix NGAP transportLayerAddress ipv6 parsing issue

## GRISM-5.5.250114
\- bug fixed \-
- fix RAN-UE-NGAP-ID integer range issue
- fix NGAP transportLayerAddress ipv6 parsing issue

## GRISM-5.9.241104
\- function added \-
- add more DPI function support
- add syslogd more rotated logs to keep
- add global time zone setting support

\- bug fixed \-
- icmp heartbeat vlan support
- fix arp table recording
- fix www service issue

## GRISM-5.8.241004
\- function added \-
- nginx configuration enhancement
- set xmlrpc service default localhost for security issue

\- bug fixed \-
- fix 25G/10G link status issue for Q4T4

## GRISM-5.8.240730
\- function added \-
- add Country Statistics support
- upgrade openssh to 9.8p1
- add ssl dpi ja3s syslog support
- add dns query name filter default case insensitive support
- add tcp segments reassembled support for only ssl protocol and two segments maximum
- add G8S/T12S/T12/T20(F2T12,F4T4,Q4T4) iptables support for icmp type timestamp request/reply block (linux kernel reinstall needed)
- add web console DEV API page support
- add web console Flow page hardware bypass/heartbeat port status support

## GRISM-5.7.240603
\- function added \-
- add model Q4T4 build support

\- bug fixed \-
- fix modify_tcp_syn_mss issue
- fix gre correlation tunnel issue

## GRISM-5.7.240529
\- function added \-
- vxlan breakout enhancement for working with MEC [MEC+VXLAN Breakout](https://packetx.gitbook.io/grism-xml/docs/mec+vxlan-breakout)
- add more heartbeat bypass detail logs

\- bug fixed \-
- fix netflow v10 uptime issue
- fix vxlan/gre correlation tunnel issue

## GRISM-5.6.240423
\- function added \-
- deduplication specific ports (include LOOP ports)
- adjust KOB3410 (G8S) fan default speed to 50%

## GRISM-5.6.240418
\- function added \-
- upgrade sshd to 9.7p1
- remove ChaCha20-Poly1305 algorithm from ssh

## GRISM-5.5.240418
\- function added \-
- upgrade sshd to 9.7p1
- remove ChaCha20-Poly1305 algorithm from ssh

## GRISM-5.6.240412
\- function added \-
- add modify source/destination port
- add output type TCP Reset support
- add T12S hardware(KOB3400) smart fan control support
- more general using for icmp heartbeat function

## GRISM-5.6.240320
\- function added \-
- add Statistics -> Service support
- add T12S/G8S Configuration -> Interface port mapping device picture support
- add [detect if destination IP is NOT in the DNS Response IP Table](https://packetx.gitbook.io/grism-xml/docs/detect-if-dstination-ip-not-in-dns-response-ip-table)

## GRISM-5.6.240201
\- function added \-
- add online update version check, download and upgrade

## GRISM-5.5.240122
\- function added \-
- nginx configuration enhancement

## GRISM-5.5.240108
\- function added \-
- add [NAT](https://packetx.gitbook.io/grism-xml/docs/l3nat_breakout) support
- add [mec.js](https://packetx.gitbook.io/grism-xml/reference/mec.js) support to simplify GRISM XML for MEC
- add ICMP reply support
- add ICMP type, code filter
- add [action arp reply / icmp reply / icmp reply fragment need](https://packetx.gitbook.io/grism-xml/readme/action#less-than-arp_reply_default_mac-greater-than) tag support
- add arp_srcip output support for arp dest mac
- modify Country Codes filter add block if empty parameter
- add Status -> Process core process status display
- add syslog alert if core process crash

## GRISM-5.4.231214
\- function added \-
- add dns.a filter matched log dns_query_name field
- add nvme driver for IM8724 model to add NVME ssd support
- change product logo
- add Hard disk write throughput display at Flow page
- enhance snort rule to filter gadget

\- bug fixed \-
- more stable for extreme s1ap streamming test
- more stable for write pcap files
- fix T12s bypass interface crc error happen occasionally
- fix delete VPorts issue

## GRISM-5.4.231020
\- function added \-
- l2gre, vxlan mapping table filter support
- enhance replay pcap speed and more stable
- ICMP heartbeat support

\- bug fixed \-
- SCTP reconstruct function crash at CN73xx in one day and CN78xx in two weeks
- fix System->Status Time Zone issue
- fixed www add user allow duplicate

## GRISM-5.3.230918
\- bug fixed \-
- fix System->Status Time Zone issue

## GRISM-5.4.230908
\- bug fixed \-
- CPU issue when non-MEC device try cron to clear s1ap table items (defult every monday 02:00 am)

## GRISM-5.4.230825
\- function added \-
- add pcap file upload for pcap replay function support
- add pcapng format and Linux cooked captured v1 layer for pcap replay function support
- add syslog daemon default enable support
- add G8S Bypass WDT function support
- add clear filter counter for clear lite counter api support

## GRISM-5.3.230824
\- function added \-
- restore password after upgrade firmware

\- bug fixed \-
- fix SCTP Reconstruct memory issue

## GRISM-5.3.230814
\- function added \-
- ssh upgrade from 8.6p1 to 9.4p1
- nginx configuration enhancement

## GRISM-5.3.230804
\- function added \-
- add snmp get Fan Fault and Temperature overheat
- disable system status syslog alert function which default enable since 5.2 but now is useless

## GRISM-5.3.230719
\- function added \-
- add [filter](https://packetx.gitbook.io/grism-xml/readme/filter) start, position, within and mpslog parameters support
- add http.host and http.request.uri filter support
- add ssl.ja3s_digest filter support
- add MEC cron to clear s1ap table items which idle time over max
- add matched per second support for counter and syslog alert
- add MEC Mapping <=, > filter and clear idle data support

## GRISM-5.2.230615
\- function added \-
- MEC mec.mapping.ue.ipv4.connected filter support for trigger core paging issue
- System Status Temperature/Fan support
- Syslog alert boot/Link status/Temperature/Fan/CPU usage/ support

\- bug fixed \-
- MEC CELL-ID field parsing issue

## GRISM-5.1.230517
\- function added \-
- add PLMN ID, CELL ID, SPID columns to MEC Mapping
- add MEC gtp.data.by.s1ap.CellIdentity, gtp.data.by.s1ap.SubscriberProfileIDforRFP filters
- add In-Tunnel L2MPLS/MPLS in UDP/GRE
- add output/action striping mpls-in-gre xml support
- add output tagging vxlan xml support [VXLAN breakout](https://packetx.gitbook.io/grism-xml/docs/vxlan-breakout)

## GRISM-5.0.230330
\- function added \-
- 5.0 and later drop CN68XX CPU support (models T2G8, T16, T32)
- add Storage page: mounted filesystems and mount settings
- mount usb file systems automatically
- add G8S model support, 8x1G with two bypass pairs
- add source mac, dest mac support to traffic-gen
- add output dns_response_ipv6 xml support
- Memory Usage now reports available instead of free
- add SOA and TXT formats to the dns log
- https service now goes through the nginx proxy
- replace python2 with python3 for performance and stability
- add SCTP DATA chunk packet reassembly
- dnslog/httplog/ssllog client/server ipv6 support

\- bug fixed \-
- fix input bytes accounting error on CN71XX

## GRISM-4.8.230103
\- function added \-
- add T12S Hardware model support
  - 12 10G ports, includes two pairs of 10G Hardware Bypass modules
- add \<output\>\<tagging\> l2gre support, see [L2 GRE Breakout](https://packetx.gitbook.io/grism-xml/docs/l2-gre-breakout)
- add Help -> User Manual [GRISM XML](https://packetx.gitbook.io/grism-xml)
- Pcap Replay/Traffic Generate/Packet Snapshot pages offer a stop function and the GRISM XML they generate
- add filter -> find arp.request.sender.ip

## GRISM-4.7.221122
\- enhancement \-
* more suitable hash table collision method for MEC s1ap/ngap record mapping

## GRISM-4.7.221116
\- bug fixed \-
* fix MEC Handover issue 

## GRISM-4.6.221101
\- function added \-
* add output xml nvgre_sip, nvgre_dmac, nvgre_type, so a GRE Tunnel can be sent without an interface source IP; type may be eth or ip

\- bug fixed \-
* Web Console no longer preloads run1-run9 xml and js, which failed to load when large

## GRISM-4.5.221022
\- function added \-
* Web Console supports editing multiple GRISM XML files: run.xml, run1.xml ~ run9.xml and js files
* add Web Console System->Log page
* add category parameter to output dir xml, filing pcap files by time; Packet Snapshot and Pcap Replay support one more directory level

\- bug fixed \-
* fix heartbeat id failing when set without a description
* fix IP Fragmentation Correlation and Reconstruct when packets arrive out of order


## GRISM-4.3.221004
\- function added \-
* add GRISM XML ```<script></script>``` support
```xml
<script src="common.js"></script>
<script>
<![CDATA[
    port_mirror('P0', 'P1,P2');
]]>
</script>
```
* add Packet Data builder tool
* add Packet Data setting to traffic-gen

\- bug fixed \-
* rework MEC S1AP/NGAP items table resource release for frequent UE attach/detach on a shared large network
* snmpd temporary files moved to ramdisk, in case the filesystem is read-only after an abnormal reboot


## GRISM-4.2.220826
\- function added \-
* MEC Mapping change to packet base
* MEC NGAP Handover support
* MEC S1AP/NGAP parsing items syslog support
* MEC Server to UE more smooth
* traffic-gen xml \<msinterval\> tag and ICMP protocol support

\- bug fixed \-
* fix MEC S1AP/NGAP items table timeout release without lock

## GRISM-3.13.220718
\- function added \-
* add Pcap Replay page
* add Traffic Generate page

## GRISM-3.12.220711
\- bug fixed \-
* fix SCTP DATA chunk padding size miscalculation

## GRISM-3.11.220628
\- bug fixed \-
* fix some flows never timing out since GRISM-3.7.220527

## GRISM-3.10.220623
\- function added \-
* add output icmp_reply_fragment_need support [<icmp_reply_fragment_need/>](https://packetx.gitbook.io/grism-xml/readme/output#less-than-icmp_reply_fragment_need-greater-than)

\- bug fixed \-
* www supports sending syslog to more than one server


## GRISM-3.9.220621
\- function added \-
* Syslog type:system send more details about setting ifcfgs, interface enable/disable, tacacs+, netflow, etc.
* TACACS+ falls back to local login when it cannot be reached
* add resolve name server setting
* add filter find ip.flags.df and ip.flags.mf
```xml
<filter id="1000" alt="test" sessionBase="no">
    <or>
        <find name="ip.flags.df" relation="==" content="1"/>
        <find name="ip.flags.mf" relation="==" content="0"/>
    </or>
</filter>
```
* add filter find packet.len, and the >= , <= relations (supported on some filters only, such as tcp.port, udp.port, packet.len)
```xml
<filter id="1" sessionBase="no">
    <and>
        <find name="packet.len" relation="&gt;=" content="128"/>
        <find name="packet.len" relation="&lt;=" content="512"/>
    </and>
</filter>
```

## GRISM-3.8.220602
\- function added \-
* rework account management
  *  the current account list can be enumerated
  *  deleting needs no password (they are all admin)
  *  the default [packetx] account is protected and cannot be deleted
* add output packet modification with the default mac, for [L3 breakout](https://packetx.gitbook.io/grism-xml/docs/l3nat_breakout)
```xml
<output id="3">
    <port>P5</port>
    <arp_reply_default_mac/>
</output>
<output id="5">
    <port>P5</port>
    <modify_src_default_mac/>
</output>
```

## GRISM-3.7.220527
\- function added \-
* add output minbps, maxbps bandwidth limiting
```xml
<output id="8" minbps="200000000" maxbps="500000000">
    <port>P8</port>
</output>
```

## GRISM-3.6.220525
\- function added \-
* add / delete user
* add Port Enable/Disable (takes effect after a reboot)
* add Snapshot Refresh, and expose the Storage and Dir fields of the save path
* management interface IP and time server take effect without a reboot

\- bug fixed \-
* firmware update locks the screen while the file uploads
* fix clear counter on the Counter page not clearing the Filter Matched Counter

## GRISM-3.4.220513
\- function added \-
* add get statistics json uptime second (uptime_s)
* flow/flowv6 enable/disable and size changes no longer need a reboot
* GRISM XML `<chain/><in/>` may be followed directly by `<next/>`, with no `<fid/>` first

old
```xml
<chain>
    <in>P0</in>
    <fid>F1</fid>
    <next>
        <out>P1</out>
    </next>   
</chain>
```
new
```xml
<chain>
    <in>P0</in>
    <next>
        <out>P1</out>
    </next>   
</chain>
```

## GRISM-3.4.220428
\- bug fixed \-
* fix MEC handover issue

## GRISM-3.3.220422
\- bug fixed \-
* fix snmp not returning VPORT traffic
* fix snmp not returning Flow Counter

## GRISM-3.3.220420
\- function added \-
* add Backup, Restore from file and Factory reset
* add filter tuple5_live_hashtable_size, so 5-tuple conditions can be added live over syslog or xmlrpc
* add L2 Switch like function: store source mac in the mac table, look up dest mac to find the output interface

\- bug fixed \-
* too many VPORTs caused problems; all interfaces together are now limited to 63
* fix lite clear counter briefly clearing the link status

## GRISM-3.2.220309
\- function added \-
* add Statistic Counter Protocol/TCP/UDP concurrent bytes
* add Service enable/disable page

\- bug fixed \-
* add heartbeat id setting and filtering, so deleting one no longer confuses the filters

## 2022-01-13 (3.1)
\- function added \-
* add web console v3 
* add TACACS+ 
* GRISM-A models: read/set the virtual-to-physical interface mapping and update filters dynamically
* add Heartbeat status display and description setting
* a not filter may now be used directly in a Chain (ex. \<fid\>!F1\</fid\>) 
* add DNS syslog type AAAA and reply error code (1-9) 
* add GTP-U parsing extension header 
* add GRISM Port Linkdown filter

\- bug fixed \-
* fix www service sometimes failing to start at boot

## 2021-09-16 (3.0)
\- function added \-
* version number 3.0 released
* upgrade sshd to OpenSSH_8.6p1, fixing older vulnerabilities
* use a more stable httpd(www) build, ending the web console instability that needed a periodic restart
* raise GRISM xml Save/Load to 20 slots
* add GRISM Xml 2chart text-graphic view and 2gml low-level view to check a configuration

\- bug fixed \-
* netflow v9 and later bytes widened from 4 to 8 Bytes, fixing connections larger than 4G

## 2021-06-28
\- function added \-
* add ftp filtering
  * covers tcp 20/21 and the ftp-data dynamic ip/port parsed out of ftp passive mode
```xml
<filter id="1" sessionBase="no">
<or>
  <find name="ftp" relation="==" content=""/>
</or>
</filter>
```
* add filter blockifempty parameter
  * a filter with no conditions passes everything; this parameter blocks instead
```xml
<filter id="1" blockifempty="yes">
<or>
</or>
</filter>
```
* add standard GRISM BYPASS models, for example G8-BPS
* adjust the MEC Template to match real deployments

## 2021-03-22
\- function added \-
* add dns.qry.name_public_suffix filter
```xml
<filter id="10004" sessionBase="no">
      <or>
        <find name="dns.qry.name_public_suffix" relation="==" content="*.facebook.com" />
        <find name="dns.qry.name_public_suffix" relation="==" content="*.google.com" />
      </or>
</filter>
```
* add SNMP Read Community setting
* add input packet drop syslog
  * Configuration -> Syslog -> Add New Target -> type:system -> subtype:alert_dropped_packets
  * type=3(system) subtype=0(input packet drop)
```
Mar 23 14:55:16 192.168.1.124 datetime=1970-01-01 03:40:43,type=3,subtype=0,interface=P1,packets=28,Mbps=150.63,Pps=22295,flows=49493/2097152,v6flows=261003/262144,cpu_load_average=13.28;4.47;3.84,mem_usage=519892/2060344
```
* (MEC) add MEC Template
* (MEC) add S1AP Table

## 2021-02-09
\- function added \-
* add inpps, outpps (in/out packets per second) to the Flow page
* add dns.qry.type, dns.count.add_rr filter find support
* add the 1 == 1 filter find form, a condition that always or never matches, to guard against a generated blacklist coming out empty

## 2020-11-03
\- function added \-
* add VPort settings, pairing with a switch over vlan tagging + trunk port to extend the usable interfaces
* add ssl.server_name, ssl.server_name_public_suffix filters, matching the server name in the ssl certificate
* add system common syslog: web login/logout, password changes, settings and xml task submissions
* add filter counts, for both xml task filters and hidden blacklist filters
* add time zone switching, currently Taipei and None
* IP filtering also matches the outer IP when a tunnel sits above the IP layer
* models with a usb management interface (T20, F4T4, F2T12) configure it on usb hotplug

## 2020-09-17
\- performance tuning \-
* faster VPort (T16 V0-V15) duplication to several outputs and custom outputs (\<output\>)

\- function added \-
* add dns response IPv4 output, pairing with the dns.qry.name filter to answer with a chosen IPv4
* add snapshot, capturing traffic briefly into a pcap file

## 2020-07-12
\- bug fixed \-
* Record ID on the Heartbeat page starts from 0, matching the xml

\- function added \-
* add arp filters 
  * arp
  * arp.request
  * arp.reply
  * arp.request.target.ip
* add arp reply target mac output, pairing with the arp.request.target.ip filter to answer with a chosen mac

## 2020-04-20
\- function added \-
* add ssl.ja3_digest filter, matching the ssl ja3 hash
* add repeated pcap replay, \<input type="replayPcap"\> tag
* add Traffic Generator, building IP/Port and producing line-rate traffic beyond 10 Gbps, \<input type="traffic-gen"\> tag
* add guest account: no settings after login, view only
* the three features above are configured in xml, see https://packetx.github.io/gml

## 2020-04-10
\- function added \-
* restore factory defaults, Help -> Restore

## 2020-03-10
\- function added \-
* T20, F2T12 10G<->1G switching
* G8 LAN bypass
  * bypass from boot until the main program runs
  * simplified status display and settings
* simplified xml
  * chain id is now optional
  * ```<find name="" relation="" content="" />``` may be shortened to ```<f n="" r="" c=""/>```
* add flow table size setting on the Configuration XML page: flowCacheBaseSize, flowv6TableSize under args

## 2020-02-25
\- bug fixed \-
* GRISM Task->Map xml comments disappeared after Save and Load

\- function added \-
* add dns query name response ip addr filter
