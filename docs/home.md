## GRISM-6.3.250714
\- function added \-
- simplify XML save load function
- enhance GRISM-A heartbeat detect and switch XML config
- add netflow packet length statistics choise to use ip header length

\- bug fixed \-
- fix google chrome(version：138) view issue

## GRISM-6.2.250502
\- function added \-
- GRISM-A heartbeat submit function support
- gtpc log ipv6 prefix support 

\- bug fixed \-
- fix vxlan change client port issue
- fix web service security issue
- fix G8s bypass init issue

## GRISM-6.1.250327
\- function added \-
- add ja4 ja4s support
- add matched syslog blocked=yes/no string support
- fix security issue

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
- fix security issue

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
- bug fix for record arp table
- bug fix for www service issue

## GRISM-5.8.241004
\- function added \-
- nginx configuration enhancement
- set xmlrpc service default localhost for security issue

\- bug fixed \-
- fix 25G/10G link status issue for Q4T4

## GRISM-5.8.240730
\- function added \-
- add Country Statictics support
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
- fix gre orrelation tunnel issue

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
- add modify source/destionation port
- add output type TCP Reset support
- add T12S hardware(KOB3400) smart fan control support
- more general using for icmp heartbeat function

## GRISM-5.6.240320
\- function added \-
- add Statistics -> Service support
- add T12S/G8S Configuration -> Interface port mapping device picture support
- add [detect If Dstination IP NOT in DNS Response IP Table](https://packetx.gitbook.io/grism-xml/docs/detect-if-dstination-ip-not-in-dns-response-ip-table)

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
- add Hard disk write throuthput display at Flow page
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
- MEC Mapping 頁面 新增 PLMN ID, CELL ID, SPID 欄位
- MEC 新增 gtp.data.by.s1ap.CellIdentity, gtp.data.by.s1ap.SubscriberProfileIDforRFP 過濾條件
- In-Tunnel 新增 L2MPLS/MPLS in UDP/GRE
- 新增 output/action striping mpls-in-gre xml support
- 新增 output tagging vxlan xml support [VXLAN breakout](https://packetx.gitbook.io/grism-xml/docs/vxlan-breakout)

## GRISM-5.0.230330
\- function added \-
- 5.0 之後版本不再支援 CPU CN68XX 架構包含型號 T2G8, T16, T32 
- 新增 Storage 頁面，包含目前已掛載檔案系統以及修改掛載設定
- 自動掛載usb file system
- 新增G8S 型號支援，8x1G, 其中包含兩對 bypass
- traffic-gen 新增 source mac, dest mac 支援
- 新增 output dns_response_ipv6 xml 支援
- Memory Usage 顯示 free 改成 available 
- dns log 新增 SOA TXT 格式
- https 服務改用 nginx 代理
- 使用 python3 取代 python2，提高效能以及穩定度
- 新增 SCTP DATA chunk 封包重組功能
- dnslog/httplog/ssllog client/server ipv6 support

\- bug fixed \-
- 修正 CN71XX 型號 input bytes 流量計算誤差

## GRISM-4.8.230103
\- function added \-
- add T12S Hardware model support
  - 12 10G ports, includes two pairs of 10G Hardware Bypass modules
- 新增\<output\>\<tagging\> l2gre support, 請參考 [L2 GRE Breakout](https://packetx.gitbook.io/grism-xml/docs/l2-gre-breakout)
- 新增 Help -> User Manual [GRISM XML](https://packetx.gitbook.io/grism-xml) 手冊
- Pcap Replay/Traffic Generate/Packet Snapshot 頁面提供停止功能以及 GRISM XML 語法
- 新增 filter -> find arp.request.sender.ip 過濾條件

## GRISM-4.7.221122
\- enhancement \-
* more suitable hash table collision method for MEC s1ap/ngap record mapping

## GRISM-4.7.221116
\- bug fixed \-
* fix MEC Handover issue 

## GRISM-4.6.221101
\- function added \-
* 新增 output xml 參數 nvgre_sip, nvgre_dmac, nvgre_type 支援以方便在不指定介面來源IP的情況下也能送 GRE Tunnel，另新增 eth/ip 兩種 type 設定

\- bug fixed \-
* Web Console 調整不預先載入 run1-run9 xml 以及 js 等內容，避免因檔案內容過大造成載入異常

## GRISM-4.5.221022
\- function added \-
* Web Console 支援 GRISM XML 多檔案編輯包含 run.xml, run1.xml ~ run9.xml 以及 js 檔案
* Web Console 新增 System->Log 頁面
* ouptut dir xml 新增 category 參數以時間目錄分類 pcap 檔案 以及 Packet Snapshot, Pcap Replay 多支援一層目錄進出

\- bug fixed \-
* 修正設定 heartbeat id 如果沒有同時設定 description 會失敗的問題
* 修正 IP Fragmentation Correlation 以及 Reconstruct 在封包順序不對時可能會有問題


## GRISM-4.3.221004
\- function added \-
* GRISM XML 新增 ```<script></script>``` support
```xml
<script src="common.js"></script>
<script>
<![CDATA[
    port_mirror('P0', 'P1,P2');
]]>
</script>
```
* 新增 Packet Data 封包內容產生工具
* traffic-gen 新增 Packet Data 設定

\- bug fixed \-
* MEC S1AP/NGAP items table 資源釋放調整解決於大網共構下UE頻繁attach/detach能運作正常
* snmpd 暫存檔案改寫到ramdisk 避免檔案系統因異常重開機後發生無法寫入的問題


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
* 新增 Pcap Replay 功能操作頁面
* 新增 Traffic Generate 功能操作頁面

## GRISM-3.12.220711
\- bug fixed \-
* 修正 SCTP DATA chunk padding size 計算錯誤

## GRISM-3.11.220628
\- bug fixed \-
* 修正 GRISM-3.7.220527 之後版本部分 flow 無法 timeout 的問題

## GRISM-3.10.220623
\- function added \-
* 支援 output icmp_reply_fragment_need 功能 [<icmp_reply_fragment_need/>](https://packetx.gitbook.io/grism-xml/readme/output#less-than-icmp_reply_fragment_need-greater-than)

\- bug fixed \-
* www 支援設定送 syslog 到多個 server


## GRISM-3.9.220621
\- function added \-
* Syslog type:system send more details about setting ifcfgs, interface enable/disable, tacacs+, netflow, etc.
* TACACS+ 如無法連上則改用本機登入
* 新增 resolve name server 設定
* 新增 filter find ip.flags.df 以及 ip.flags.mf 過濾條件
```xml
<filter id="1000" alt="test" sessionBase="no">
    <or>
        <find name="ip.flags.df" relation="==" content="1"/>
        <find name="ip.flags.mf" relation="==" content="0"/>
    </or>
</filter>
```
* 新增 filter find packet.len 封包長度過濾條件 以及 >= , <= relation 參數(只支援部分如 tcp.port, udp.port, packet.len 等過濾條件)
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
* 調整帳號管理功能
  *  可以列舉目前的帳號清單
  *  刪除無需要密碼 （因為全部都是admin）
  *  保護[packetx]這個預設帳號不可以刪除
* 新增 output 使用預設 mac 修改封包功能，可搭配實作 [L3 breakout](https://packetx.gitbook.io/grism-xml/docs/l3nat_breakout) 功能 
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
* 新增 ouput 支援 minbps, maxbps 限制頻寬功能
```xml
<output id="8" minbps="200000000" maxbps="500000000">
    <port>P8</port>
</output>
```

## GRISM-3.6.220525
\- function added \-
* 新增/刪除 使用者功能
* 新增 Port Enable/Disable 功能 (需重開機生效)
* 新增 Snapshot Refresh 功能以及開放設定儲存路徑 Storage, Dir 欄位
* 新增設定管理介面IP以及time server不須重開機即可生效

\- bug fixed \-
* 修正 更新 firmware 時上傳檔案提供鎖住並提示的畫面
* 修正 Counter 頁面 clear counter 沒有清掉 Filter Matched Counter

## GRISM-3.4.220513
\- function added \-
* 新增 get statistics json uptime second (uptime_s) 參數
* 支援 設定flow/flowv6 enable/disable以及調整大小不需要重新開機
* 支援 GRISM XML `<chain/><in/>` 後面可以直接放 `<next/>`，不需要先放 `<fid/>`

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
* 修正 MEC handover 問題

## GRISM-3.3.220422
\- bug fixed \-
* 修正 snmp 無法取得 VPORT 的流量資訊
* 修正 snmp 無法取得 Flow Counter 資訊

## GRISM-3.3.220420
\- function added \-
* 新增 Backup, Restore from file 以及 Factory reset 功能
* 新增 filter 參數 tuple5_live_hashtable_size 支援，可由 syslog 或是 xmlrpc 動態新增 5-tuple 條件
* 新增 L2 Switch like 功能，能指定介面並儲存 source mac address 到 mac table，並根據 dest mac address 查詢 mac table 找到輸出介面

\- bug fixed \-
* VPORT 增加過多造成問題，調整為全部介面加起來不能超過 63
* 修正 lite clear counter 會短暫清掉 link status 的問題

## GRISM-3.2.220309
\- function added \-
* 新增 Statistic Counter Protocol/TCP/UDP coucurrent bytes
* 新增 Service enable/disable 頁面

\- bug fixed \-
* 新增 heartbeat id 設定以及過濾支援以避免刪除 heartbeat 設定造成過濾條件誤判

## 2022-01-13 (3.1)
\- function added \-
* 新增 web console v3 
* 新增 TACACS+ 
* 與 GRISM-A 型號整合虛擬介面對應實體介面的狀態取得以及設定、動態更新過濾條件
* 新增 Heartbeat 狀態顯示以及描述設定
* 新增直接在 Chain 裡面使用 not filter (ex. \<fid\>!F1\</fid\>) 
* 新增 DNS syslog type AAAA 以及 reply error code (1-9) 
* 新增 GTP-U parsing extension header 
* 新增 GRISM Port Linkdown filter

\- bug fixed \-
* 修正 www service 開機有時無法正常啟動

## 2021-09-16 (3.0)
\- function added \-
* 新增版本號碼 3.0 釋出
* 升級 sshd 版本到 OpenSSH_8.6p1，以修正舊版漏洞
* 使用更穩定的 httpd(www) 的軟體版本，解決長期 web console 不穩定需要定時重開的問題
* 提高 GRISM xml Save/Load 數量到 20 組
* 新增 GRISM Xml 2chart 文字圖形界面以及 2gml 底層格式界面，可選擇在不同模式下檢查設定是否正確

\- bug fixed \-
* netflow v9 以上 bytes 欄位從 4 Bytes 調整到 8 Bytes，以修正單一連線大小超過4G的問題

## 2021-06-28
\- function added \-
* 新增 ftp 過濾功能
  * 包含所有 tcp 20/21 port 以及 ftp passive mode 解析到 ftp-data 動態 ip/port 的連線
```xml
<filter id="1" sessionBase="no">
<or>
  <find name="ftp" relation="==" content=""/>
</or>
</filter>
```
* 新增 filter blockifempty 參數
  * 為了解決 filter 預設過濾條件是空的情況下會無條件放行，新增此參數可以調整成無條件阻擋
```xml
<filter id="1" blockifempty="yes">
<or>
</or>
</filter>
```
* 新增 GRISM BYPASS 設備標準型號，例如 G8-BPS
* 調整 MEC Template 更符合實際環境設定

## 2021-03-22
\- function added \-
* 新增 dns.qry.name_public_suffix 過濾功能
```xml
<filter id="10004" sessionBase="no">
      <or>
        <find name="dns.qry.name_public_suffix" relation="==" content="*.facebook.com" />
        <find name="dns.qry.name_public_suffix" relation="==" content="*.google.com" />
      </or>
</filter>
```
* 新增 SNMP Read Community 設定
* 新增 input packet drop syslog
  * Configuration -> Syslog -> Add New Target -> type:system -> subtype:alert_dropped_packets
  * type=3(system) subtype=0(input packet drop)
```
Mar 23 14:55:16 192.168.1.124 datetime=1970-01-01 03:40:43,type=3,subtype=0,interface=P1,packets=28,Mbps=150.63,Pps=22295,flows=49493/2097152,v6flows=261003/262144,cpu_load_average=13.28;4.47;3.84,mem_usage=519892/2060344
```
* (MEC) 新增 MEC Template
* (MEC) 新增 S1AP Table

## 2021-02-09
\- function added \-
* 新增於 Flow 頁面顯示 inpps, outpps (in/out packets per second) 流量
* 新增 dns.qry.type, dns.count.add_rr filter find support
* 新增 1 == 1 filter find 語法支援"一定會"或"一定不會"過濾到的條件，可運用在自動產生的黑名單過濾條件中以預防黑名單出現0筆的情況

## 2020-11-03
\- function added \-
* 新增 VPort 功能設定，可搭配switch設備vlan tagging+trunk port進到設備，以擴充可使用之實體介面數量
* 新增 ssl.server_name, ssl.server_name_public_suffix filter支援，過濾ssl憑證裡面的server name欄位
* 新增 system common syslog 支援，包含網頁登入登出、修改密碼、設定以及xml task送出等紀錄
* 新增 filter 筆數統計，包含xml task裡面的filter以及隱藏式黑名單filter的筆數
* 新增時區切換，目前支援 Taipei 以及 None 選項
* 新增IP過濾的同時如遇到有IP層以上的Tunnel，也比對Tunnel外層的IP
* 有外接usb管理介面的型號如T20,F4T4,F2T12，新增動態插拔usb也會自動設定管理介面

## 2020-09-17
\- performance tuning \-
* 提升VPort(T16 V0-V15)複製多份輸出以及複製多份自定義輸出(\<output\>)之效能

\- function added \-
* 新增 dns response IPv4 output，可搭配 dns.qry.name filter 過濾回應指定 IPv4
* 新增 snapshot 功能，可短暫側錄流量儲存成pcap檔案

## 2020-07-12
\- bug fixed \-
* Heartbeat 設定頁面的 Record ID 改由0開始以符合xml設定之代號

\- function added \-
* 新增 arp 相關filter 
  * arp
  * arp.request
  * arp.reply
  * arp.request.target.ip
* 新增 arp reply target mac output，可搭配 arp.request.target.ip filter 過濾回應指定 mac

## 2020-04-20
\- function added \-
* 新增 ssl.ja3_digest 過濾條件，可過濾 ssl ja3 hash 值
* 新增重複撥放 pcap 檔案功能 \<input type="replayPcap"\> tag
* 新增 Traffic Generator 功能可產生IP與Port資訊，並製造10 Gbps以上的線速流量 \<input type="traffic-gen"\> tag
* 新增 guest 帳號，登入後沒有設定權限，只有觀看權限
* 前三項功能都透過xml設定，語法請參考 https://packetx.github.io/gml

## 2020-04-10
\- function added \-
* 回復原廠設定功能 Help -> Restore

## 2020-03-10
\- function added \-
* T20, F2T12 10G<->1G 切換功能
* G8 LAN bypass
  * 開機bypass到主程式執行
  * 狀態顯示以及設定項目精簡
* 簡化xml設定
  * chain id 改成 optional
  * ```<find name="" relation="" content="" />``` 可簡化成 ```<f n="" r="" c=""/>```
* 新增flow table size設定。需從Configuration XML頁面設定，項目為 args 下面的 flowCacheBaseSize 以及 flowv6TableSize 參數

## 2020-02-25
\- bug fixed \-
* GRISM Task->Map xml註解Save後Load回來會消失

\- function added \-
* 新增 dns query name response ip addr filter
