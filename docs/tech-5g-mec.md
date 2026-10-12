# 用 UERANSIM + Open5GS 架 5G 測試環境,驗證 GRISM 的 MEC breakout

## 目的

在一台 Linux VM 上同時跑 5G 核心網(Open5GS)與基地台/手機模擬器(UERANSIM),讓 N2(NGAP over SCTP)與 N3(GTP-U)**真的走實體線**經過 GRISM 的 inline 鏈結,再接一台邊緣主機,驗證 MEC breakout:指定 cell 的 UE 存取邊緣服務時,GRISM 去掉 GTP 直接送到邊緣,回程再封裝回 gNB;其餘流量照走核心網,核心網完全看不到被 breakout 的部分。

本文用的機器:GRISM G8S-MEC(P4 接 gNB 側、P5 接核心網側、P3 接邊緣主機)與一台 Ubuntu 24.04 VM(8 核、16 GB,四張網卡)。

## 架構

```
VM                                                        GRISM G8S-MEC
┌─────────────────────────────────┐
│ [ran namespace]                 │
│   ens19  10.5.0.1  gNB A        │───── P4 ──┐
│          10.5.0.11 gNB B        │           │  inline chain
│          10.5.0.12 gNB C        │           │  P4 <-> P5
│          UE1 UE2 UE3 (uesimtunN)│           │
│                                 │           │
│ [主機]                           │           │
│   ens20  10.5.0.2  AMF(N2) UPF(N3)│───── P5 ──┘
│   ogstun 10.45.0.1/16 ─ NAT ─ ens18 ─ 網際網路
│                                 │
│ [edge namespace]                │
│   ens21  10.45.255.254  HTTP    │───── P3 (邊緣,breakout 出口)
└─────────────────────────────────┘
```

| 介面 | namespace | 位址 | 角色 | GRISM 埠 |
|---|---|---|---|---|
| ens19 | `ran` | 10.5.0.1 / .11 / .12 | 三個 gNB 的 N2/N3 位址,UE 也在這裡 | P4 |
| ens20 | 主機 | 10.5.0.2 | Open5GS AMF 的 NGAP、UPF 的 GTP-U | P5 |
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

## 操作備忘

- 整套用 systemd 管理:接線(oneshot)→ `open5gs-amfd/upfd`(drop-in 讓它們等接線完成,否則綁不到 10.5.0.2)→ `ueransim-gnb@A/B/C` → `ueransim-ue@1/2/3` → 流量產生器。UERANSIM 的 gNB 開機時只連一次 AMF,連不上不會重試,用一個每分鐘檢查「沒完成 NG Setup 就重啟」的 timer 補上。
- `journalctl -u 'ueransim-*' -f` 看註冊與 PDU session;`get_s1ap_items` 看 GRISM 學到的 session;介面流量頁看 P3/P4/P5。
- 切換策略就是重送 run.xml(`submitxml`),引擎重載約一秒,UE 的 session 不受影響。
