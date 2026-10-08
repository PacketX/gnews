## GRISM-7.6.261008.1
\- 新增功能 \-
- 告警功能:CPU、記憶體、磁碟、溫度、風扇、服務與封包引擎狀態(設定 → 告警)
- 告警可送出 SNMP trap 與 syslog
- 帳號權限分為檢視、操作、設定、管理四級(設定 → 登入驗證)

\- 問題修正 \-
- 修正 SNMP trap 無法送出的問題

\- 項目變更 \-
- OpenSSH 更新至 10.6p1
- 設定頁顯示登入驗證方式與套用中狀態
- SNMP 系統通知服務可在設定 → 服務開關,預設關閉
- 系統狀態讀取更有效率


## GRISM-6.6.261008.4
\- 新增功能 \-
- 告警功能:CPU、記憶體、磁碟、溫度、風扇、服務與封包引擎狀態(設定 → 告警)
- 告警可送出 SNMP trap 與 syslog
- 帳號權限分為檢視、操作、設定、管理四級(設定 → 登入驗證)

\- 問題修正 \-
- 修正 SNMP trap 無法送出的問題
- 修正手動更新 SSH 元件失敗的問題

\- 項目變更 \-
- OpenSSH 更新至 10.6p1
- 設定頁顯示登入驗證方式與套用中狀態
- SNMP 系統通知服務可在設定 → 服務開關,預設關閉
- 系統狀態讀取更有效率,磁碟清單不再重複列出


## GRISM-7.6.261007.5
\- 新增功能 \-
- MEC 對應表顯示各 UE 的上下行流量,並可點欄位排序
- 管理服務可限制允許連線的 IP(設定 → 服務)
- 系統狀態顯示目前的主要設定
- 系統紀錄可清除
- 重播 pcap 支援奈秒時間格式

\- 問題修正 \-
- 修正 JA4 偶發計算錯誤
- 修正封包重播的問題
- 修正部分機種更新後服務未重新啟動
- 改善大型 pcap 檔案上傳的穩定性

\- 項目變更 \-
- pcap 上傳前檢查可用空間,並顯示進度
- 重播 pcap 時記錄正在播放的檔案
- 匯出頁與 MEC 對應表的顯示改善


## GRISM-6.6.261007.5
\- 新增功能 \-
- MEC 對應表顯示各 UE 的上下行流量,並可點欄位排序
- 管理服務可限制允許連線的 IP(設定 → 服務)
- 系統狀態顯示目前的主要設定
- 系統紀錄可清除
- 重播 pcap 支援奈秒時間格式

\- 問題修正 \-
- 修正 JA4 偶發計算錯誤
- 修正封包重播的問題
- 改善大型 pcap 檔案上傳的穩定性

\- 項目變更 \-
- pcap 上傳前檢查可用空間,並顯示進度
- 重播 pcap 時記錄正在播放的檔案
- 匯出頁與 MEC 對應表的顯示改善


## GRISM-7.6.261005.4
\- 新增功能 \-
- QUIC 連線支援 TLS 篩選與 JA4(設定 → 封包處理,預設關閉)
- 流量產生可設定播放時間
- 重播 pcap 可一次選擇多個檔案

\- 問題修正 \-
- 修正 JA4 計算問題
- 改善大型篩選清單載入與更新的穩定性
- 修正部分設定修改後未生效
- 修正清除計數的問題

\- 項目變更 \-
- 改善埠的顯示與選單
- 流表大小設定顯示可容納的連線數
- 匯出頁與封包處理頁的操作改善


## GRISM-6.6.261005.6
\- 新增功能 \-
- QUIC 連線支援 TLS 篩選與 JA4(設定 → 封包處理,預設關閉)
- 流量產生可設定播放時間
- 重播 pcap 可一次選擇多個檔案

\- 問題修正 \-
- 修正 JA4 計算問題
- 改善大型篩選清單載入與更新的穩定性
- 修正部分設定修改後未生效
- 修正封包重播的問題

\- 項目變更 \-
- 改善埠的顯示與選單
- 流表大小設定顯示可容納的連線數
- 匯出頁與封包處理頁的操作改善


## GRISM-7.6.261001.2
\- 問題修正 \-
- 修正韌體更新畫面在送出後約一秒就顯示「更新完成」,未經過「重開機中」與重新連線的階段;MIPS 機種尤其明顯
- 修正線上更新完成後 grism_watcher 可能沒有重新啟動。此服務負責套用韌體更新,一旦遺失,之後所有更新都不會生效,且不會有任何錯誤訊息;需重新開機才能恢復。本次起會驗證並重試,失敗也會寫入記錄
- 修正核心傾印檔只在網頁列出時才清理,改為開機時一併清理,避免佔用空間
- 修正首頁鏈結圖輸出埠較多時互相重疊
- 修正鏈結編輯畫面入口埠較多時文字超出節點範圍
- 修正部分埠選單未顯示埠的描述名稱(去重複、SD-WAN、Heartbeat、紀錄輸出、篩選器條件、模擬)
- 篩選器頁面無任何篩選器時的版面調整

\- 項目變更 \-
- 連線統計的 TCP／UDP 埠列表註明僅統計 1024 以下的埠
- 匯出頁「提交前自動保留的版本」由 5 份增加為 10 份

## GRISM-6.6.261001.2
\- 問題修正 \-
- 修正韌體更新畫面在送出後約一秒就顯示「更新完成」,未經過「重開機中」與重新連線的階段;MIPS 機種尤其明顯
- 修正核心傾印檔只在網頁列出時才清理,改為開機時一併清理,避免佔用空間
- 修正首頁鏈結圖輸出埠較多時互相重疊
- 修正鏈結編輯畫面入口埠較多時文字超出節點範圍
- 修正部分埠選單未顯示埠的描述名稱(去重複、SD-WAN、Heartbeat、紀錄輸出、篩選器條件、模擬)
- 篩選器頁面無任何篩選器時的版面調整

\- 項目變更 \-
- 連線統計的 TCP／UDP 埠列表註明僅統計 1024 以下的埠
- 匯出頁「提交前自動保留的版本」由 5 份增加為 10 份

## GRISM-7.6.260930.4
\- 新增功能 \-
- 全新網頁介面 GRISM Studio：視覺化編輯封包處理鏈結、提交前模擬封包路徑、即時流量與系統狀態、設定比對與版本回溯；支援中／英文與日間／夜間主題
- 封包側錄新增即時封包檢視，錄製中即可看到解析後的封包內容
- 新增當機偵錯：可開啟 core dump 收集並從網頁下載，封包引擎也可從網頁線上重新啟動
- 韌體更新改為只重啟相關服務，不再需要重新開機
- 新增 TLS JA4／JA4S 篩選欄位
- 新增服務管理：查詢各服務版本、啟動與停止服務
- 新增前端交換器（Q16）狀態查詢與重新啟動
- 系統狀態的記憶體資訊改為列出 hugepages 與 flow table 的固定佔用，使用率以扣除後計算
- sshd 與國家資料庫可單獨上傳更新，不需更新整個韌體
- 設定重新載入後保留統計計數與連線資訊
- 網頁閒置登出時間由 10 分鐘延長為 30 分鐘

\- 問題修正 \-
- 修正 VXLAN／GRE 隧道的內層封包在非 correlation 埠未被解析，導致針對內層欄位的篩選器失效
- 修正 LOOP 入口的封包內層未被解析
- 修正提交設定後封包引擎可能異常終止
- 修正網頁顯示時間的時區錯誤
- 修正 syslog 篩選器不支援新運算式
- 修正 JA4S 讀取 TLS 版本的問題
- 修正登入帳號權限判斷問題
- 修正更新檢查失敗時未說明原因
- 出廠設定移除範例對應規則

## GRISM-6.6.260930.4
\- 新增功能 \-
- 全新網頁介面 GRISM Studio：視覺化編輯封包處理鏈結、提交前模擬封包路徑、即時流量與系統狀態、設定比對與版本回溯；支援中／英文與日間／夜間主題
- 封包側錄新增即時封包檢視，錄製中即可看到解析後的封包內容
- 新增當機偵錯：可開啟 core dump 收集並從網頁下載，供原廠分析
- 篩選器支援巢狀運算式（and／or／not 可多層組合）
- 新增 TLS JA4／JA4S 篩選欄位
- 新增 S1AP／NGAP cellid 支援
- 新增服務管理：查詢各服務版本、啟動與停止服務
- sshd 與國家資料庫可單獨上傳更新，不需更新整個韌體
- 設定重新載入後保留統計計數與連線資訊
- 網頁閒置登出時間由 10 分鐘延長為 30 分鐘

\- 問題修正 \-
- 修正 VXLAN／GRE 隧道的內層封包在非 correlation 埠未被解析，導致針對內層欄位的篩選器失效
- 修正 LOOP 入口的封包內層未被解析
- 修正提交設定後封包引擎可能異常終止
- 修正輸出設定 QinQ 時 VLAN 計數錯誤
- 修正篩選器 and 條件的運算問題
- 修正 syslog 篩選器不支援新運算式
- 修正登入帳號權限判斷問題
- 修正更新檢查失敗時未說明原因

## GRISM-6.5.260715
\- 新增功能 \-
- 管理介面設定支援更多項目
- 新增 get dns qry name resp table dev api 支援

\- 問題修正 \-
- 修正 config set 問題
- 修正 country iso counter 問題
- 修正 filter matchResult 問題

## GRISM-6.5.260612
\- 問題修正 \-
- 修正 dns name 解析問題

## GRISM-6.5.260601
\- 新增功能 \-
- 新增 dns dynamic update 解析以及 syslog 支援
- 新增 dns opcode, rcode syslog，移除 dtype_num 欄位
- 新增管理介面 link status filter 供 HA 使用
- 新增 grism xml chain in/out vlan tagging/stripping 屬性支援
- 新增 web service 10 分鐘閒置自動登出
- 新增 vxlan strip 資料移到封包尾端再加回封包前端
- 新增 web GRISM XML 刪除 run1-9 xml 支援

\- 問題修正 \-
- 移除 ssh dsa key 產生，修正 G8s 偶爾開機很慢

## GRISM-6.5.260320
\- 新增功能 \-
- 新增 management1 支援

\- 問題修正 \-
- 修正修改 tcp mss 欄位造成封包重複的問題
- 修正 G8s(KOB3410) 硬體 phy port 初始化問題

## GRISM-6.5.260123
\- 新增功能 \-
- 新增 hostname 以及 lldpd 支援

\- 問題修正 \-
- 修正 pcap replay 檔案數量上限問題
- 修正無法取得 l2gre/vxlan correlation table 的問題
- 修正 l2gre 以及 vxlan l2broadcast 問題
- 修正帶有 vlan 的 broadcast tunnel 封包問題
- 修正 console 登入訊息持續出現的問題
- 修正 dns query type log 問題

## GRISM-6.5.251230
\- 新增功能 \-
- 新增管理介面 mac address 取得
- runX.gdp 數量增加到 16 組

\- 問題修正 \-
- 修正 decrypt 功能問題
- 修正 traffic-gen 功能問題
- 修正首次安裝 sshd session link path 錯誤

## GRISM-6.5.251107
\- 問題修正 \-
- 修正 L3/L4 IP/TCP/UDP 封包錯誤檢查處理

## GRISM-6.5.251029
\- 新增功能 \-
- sshd 升級到 10.2p1 以修正安全性問題
- dbip country mmdb 更新到 2025-10

## GRISM-6.4.250911
\- 問題修正 \-
- 修正 www 服務不穩定的問題

## GRISM-6.4.250731
\- 新增功能 \-
- 新增 netflow counter 以及 eps 支援
- netflow 以及 statistics 支援介面 ifidx

## GRISM-6.3.250714
\- 新增功能 \-
- 簡化 XML 儲存/載入功能
- 強化 GRISM-A heartbeat 偵測以及切換 XML 設定
- netflow 封包長度統計新增可選擇使用 ip header length

\- 問題修正 \-
- 修正 google chrome (version 138) 顯示問題

## GRISM-6.2.250502
\- 新增功能 \-
- GRISM-A heartbeat 送出功能支援
- gtpc log 支援 ipv6 prefix

\- 問題修正 \-
- 修正 vxlan 變更 client port 的問題
- 修正 web 服務安全性問題
- 修正 G8s bypass 初始化問題

## GRISM-6.1.250327
\- 新增功能 \-
- 新增 ja4 ja4s 支援
- 新增 matched syslog blocked=yes/no 字串支援
- 修正安全性問題

## GRISM-5.5.250317
\- 新增功能 \-
- sshd 升級到 9.9p2

## GRISM-6.0.250310
\- 新增功能 \-
- sshd 升級到 9.9p2
- 新增 DNS over TCP 支援
- 新增 dns client query per second (QPS) syslog 支援

## GRISM-5.10.250224
\- 新增功能 \-
- syslog, dpilog, netflow 以及介面描述設定不中斷系統即可生效
- 新增 tacacs+ admin(15)/guest(1) 權限支援
- 新增更多 GTP-C syslog 支援
- 新增 GRISM 監看其他 GRISM 介面
- 新增 UI 隱藏 runX.xml
- 新增 encapsulation Encrypt Key timeout 支援
- 新增 flowv6 filter 支援

\- 問題修正 \-
- 修正 gtp-c v1,v2 解析問題
- 修正 grism xml hash table 記憶體使用量問題
- 修正解析到不支援的 find type 時的問題

## GRISM-5.5.250206
\- 新增功能 \-
- sshd 升級到 9.8p1
- 修正安全性問題

## GRISM-5.9.250114
\- 新增功能 \-
- 新增 G8 硬體 iptables 支援
- dbip country mmdb 更新到 2024-12
- netflow v9 新增 gtp/gre/vxlan tunnel 外層 ip

\- 問題修正 \-
- 修正 RAN-UE-NGAP-ID 整數範圍問題
- 修正 NGAP transportLayerAddress ipv6 解析問題

## GRISM-5.5.250114
\- 問題修正 \-
- 修正 RAN-UE-NGAP-ID 整數範圍問題
- 修正 NGAP transportLayerAddress ipv6 解析問題

## GRISM-5.9.241104
\- 新增功能 \-
- 新增更多 DPI 功能支援
- syslogd 保留更多輪替後的 log 檔案
- 新增全域時區設定支援

\- 問題修正 \-
- icmp heartbeat 支援 vlan
- 修正 arp table 記錄問題
- 修正 www 服務問題

## GRISM-5.8.241004
\- 新增功能 \-
- nginx 設定強化
- xmlrpc 服務預設改為 localhost 以解決安全性問題

\- 問題修正 \-
- 修正 Q4T4 25G/10G link status 問題

## GRISM-5.8.240730
\- 新增功能 \-
- 新增 Country Statistics 支援
- openssh 升級到 9.8p1
- 新增 ssl dpi ja3s syslog 支援
- 新增 dns query name filter 預設不分大小寫的支援
- 新增 tcp segments 重組，僅限 ssl 且最多兩個 segment
- 新增 G8S/T12S/T12/T20(F2T12,F4T4,Q4T4) iptables 阻擋 icmp timestamp request/reply（需重裝 linux kernel）
- 新增 web console DEV API 頁面支援
- 新增 web console Flow 頁面顯示硬體 bypass/heartbeat 介面狀態

## GRISM-5.7.240603
\- 新增功能 \-
- 新增 Q4T4 型號 build 支援

\- 問題修正 \-
- 修正 modify_tcp_syn_mss 問題
- 修正 gre correlation tunnel 問題

## GRISM-5.7.240529
\- 新增功能 \-
- 強化 vxlan breakout 搭配 MEC [MEC+VXLAN Breakout](https://packetx.gitbook.io/grism-xml/docs/mec+vxlan-breakout)
- 新增更多 heartbeat bypass 詳細記錄

\- 問題修正 \-
- 修正 netflow v10 uptime 問題
- 修正 vxlan/gre correlation tunnel 問題

## GRISM-5.6.240423
\- 新增功能 \-
- 支援指定介面去重複(deduplication)，包含 LOOP 介面
- KOB3410 (G8S) 風扇預設轉速調整為 50%

## GRISM-5.6.240418
\- 新增功能 \-
- sshd 升級到 9.7p1
- 從 ssh 移除 ChaCha20-Poly1305 演算法

## GRISM-5.5.240418
\- 新增功能 \-
- sshd 升級到 9.7p1
- 從 ssh 移除 ChaCha20-Poly1305 演算法

## GRISM-5.6.240412
\- 新增功能 \-
- 新增修改來源/目的 port
- 新增 output type TCP Reset 支援
- 新增 T12S 硬體(KOB3400) 智慧風扇控制支援
- icmp heartbeat 功能更為通用

## GRISM-5.6.240320
\- 新增功能 \-
- 新增 Statistics -> Service 支援
- 新增 T12S/G8S Configuration -> Interface 介面對應設備圖支援
- 新增 [偵測目的 IP 不在 DNS Response IP Table 內](https://packetx.gitbook.io/grism-xml/docs/detect-if-dstination-ip-not-in-dns-response-ip-table)

## GRISM-5.6.240201
\- 新增功能 \-
- 新增線上更新版本檢查、下載以及升級

## GRISM-5.5.240122
\- 新增功能 \-
- nginx 設定強化

## GRISM-5.5.240108
\- 新增功能 \-
- 新增 [NAT](https://packetx.gitbook.io/grism-xml/docs/l3nat_breakout) 支援
- 新增 [mec.js](https://packetx.gitbook.io/grism-xml/reference/mec.js)，簡化 MEC 的 GRISM XML
- 新增 ICMP reply 支援
- 新增 ICMP type, code filter
- 新增 [action arp reply / icmp reply / icmp reply fragment need](https://packetx.gitbook.io/grism-xml/readme/action#less-than-arp_reply_default_mac-greater-than) tag 支援
- 新增 arp_srcip output 支援，用於 arp dest mac
- Country Codes filter 新增 block if empty 參數
- 新增 Status -> Process 核心程式狀態顯示
- 核心程式異常結束時新增 syslog 告警

## GRISM-5.4.231214
\- 新增功能 \-
- dns.a filter 比對記錄新增 dns_query_name 欄位
- 新增 IM8724 nvme 驅動，支援 NVME ssd
- 更換產品 logo
- Flow 頁面新增硬碟寫入吞吐量顯示
- 強化 snort rule 過濾

\- 問題修正 \-
- 極端 s1ap 串流測試下更為穩定
- 寫入 pcap 檔案更為穩定
- 修正 T12s bypass 介面偶爾 crc error
- 修正刪除 VPorts 的問題

## GRISM-5.4.231020
\- 新增功能 \-
- 支援 l2gre, vxlan mapping table filter
- 強化 replay pcap 速度並更為穩定
- 支援 ICMP heartbeat

\- 問題修正 \-
- 修正 SCTP 重組在 CN73xx 一天、CN78xx 兩週後 crash
- 修正 System->Status 時區問題
- 修正 www 新增使用者允許重複的問題

## GRISM-5.3.230918
\- 問題修正 \-
- 修正 System->Status 時區問題

## GRISM-5.4.230908
\- 問題修正 \-
- 修正非 MEC 設備 cron 清除 s1ap table items 造成 CPU 飆高（預設每週一 02:00）

## GRISM-5.4.230825
\- 新增功能 \-
- 新增 pcap 檔案上傳供 pcap replay 使用
- pcap replay 功能新增 pcapng 格式以及 Linux cooked captured v1 層支援
- 新增 syslog daemon 預設啟用支援
- 新增 G8S Bypass WDT 功能支援
- clear lite counter api 新增清除 filter counter 支援

## GRISM-5.3.230824
\- 新增功能 \-
- 升級韌體後還原密碼設定

\- 問題修正 \-
- 修正 SCTP Reconstruct 記憶體問題

## GRISM-5.3.230814
\- 新增功能 \-
- ssh 從 8.6p1 升級到 9.4p1
- nginx 設定強化

## GRISM-5.3.230804
\- 新增功能 \-
- snmp 新增取得風扇故障以及溫度過高資訊
- 停用系統狀態 syslog 告警（5.2 起預設開啟，已無用途）

## GRISM-5.3.230719
\- 新增功能 \-
- [filter](https://packetx.gitbook.io/grism-xml/readme/filter) 新增 start, position, within 以及 mpslog 參數支援
- 新增 http.host 以及 http.request.uri filter 支援
- 新增 ssl.ja3s_digest filter 支援
- 新增 MEC cron 清除閒置超過上限的 s1ap table items
- counter 以及 syslog 告警新增每秒比對數(matched per second)支援
- MEC Mapping 新增 <=, > filter 以及清除閒置資料支援

## GRISM-5.2.230615
\- 新增功能 \-
- 支援 MEC mec.mapping.ue.ipv4.connected filter，用於觸發 core paging 問題
- System Status 支援溫度/風扇顯示
- Syslog 告警支援開機/Link status/溫度/風扇/CPU 使用率

\- 問題修正 \-
- 修正 MEC CELL-ID 欄位解析問題

## GRISM-5.1.230517
\- 新增功能 \-
- MEC Mapping 頁面 新增 PLMN ID, CELL ID, SPID 欄位
- MEC 新增 gtp.data.by.s1ap.CellIdentity, gtp.data.by.s1ap.SubscriberProfileIDforRFP 過濾條件
- In-Tunnel 新增 L2MPLS/MPLS in UDP/GRE
- 新增 output/action striping mpls-in-gre xml
- 新增 output tagging vxlan xml [VXLAN breakout](https://packetx.gitbook.io/grism-xml/docs/vxlan-breakout)

## GRISM-5.0.230330
\- 新增功能 \-
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
- dnslog/httplog/ssllog 支援 client/server ipv6

\- 問題修正 \-
- 修正 CN71XX 型號 input bytes 流量計算誤差

## GRISM-4.8.230103
\- 新增功能 \-
- 新增 T12S 硬體型號支援
  - 12 個 10G 介面，包含兩對 10G 硬體 Bypass 模組
- 新增\<output\>\<tagging\> l2gre support, 請參考 [L2 GRE Breakout](https://packetx.gitbook.io/grism-xml/docs/l2-gre-breakout)
- 新增 Help -> User Manual [GRISM XML](https://packetx.gitbook.io/grism-xml) 手冊
- Pcap Replay/Traffic Generate/Packet Snapshot 頁面提供停止功能以及 GRISM XML 語法
- 新增 filter -> find arp.request.sender.ip 過濾條件

## GRISM-4.7.221122
\- 功能強化 \-
* 調整 MEC s1ap/ngap record mapping 的 hash table 碰撞處理方式

## GRISM-4.7.221116
\- 問題修正 \-
* 修正 MEC Handover 問題

## GRISM-4.6.221101
\- 新增功能 \-
* 新增 output xml 參數 nvgre_sip, nvgre_dmac, nvgre_type 支援以方便在不指定介面來源IP的情況下也能送 GRE Tunnel，另新增 eth/ip 兩種 type 設定

\- 問題修正 \-
* Web Console 調整不預先載入 run1-run9 xml 以及 js 等內容，避免因檔案內容過大造成載入異常

## GRISM-4.5.221022
\- 新增功能 \-
* Web Console 支援 GRISM XML 多檔案編輯包含 run.xml, run1.xml ~ run9.xml 以及 js 檔案
* Web Console 新增 System->Log 頁面
* ouptut dir xml 新增 category 參數以時間目錄分類 pcap 檔案 以及 Packet Snapshot, Pcap Replay 多支援一層目錄進出

\- 問題修正 \-
* 修正設定 heartbeat id 如果沒有同時設定 description 會失敗的問題
* 修正 IP Fragmentation Correlation 以及 Reconstruct 在封包順序不對時可能會有問題


## GRISM-4.3.221004
\- 新增功能 \-
* GRISM XML 新增 ```<script></script>```
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

\- 問題修正 \-
* MEC S1AP/NGAP items table 資源釋放調整解決於大網共構下UE頻繁attach/detach能運作正常
* snmpd 暫存檔案改寫到ramdisk 避免檔案系統因異常重開機後發生無法寫入的問題


## GRISM-4.2.220826
\- 新增功能 \-
* MEC Mapping 改為以封包為基礎
* 支援 MEC NGAP Handover
* 支援 MEC S1AP/NGAP 解析項目的 syslog
* MEC Server 到 UE 的處理更為順暢
* traffic-gen xml 支援 \<msinterval\> tag 以及 ICMP 協定

\- 問題修正 \-
* 修正 MEC S1AP/NGAP items table timeout 釋放未加鎖的問題

## GRISM-3.13.220718
\- 新增功能 \-
* 新增 Pcap Replay 功能操作頁面
* 新增 Traffic Generate 功能操作頁面

## GRISM-3.12.220711
\- 問題修正 \-
* 修正 SCTP DATA chunk padding size 計算錯誤

## GRISM-3.11.220628
\- 問題修正 \-
* 修正 GRISM-3.7.220527 之後版本部分 flow 無法 timeout 的問題

## GRISM-3.10.220623
\- 新增功能 \-
* 支援 output icmp_reply_fragment_need 功能 [<icmp_reply_fragment_need/>](https://packetx.gitbook.io/grism-xml/readme/output#less-than-icmp_reply_fragment_need-greater-than)

\- 問題修正 \-
* www 支援設定送 syslog 到多個 server


## GRISM-3.9.220621
\- 新增功能 \-
* Syslog type:system 送出更多細節，包含 ifcfgs 設定、介面啟用/停用、tacacs+、netflow 等
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
\- 新增功能 \-
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
\- 新增功能 \-
* 新增 ouput 支援 minbps, maxbps 限制頻寬功能
```xml
<output id="8" minbps="200000000" maxbps="500000000">
    <port>P8</port>
</output>
```

## GRISM-3.6.220525
\- 新增功能 \-
* 新增/刪除 使用者功能
* 新增 Port Enable/Disable 功能 (需重開機生效)
* 新增 Snapshot Refresh 功能以及開放設定儲存路徑 Storage, Dir 欄位
* 新增設定管理介面IP以及time server不須重開機即可生效

\- 問題修正 \-
* 修正 更新 firmware 時上傳檔案提供鎖住並提示的畫面
* 修正 Counter 頁面 clear counter 沒有清掉 Filter Matched Counter

## GRISM-3.4.220513
\- 新增功能 \-
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
\- 問題修正 \-
* 修正 MEC handover 問題

## GRISM-3.3.220422
\- 問題修正 \-
* 修正 snmp 無法取得 VPORT 的流量資訊
* 修正 snmp 無法取得 Flow Counter 資訊

## GRISM-3.3.220420
\- 新增功能 \-
* 新增 Backup, Restore from file 以及 Factory reset 功能
* 新增 filter 參數 tuple5_live_hashtable_size 支援，可由 syslog 或是 xmlrpc 動態新增 5-tuple 條件
* 新增 L2 Switch like 功能，能指定介面並儲存 source mac address 到 mac table，並根據 dest mac address 查詢 mac table 找到輸出介面

\- 問題修正 \-
* VPORT 增加過多造成問題，調整為全部介面加起來不能超過 63
* 修正 lite clear counter 會短暫清掉 link status 的問題

## GRISM-3.2.220309
\- 新增功能 \-
* 新增 Statistic Counter Protocol/TCP/UDP concurrent bytes
* 新增 Service enable/disable 頁面

\- 問題修正 \-
* 新增 heartbeat id 設定以及過濾支援以避免刪除 heartbeat 設定造成過濾條件誤判

## 2022-01-13 (3.1)
\- 新增功能 \-
* 新增 web console v3 
* 新增 TACACS+ 
* 與 GRISM-A 型號整合虛擬介面對應實體介面的狀態取得以及設定、動態更新過濾條件
* 新增 Heartbeat 狀態顯示以及描述設定
* 新增直接在 Chain 裡面使用 not filter (ex. \<fid\>!F1\</fid\>) 
* 新增 DNS syslog type AAAA 以及 reply error code (1-9) 
* 新增 GTP-U parsing extension header 
* 新增 GRISM Port Linkdown filter

\- 問題修正 \-
* 修正 www service 開機有時無法正常啟動

## 2021-09-16 (3.0)
\- 新增功能 \-
* 新增版本號碼 3.0 釋出
* 升級 sshd 版本到 OpenSSH_8.6p1，以修正舊版漏洞
* 使用更穩定的 httpd(www) 的軟體版本，解決長期 web console 不穩定需要定時重開的問題
* 提高 GRISM xml Save/Load 數量到 20 組
* 新增 GRISM Xml 2chart 文字圖形界面以及 2gml 底層格式界面，可選擇在不同模式下檢查設定是否正確

\- 問題修正 \-
* netflow v9 以上 bytes 欄位從 4 Bytes 調整到 8 Bytes，以修正單一連線大小超過4G的問題

## 2021-06-28
\- 新增功能 \-
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
\- 新增功能 \-
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
\- 新增功能 \-
* 新增於 Flow 頁面顯示 inpps, outpps (in/out packets per second) 流量
* 新增 dns.qry.type, dns.count.add_rr filter find
* 新增 1 == 1 filter find 語法支援"一定會"或"一定不會"過濾到的條件，可運用在自動產生的黑名單過濾條件中以預防黑名單出現0筆的情況

## 2020-11-03
\- 新增功能 \-
* 新增 VPort 功能設定，可搭配switch設備vlan tagging+trunk port進到設備，以擴充可使用之實體介面數量
* 新增 ssl.server_name, ssl.server_name_public_suffix filter支援，過濾ssl憑證裡面的server name欄位
* 新增 system common syslog 支援，包含網頁登入登出、修改密碼、設定以及xml task送出等紀錄
* 新增 filter 筆數統計，包含xml task裡面的filter以及隱藏式黑名單filter的筆數
* 新增時區切換，目前支援 Taipei 以及 None 選項
* 新增IP過濾的同時如遇到有IP層以上的Tunnel，也比對Tunnel外層的IP
* 有外接usb管理介面的型號如T20,F4T4,F2T12，新增動態插拔usb也會自動設定管理介面

## 2020-09-17
\- 效能調校 \-
* 提升VPort(T16 V0-V15)複製多份輸出以及複製多份自定義輸出(\<output\>)之效能

\- 新增功能 \-
* 新增 dns response IPv4 output，可搭配 dns.qry.name filter 過濾回應指定 IPv4
* 新增 snapshot 功能，可短暫側錄流量儲存成pcap檔案

## 2020-07-12
\- 問題修正 \-
* Heartbeat 設定頁面的 Record ID 改由0開始以符合xml設定之代號

\- 新增功能 \-
* 新增 arp 相關filter 
  * arp
  * arp.request
  * arp.reply
  * arp.request.target.ip
* 新增 arp reply target mac output，可搭配 arp.request.target.ip filter 過濾回應指定 mac

## 2020-04-20
\- 新增功能 \-
* 新增 ssl.ja3_digest 過濾條件，可過濾 ssl ja3 hash 值
* 新增重複撥放 pcap 檔案功能 \<input type="replayPcap"\> tag
* 新增 Traffic Generator 功能可產生IP與Port資訊，並製造10 Gbps以上的線速流量 \<input type="traffic-gen"\> tag
* 新增 guest 帳號，登入後沒有設定權限，只有觀看權限
* 前三項功能都透過xml設定，語法請參考 https://packetx.github.io/gml

## 2020-04-10
\- 新增功能 \-
* 回復原廠設定功能 Help -> Restore

## 2020-03-10
\- 新增功能 \-
* T20, F2T12 10G<->1G 切換功能
* G8 LAN bypass
  * 開機bypass到主程式執行
  * 狀態顯示以及設定項目精簡
* 簡化xml設定
  * chain id 改成 optional
  * ```<find name="" relation="" content="" />``` 可簡化成 ```<f n="" r="" c=""/>```
* 新增flow table size設定。需從Configuration XML頁面設定，項目為 args 下面的 flowCacheBaseSize 以及 flowv6TableSize 參數

## 2020-02-25
\- 問題修正 \-
* GRISM Task->Map xml註解Save後Load回來會消失

\- 新增功能 \-
* 新增 dns query name response ip addr filter
