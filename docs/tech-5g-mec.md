# 用 UERANSIM / srsRAN + Open5GS 架 5G 與 4G/NSA 測試環境,驗證 GRISM 的 MEC breakout

## 目的

在一台 Linux VM 上同時跑核心網(Open5GS,5GC 與 EPC 都有)與基地台/手機模擬器(5G SA 用 UERANSIM,4G 與 NSA 用 srsRAN 4G),讓 N2/S1-MME(NGAP、S1AP over SCTP)與 N3/S1-U(GTP-U)**真的走實體線**經過 GRISM 的 inline 鏈結,再接一台邊緣主機,驗證 MEC breakout:指定 cell 的 UE 存取邊緣服務時,GRISM 去掉 GTP 直接送到邊緣,回程再封裝回基地台;其餘流量照走核心網,核心網完全看不到被 breakout 的部分。

本文用的機器:GRISM G8S-MEC(P4 接基地台側、P5 接核心網側、P3 接邊緣主機)與一台 Ubuntu 24.04 VM(8 核、16 GB,四張網卡)。

## 架構

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
│ [主機]                           │           │
│   ens20  10.5.0.2  AMF(N2) UPF(N3)│───── P5 ──┘
│          10.5.0.3  SGW-U(S1-U)  MME(S1-MME) 也在 10.5.0.2
│   ogstun 10.45.0.1/16 ─ NAT ─ ens18 ─ 網際網路
│                                 │
│ [edge namespace]                │
│   ens21  10.45.255.254  HTTP    │───── P3 (邊緣,breakout 出口)
└─────────────────────────────────┘
```

| 介面 | namespace | 位址 | 角色 | GRISM 埠 |
|---|---|---|---|---|
| ens19 | `ran` | 10.5.0.1 / .11 / .12,10.5.0.21 | 三個 gNB 與一個 eNB 的 N2/S1 位址,UE 也在這裡 | P4 |
| ens20 | 主機 | 10.5.0.2、10.5.0.3 | AMF 的 NGAP 與 MME 的 S1AP、UPF 的 GTP-U(.2);SGW-U 的 GTP-U(.3) | P5 |
| ens21 | `edge` | 10.45.255.254/16 | 邊緣主機(HTTP 服務) | P3 |
| ens18 | 主機 | 192.168.x.x | 管理與 UE 的 NAT 出口 | — |

兩個設計重點:

- **gNB 那一側必須放在獨立的 network namespace。** 同一台主機上的兩張網卡若在同一個 namespace,核心會在本機直接把封包交給對方,根本不會出線;分開之後,gNB 送給 AMF 的每個封包都從 ens19 出去、經 GRISM 再從 ens20 進來。
- **邊緣主機也要獨立的 namespace。** 主機本身因為 UPF 的 `ogstun` 已持有 UE 位址範圍 10.45.0.0/16,邊緣主機的 IP 又必須在同一範圍內(見下文 MEC 設定),若放在主機上,邊緣對 UE 的回覆會走 `ogstun` 而不是出線。

## 安裝

```bash
apt install -y software-properties-common git make gcc g++ cmake libsctp-dev lksctp-tools docker.io iptables

# MongoDB:Open5GS 需要;5.0 以上要 CPU 有 AVX,QEMU 預設 CPU 型號沒有,改用 Docker 跑 4.4
systemctl enable --now docker
docker run -d --name open5gs-mongo --restart unless-stopped -p 127.0.0.1:27017:27017 -v open5gs-mongo:/data/db mongo:4.4

# Open5GS 2.8
add-apt-repository -y ppa:open5gs/latest && apt install -y open5gs

# UERANSIM
git clone --depth 1 https://github.com/aligungr/UERANSIM /opt/UERANSIM && make -C /opt/UERANSIM -j8
```

PPA 沒有附 `open5gs-dbctl`,從 Open5GS 原始碼 `misc/db/open5gs-dbctl` 取得;它呼叫 `mongosh`,用一個 wrapper 轉給容器即可:

```sh
#!/bin/sh
exec docker exec -i open5gs-mongo mongo "$@"      # 存成 /usr/local/bin/mongosh
```

## Open5GS 設定

只有兩個位址要改:AMF 的 N2 與 UPF 的 N3 綁到 ens20,其餘介面維持 loopback。三個 gNB 用不同 TAC,`tai` 要都列出。

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

用戶用 UERANSIM 範例的金鑰:

```bash
for i in 1 2 3; do open5gs-dbctl add 99970000000000$i 465B5CE8B199B49FAA5F0A2EE238A6BC E8ED289DEBA952E4283B54E88E6183CA; done
```

UE 的流量從 UPF 的 `ogstun` 出來後要 NAT 出去。Docker 會把 FORWARD 預設改成 DROP,所以要明確放行:

```bash
sysctl -w net.ipv4.ip_forward=1
iptables -I FORWARD -s 10.45.0.0/16 -j ACCEPT
iptables -I FORWARD -d 10.45.0.0/16 -j ACCEPT
iptables -t nat -A POSTROUTING -s 10.45.0.0/16 -o ens18 -j MASQUERADE
```

## 接線:namespace 與位址

```bash
ip netns add ran;  ip link set ens19 netns ran
ip -n ran addr add 10.5.0.1/24 dev ens19; ip -n ran addr add 10.5.0.11/24 dev ens19; ip -n ran addr add 10.5.0.12/24 dev ens19
ip -n ran link set lo up; ip -n ran link set ens19 up
ip addr add 10.5.0.2/24 dev ens20
ip netns add edge; ip link set ens21 netns edge
ip -n edge addr add 10.45.255.254/16 dev ens21; ip -n edge link set lo up; ip -n edge link set ens21 up
mkdir -p /etc/netns/ran && echo "nameserver 8.8.8.8" > /etc/netns/ran/resolv.conf
```

先確認線路:在 `ran` 裡 `ping 10.5.0.2`,同時看 GRISM 介面流量頁的 P4/P5 計數有沒有動;兩個方向都要通。

## UERANSIM 設定

三個 gNB 同 PLMN,cell 與 TAC 不同;每個 UE 用 `gnbSearchList` 指定掛哪一個 gNB。設定檔從 `config/open5gs-gnb.yaml`、`config/open5gs-ue.yaml` 複製修改。

| gNB | `nci` | `tac` | `linkIp` / `ngapIp` / `gtpIp` | `amfConfigs` | UE |
|---|---|---|---|---|---|
| A | `0x001000000` | 1 | 10.5.0.1 | 10.5.0.2 | imsi-999700000000001 |
| B | `0x002000000` | 2 | 10.5.0.11 | 10.5.0.2 | imsi-999700000000002 |
| C | `0x003000000` | 3 | 10.5.0.12 | 10.5.0.2 | imsi-999700000000003 |

全部在 `ran` namespace 裡執行:

```bash
ip netns exec ran /opt/UERANSIM/build/nr-gnb -c gnb-A.yaml      # B、C 同理
ip netns exec ran /opt/UERANSIM/build/nr-ue  -c ue-1.yaml       # 2、3 同理
```

UE 註冊成功後會在 `ran` 裡建立 `uesimtun0..2`(10.45.0.x)。要讓 `ran` 裡的 curl/ping 都「用手機上網」,可把預設路由指向 `uesimtun0`,但**先把核心網的位址固定在實體線上**(`ip -n ran route add 10.5.0.2/32 dev ens19`),否則 gNB 自己的 N2/N3 也會被導進 UE 隧道。

邊緣主機用 Python 一行起 HTTP 服務:

```bash
ip netns exec edge python3 -m http.server 80 --bind 10.45.255.254 --directory /srv/edge
```

## 4G / NSA:srsRAN 4G 的 eNB 加一個 NR cell

UERANSIM 只做 5G SA。4G 與 NSA(EN-DC)用 [srsRAN 4G](https://github.com/srsran/srsRAN_4G) 的 srsENB + srsUE,射頻以 ZMQ 模擬、不需要 SDR;NSA 是 srsENB 多帶一個 NR cell、srsUE 開啟 NR RAT,照 srsRAN 的「5G NSA over ZMQ」應用筆記設定。

編譯(release 23.11)在這台 VM 上有兩個必要選項:GCC 13 會把 `array-bounds` 誤報當錯誤,要 `-DENABLE_WERROR=OFF`;srsRAN 預設用 `-march=skylake`,編出的程式在沒有 AVX 的 CPU 上**一啟動就 SIGILL**,要 `-DGCC_ARCH=x86-64-v2`(這顆 CPU 有 SSE4.2/POPCNT、沒有 AVX)。

```bash
cmake -DGCC_ARCH=x86-64-v2 -DENABLE_WERROR=OFF -DENABLE_AVX=OFF -DENABLE_AVX2=OFF -DENABLE_FMA=OFF -DENABLE_ZEROMQ=ON ..
make -j8 srsenb srsue
```

設定從 `srsenb/*.conf.example`、`srsue/ue.conf.example` 複製修改,全部在 `ran` namespace 裡執行:

| 檔案 | 改動 |
|---|---|
| `enb.conf` | `mcc=999 mnc=70`、`mme_addr=10.5.0.2`、`gtp_bind_addr=s1c_bind_addr=10.5.0.21`、`n_prb=50`;`[rf] device_name=zmq`,`device_args=fail_on_disconnect=true,tx_port0=tcp://*:2000,rx_port0=tcp://localhost:2001,tx_port1=tcp://*:2100,rx_port1=tcp://localhost:2101,id=enb,base_srate=23.04e6`(埠 0 是 LTE、埠 1 是 NR) |
| `rr.conf` | LTE cell 維持範例(`cell_id=0x01`、`tac=0x0007`、`dl_earfcn=3350`);`nr_cell_list` 加一個 NR cell:`rf_port=1; cell_id=0x02; tac=0x0007; pci=500; root_seq_idx=204; dl_arfcn=368500; band=3;` |
| `ue.conf` | `[rf]` 對應的 ZMQ 兩組埠;`[rat.eutra] dl_earfcn=3350 nof_carriers=1`;`[rat.nr] bands=3,78 nof_carriers=1`;`[rrc] release=15`;`[usim] imsi=999700000000004`,金鑰同 UERANSIM 範例 |

核心網這邊只多兩件事:`mme.yaml` 的 `s1ap` 綁 10.5.0.2、`tai` 加 TAC 7;`sgwu.yaml` 的 `gtpu` 綁 10.5.0.3(ens20 上多加的位址,因為 UPF 已佔 10.5.0.2:2152)。HSS 讀的是同一個 MongoDB,`open5gs-dbctl add` 的用戶 4G/5G 都能用。

兩個經驗:**先啟動 eNB 再啟動 UE**——UE 單獨重啟會卡在「Attaching UE...」(ZMQ 的連線狀態),eNB 一起重啟就正常;以及 `apt install` 函式庫時 needrestart 會順手重啟 Open5GS 的服務,SMF 一重啟所有 PDU session 都沒了、UPF 對舊 TEID 回 GTP-U Error Indication,症狀是全部 UE 同時斷線——apt 之後把基地台與 UE 重新註冊即可。

attach 成功後 UE 拿到 `tun_srsue`(10.45.0.x,與 5G 同一個位址池);console 出現「RRC NR reconfiguration successful」與 NR cell 的隨機存取,就是 EN-DC 已建立。

## GRISM 設定

### 1. 開啟 MEC 相關功能

設定頁 → MEC 卡片一鍵開啟 `s1cCorrelation`、`in-tunnels/GTP`、`flowExtensionGtpTunnelhdr`。`s1cCorrelation` 讓引擎解析 NGAP,從 InitialUEMessage 記下每個 UE 的 cell、從 PDU session 建立訊息記下兩個方向的 TEID;GTP 解封裝與 flow extension 則讓資料面封包能對回這些 session。

### 2. Breakout 鏈結

以 Studio 內建的 MEC 範例(封包處理 → MEC → 開啟範例設定)改成實際埠與位址:P4 是 gNB、P5 是核心網、P3 是邊緣。

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

各段的意思:

- 核心網 → gNB(P5 → P4)原樣通過。
- gNB → 核心網(P4):來源是 UE 位址(F1)**且**目的是邊緣主機(F3)**且**來自 cell A(F4)的封包,去掉 GTP、以 ARP 解析 `gateway` 的 MAC 後從 P3 送出(O1);其餘照送 P5。
- 邊緣 → (P3):邊緣主機對 UE 位址的 ARP 由 GRISM 代答(F2 → O3),其他封包用先前學到的隧道重新封裝成 GTP 送回 gNB(O2)。這就是為什麼邊緣主機的 IP 要放在 UE 的位址範圍裡:邊緣端把 UE 當成同一個 LAN 上的主機來 ARP。

### 3. cell 的值怎麼算

`gtp.data.by.s1ap.CellIdentity` 比的是 NGAP 裡 36 bit 的 NR Cell Identity **取高 N 個 bit** 的值,N 預設 26(設定頁 MEC 卡片的 `nrCellIdentityBits` 可改,1–32)。gNB A 的 NCI `0x001000000` 在 26 bit 時是 `0x001000000 >> 10 = 16384`;B、C 分別是 32768、49152。設成 32 時得到的就是 32 bit 的 gNB ID(A = 1048576)。

4G 走 S1AP:取 28 bit ECI 的高 20 bit,也就是 eNB ID。srsENB 的 `enb_id=0x19B` → 篩選值 411。4G 與 5G 的 session 在同一張表裡,同一個設定檔可以同時寫兩種 cell。

引擎學到的 session 可從 Studio 的 MEC 表或 `GET /grism/task/get_s1ap_items` 看到:每個 UE 的位址、gNB、`cellid`、兩方向 TEID。設定 cell 策略前先用它確認 cell 值。

## 測試結果

**基本流量**(未限制 cell,三個 UE 各自上網):每個 UE 經隧道做 HTTP 204、HTTPS 200(85 KB,0.12 s)、DNS、ping 8.8.8.8 約 8.6 ms,全部成功;GRISM 在 P4 側錄 6 秒看到 164 個 GTP-U 與 NGAP 的 SCTP heartbeat,正是經過 inline 鏈結的 N2/N3。

**MEC breakout**(未限制 cell):三個 UE `curl http://10.45.255.254/` 都拿到邊緣頁面(HTTP 200,首次 1.07 s 含 ARP,之後 5 ms)。在 ens21 抓到的是去掉 GTP 的純 IP(`10.45.0.3 → 10.45.255.254:80`),邊緣主機對 10.45.0.3 的 ARP 由 GRISM 代答;核心網的 `ogstun` 對邊緣位址看到 0 個封包。篩選器計數 F1 198/201、F3 16/198、F2 3/19。

**cell 策略**(三個 UE 同一 PLMN、同一位址範圍,只差所在 cell):

| 設定 | UE1(cell A) | UE2(cell B) | UE3(cell C) |
|---|---|---|---|
| F4 = 16384(只放行 A) | 邊緣 HTTP 200 | 逾時 | 逾時 |
| F4 = 32768(只放行 B) | 逾時 | 邊緣 HTTP 200 | 逾時 |

兩種設定下三個 UE 上網都正常;F4 的計數只在放行的那個 cell 的封包上命中。把 `nrCellIdentityBits` 改成 32、F4 改成 1048576,結果相同。

**4G / NSA**:srsUE attach 成功(MME「Attach complete」)、EN-DC 建立(NR 重設定與 NR 隨機存取各 3 次),UE 經 S1-U 上網正常。GRISM 的 session 表多出一筆 4G 的:`UE 10.45.0.7 | 基地台 10.5.0.21 | MME-UE-S1AP-ID 7 | eNB-UE-S1AP-ID 2 | cellid 411`;eNB 的 UE 編號與 gNB 的編號相同時(都從 1、2 開始)也各自正確,因為表是用「編號 + 基地台位址」當 key。S1AP 抓包:InitialUEMessage、InitialContextSetupRequest(SGW-U 10.5.0.3、TEID 0x2d75)、ICS Response 與 UplinkNASTransport 合併在同一個 SCTP 封包的三個 DATA chunk,都解析正確、沒有分段。

| 設定 | LTE UE(cell 411) | UE1(cell A) | UE2(cell B) |
|---|---|---|---|
| F4 = 411(只放行 LTE cell) | 邊緣 HTTP 200 | 逾時 | 逾時 |
| F4 = 16384(只放行 cell A) | 逾時 | 邊緣 HTTP 200 | 逾時 |

## 換手:用合成的 NGAP 訊令測 Xn 與 N2 handover

UERANSIM 與 srsRAN 4G 都不會把 UE 從一個基地台換到另一個,所以換手無法用真 UE 測。改用 `/opt/5g/load/mecho.py`:它以 pycrate 的 NGAP 定義組出換手的每一則訊息(`mecho.py check` 把每一則再解碼一次,並對照 pycrate 的 NGAP-Constants 核對 procedure code 與 IE id),連同 SCTP、IP、乙太網路標頭一起用 raw socket 從 `ran` namespace 的 ens19 送出,所以和真 gNB 的訊令一樣從 P4 進 GRISM。核心網那一端不必存在:合成的 gNB 在 10.6.0.0/16、AMF 與 UPF 在 10.7.0.0/24,主機對這兩個網段設 blackhole 路由,訊息只是為了讓 GRISM 看見。

```
VM [ran namespace]                                   GRISM G8S-MEC
  ens19 ── raw socket:                                 P4 ──┐ inline chain
    gNB g   10.6.x.y   cell 65536+g   (來源)                  │ P4 <-> P5
    gNB g+1 10.6.x.y+1 cell 65537+g   (目標)                  │
    AMF/UPF 10.7.0.250 (只存在於封包裡)                 P5 ──┘
  主機:blackhole 10.6.0.0/16、10.7.0.0/24;從管理網路讀 get_s1ap_items
```

每個 session 先照一般註冊的順序建立(InitialUEMessage → PDUSessionResourceSetupRequest/Response → 一個上行 G-PDU),再從來源 gNB g 換到 gNB g+1——另一個位址、另一個 cell,目標 gNB 自己編 RAN-UE-NGAP-ID:

| 程序 | 訊息(方向) | GRISM 應做的事 |
|---|---|---|
| Xn 換手(path switch) | PathSwitchRequest(目標 gNB → AMF:新 RAN id、來源 AMF id、UserLocationInformation 裡的新 cell、新的下行隧道)→ PathSwitchRequestAcknowledge(AMF → 目標 gNB:新 AMF id、上行隧道)→ 目標 gNB 的上行 G-PDU | 兩則訊息進 path-switch 暫存表,配對後換成正式節點 |
| N2 換手(經核心網) | HandoverRequest(AMF → 目標 gNB:AMF id、上行隧道)→ HandoverRequestAcknowledge(目標 → AMF:新 RAN id、下行隧道)→ HandoverNotify(目標 → AMF:新 cell)→ 來源 gNB 的 UEContextReleaseComplete → 目標 gNB 的上行 G-PDU | handover 暫存表配對後換成正式節點,HandoverNotify 設 cell,來源節點釋放 |

`--verify` 在換手前後各讀一次 `get_s1ap_items?ue-ipv4=<UE>`;一個 session 算通過的條件是表裡只剩一個節點,而且它的基地台位址(`ran-ipv4`)、`cellid`、RAN id 都是目標 gNB 的。

```bash
python3 /opt/5g/load/mecho.py check                       # 編碼自檢,不送任何東西
python3 /opt/5g/load/mecho.py xn --count 200 --verify     # 200 個 session 做 Xn 換手
python3 /opt/5g/load/mecho.py n2 --count 200 --verify     # 同一批做 N2 換手
```

要在主機上執行,不要包在 `ip netns exec ran` 裡:程式只在開 raw socket 時進入 `ran` namespace(socket 建立後就跟著那個 namespace),讀 GRISM 的表則從主機,管理網路的路由在那裡。它只需要 `ran` namespace 與那兩條 blackhole 路由,測試床的其他部分可以不啟動。

**結果。** 修正前兩種換手都是 0/200:換手後的節點掛在「RAN id + 基地台位址 0」底下、沒有 cell,換手後的 UE 對 cell 策略(F4)不再命中,breakout 就失效。修正後(引擎 2.11.23 / 2.9.26 起)Xn、N2 各 200/200;同一批訊框寫成 pcap、以 `submit_instant` 的 `replayPcap` 輸入從 LOOP 埠回放,在一台不在測試床上的 HL1 也是 100/100,所以任何機器都能這樣測。修的東西有三件:基地台位址沒有跟著 session 走過換手暫存表;5G 完全沒有解析 HandoverNotify,PathSwitchRequest 裡的 UserLocationInformation 也沒讀,換手後只有 4G 會更新 cell;NGAP 的分派在 PathSwitchRequest 上又做了一次一般的新增,留下一個「來源 AMF id 配目標 RAN id」的節點。

**要看 GRISM 對每一則 MEC 訊息做了什麼**,用 `submit_config` 把 `<args>` 裡的 `s1apItemsUpdateLog` 設成 `True`,並把一個 syslog 目標的 `<type>` 設成 `mec`(這個型別設定頁沒有列出,用 `get_config` 匯出目前的紀錄設定改一個目標即可):表的每一次新增、配對、換手、釋放都會按順序送到該目標。追換手類問題時這比抓包快得多。

**沒有模擬的:** 4G 的 X2/S1 換手(S1AP 的 PathSwitchRequest 與 HandoverNotify 走同一套表,4G 一向有解析,但沒有用合成訊令跑過);5G 的 PDUSessionResourceModify / ModifyIndication 目前不解析,改變隧道的那種修改 GRISM 不會跟到。

## 在 GRISM 做 breakout,相較於邊緣 UPF 的優勢

3GPP 的標準做法是在邊緣部署一個 UPF(5G 用 UL-CL/branching point 分流,4G 用本地 S-GW/P-GW 或 LIPA/SIPTO),由 SMF 下 N4 規則決定哪些流量就地落地。GRISM 的做法是把一台 inline 的封包處理設備放在基地台與核心網之間,從訊令學會每個 UE 的 cell 與隧道,再依設定把特定流量解封裝後交給邊緣。兩者的差別:

| | 邊緣 UPF | GRISM MEC breakout |
|---|---|---|
| 對核心網的要求 | SMF/PCF 要支援 UL-CL、DNAI、流量導向規則,並接受新的 UPF(N4),通常是一個與電信商的整合專案 | **核心網零改動**:不需 N4、不需 SMF 認識它,任何廠牌的核心網都適用 |
| 4G / 5G / NSA | 各代做法不同(本地 UPF vs. 本地 S/P-GW),NSA 還要看錨點在哪 | 同一台設備同時處理 S1AP 與 NGAP,4G、5G、NSA 一套設定 |
| 分流條件 | 以 DNN/切片/UE 位址為主,cell 層級的條件要靠 PCF 與位置回報 | 直接用 cell、基地台、UE 位址範圍、目的地等封包層的條件組合,也能與其他篩選器(L4–L7、IMSI 等)AND/OR |
| 部署位置與延遲 | 要有一個 UPF 的落點,流量在 UPF 內再做一次 GTP 處理與路由 | 在聚合點就地去掉 GTP,以 ARP 解析邊緣 MAC 後直接 L2 交付,路徑上少一跳 |
| 失效行為 | UPF 故障即斷服務 | inline 設備故障時可由 bypass 繼電器直通(G8S/T12S/G8 皆有),或鏈結設定成預設照走核心網 |
| 可見度 | 需另外佈署探針 | 同一台設備就有每個 UE session 的上下行封包/位元組(`get_s1ap_items`)、篩選器計數、即時側錄、syslog |
| 建置時間 | 數週到數月 | 本文的整套設定是一份 40 行的 XML |

要誠實說明的限制:被 breakout 的流量不經過核心網,因此核心網的計費與合法監聽看不到它(這通常正是企業專網或校園網要的,但電信商環境要先談清楚);GRISM 必須同時看得到控制面與使用者面(N2 與 N3、S1-MME 與 S1-U 走同一台設備);UE 在基地台之間換手時,靠的是 GRISM 的 path switch / handover 追蹤表,而不是標準的 session 錨點。UE 的 IP 是從使用者面學的、cell 是從 NGAP/S1AP 的 IE 讀的,所以 NAS 加密與否都不影響。

## 操作備忘

- 整套用 systemd 管理:接線(oneshot)→ `open5gs-amfd/upfd`(drop-in 讓它們等接線完成,否則綁不到 10.5.0.2)→ `ueransim-gnb@A/B/C` → `ueransim-ue@1/2/3`、`srsenb` → `srsue` → 流量產生器。UERANSIM 的 gNB 開機時只連一次 AMF,連不上不會重試,用一個每分鐘檢查「沒完成 NG Setup 就重啟」的 timer 補上;srsUE 要在 eNB 之後啟動。
- `journalctl -u 'ueransim-*' -f` 看註冊與 PDU session;`get_s1ap_items` 看 GRISM 學到的 session;介面流量頁看 P3/P4/P5。
- 切換策略就是重送 run.xml(`submitxml`),引擎重載約一秒,UE 的 session 不受影響。
- 換手只能用合成訊令測(`mecho.py xn|n2 --verify`,見上),塞表用 `mecfill.py`(InitialUEMessage + PDU session 建立 + 一個 G-PDU,每秒數千個 session);兩者都只要 `ran` namespace 與 blackhole 路由。
