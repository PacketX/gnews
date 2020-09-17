# Release Note
## 2020-09-17
\- performance tuning \-
* 提升VPort(T16 V0-V16)複製多份輸出以及複製多份自定義輸出(\<output\>)之效能

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
