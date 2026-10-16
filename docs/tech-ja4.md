# JA4 指紋:GRISM 怎麼算、怎麼驗證、踩過哪些坑

## JA4 是什麼

JA4 是 FoxIO 定義的 TLS 用戶端指紋,從 ClientHello 的結構算出來,不看憑證、不看內容,同一個程式(瀏覽器、惡意程式、函式庫)不論連到哪裡都算出同一個值,所以能用來辨認用戶端。格式是三段:

```
t13d1517h2_8daaf6152771_cb7bf5808d99
│ │ ││ │ └ ALPN 的第一個與最後一個字元
│ │ ││ └── 擴充欄位數(17)
│ │ │└──── cipher 數(15)
│ │ └───── d = 有 SNI,i = 沒有
│ └─────── TLS 版本(13 = 1.3)
└───────── 協定:t = TCP,q = QUIC
          ┌ JA4_b:cipher 清單排序後的 SHA-256 前 12 碼
          └ JA4_c:擴充欄位清單排序(去掉 SNI 與 ALPN)+ signature_algorithms 原序,SHA-256 前 12 碼
```

JA4S 是對應的伺服器端指紋(ServerHello):`t130200_1301_a56c5b993250` — 協定、版本、擴充數、ALPN、選定的 cipher、擴充清單雜湊。

## GRISM 提供什麼

| 篩選欄位 | 內容 |
|---|---|
| `tls.handshake.ja4` | 完整 JA4 |
| `tls.handshake.ja4_a` / `ja4_b` / `ja4_c` | 三段各自比對 — 例如只比 `ja4_b` 就是「同一組 cipher 的所有用戶端」 |
| `tls.handshake.ja4s`、`ja4s_a/b/c` | 伺服器端 |
| `ssl.ja3_digest`、`ssl.ja3s_digest` | 舊的 JA3,仍在 |

DPI 的 SSL syslog 每個連線帶 `ja4=`、`ja4s=`,所以不寫篩選器也能把整個網路的指紋分布送到 SIEM 統計。

## 怎麼驗證:一份不共用程式碼的參考實作

韌體的 JA4 不能拿韌體自己來驗。`tools/test/ja4.py` 是從頭寫的參考計算器:自己解 pcap、Ethernet/VLAN/IP/TCP、重組每個方向的 TCP、解 TLS record 與 ClientHello,只依據 FoxIO 的規格,一行程式碼也沒有跟設備共用,並以規格裡的範例自我檢查。它印出每個 ClientHello 的 JA4 與 **JA4_r**(未雜湊的原始清單),所以一旦不符,能追到是哪個欄位算錯,而不是猜。

驗證流程:把瀏覽器的封包檔用 `replayPcap` 從 LOOP 埠重播,DPI SSL syslog 指到測試主機,把 syslog 裡的 `ja4=` 逐連線和 `ja4.py` 的答案比對。`tools/test/mkhello.py` 則能合成只有 ALPN 不同的 ClientHello(含二進位 ALPN、超長名稱、`--ccs` 在前面加 ChangeCipherSpec),用來逐條驗規則。

## 2026-10-03:三個偏差

這樣一比,原本的實作有三處和規格不同,每一處都讓某一類用戶端的指紋**永遠**錯:

| 偏差 | 影響 | 修法 |
|---|---|---|
| 只看 TCP segment 裡的第一個 TLS record。TLS 1.3 的 middlebox 相容模式(RFC 8446 D.4)會在 HelloRetryRequest 之後把 ClientHello 放在一個 6 byte 的 ChangeCipherSpec record 後面 | 這種 hello 完全沒算 | 跳過前面的 CCS record 找 handshake record;前面若是 Alert 或 Application Data 就確定沒有 hello |
| JA4_c 的 signature_algorithms 沒有去掉 GREASE。Chrome 把 GREASE 值放在這個清單的**第一個**(`dada,0904,0905,…`) | 所有 Chrome 系用戶端的 `ja4_c` 與規格、Wireshark、任何公開資料庫都對不上 | 規格說 GREASE 到處都要忽略,這裡也跳過 `0x?a?a` |
| ALPN 的兩個字元來自一張已知名稱表(h2、dot、http/x、spdy/x),其他全部算 `00` | 不在表上的 ALPN 都錯 | 改用規格的規則:第一個 ALPN 值的首尾兩個 byte,其中一個不是英數字時改用首尾兩個十六進位數字;第一個 ALPN 連長度保留在 256 byte 緩衝區,二進位與超過 23 byte 的名稱不再掉成 `00` |

在 Q16 上以重播驗證:3 個 CCS 前綴的 hello、GREASE sigalgs、9 個合成 ALPN 案例全部與參考實作相同。

## 2026-10-06:每次錯在不同連線

修完之後 arm64 機種還是每 100 個 Chrome hello 有 1 個錯,但**每次錯的是不同連線** — 這不是解析錯,是時序。原因是拆 cipher/擴充清單用的 `strtok()` 在整個行程只有一個「切到哪裡」的指標,DPDK 的工作核心是同一個行程的執行緒,兩個核心同時算指紋就互相把對方的清單切短。改用 `strtok_r()` 之後,重播 1365 個 hello 零錯誤(之前 546 個錯 3 個)。完整的分析在〈[多核心下的 strtok](tech-strtok.md)〉。

## QUIC:ClientHello 是加密的

HTTP/3 走 QUIC,TLS 握手藏在 QUIC Initial 封包的 CRYPTO frame 裡,整個 Initial 的內容用 AEAD 保護。好消息是金鑰任何人都推得出來 — 它只由封包裡的 Destination Connection ID 經 HKDF 推導(RFC 9001 §5.2),目的是防止中間設備篡改而不是保密。所以要算 QUIC 的 JA4,設備得:去除 header protection(§5.4)、推導 Initial 金鑰、AEAD 解密(§5.3)、把 CRYPTO frame 依 offset 重組(RFC 9000 §19.6)。

重組是必要的,不是可選的:**每個瀏覽器的 hello 都跨 2–3 個 datagram,而且順序是亂的**。後量子金鑰交換讓 ClientHello 超過一個 datagram;Chrome 把一個 hello 切成十幾個亂序的 CRYPTO frame(連 4 byte 的 handshake 標頭都被切開),Firefox 先送尾巴。Firefox 還會在 Initial 的 Length 之後用零補滿 datagram — 合法,不是壞封包。

GRISM 的做法(設定 `quicInitialReassemble`,預設關,和其他重組開關放在一起):解密用戶端的 Initial、按 offset 收集 CRYPTO frame 直到 hello 完整,然後交給 TCP 那條路徑同一段程式。於是 `tls.handshake.ja4*`、`ssl.server_name`、`quic.tag == CHLO` 與 SSL syslog 都看得到 QUIC,JA4 帶 `q` 前綴。加解密是可攜的 C(`quic_initial.h`),三種後端:MIPS 機種用 OCTEON 核心的 AES/GFM 硬體單元,arm64 機種用 OpenSSL(用到 ARMv8 Crypto Extensions),另有純軟體參考版供主機端測試。QUIC v1 與 v2(RFC 9369)都支援。

**QUIC 的 JA4 和同一個瀏覽器的 TCP JA4 不一樣**,不要拿 TCP 的值去篩 QUIC。QUIC 只有 TLS 1.3,瀏覽器只提 3 個 cipher(`ja4_b` = `55b375c5d22e`)而不是 15 個(`8daaf6152771`):

| 瀏覽器 | QUIC JA4 |
|---|---|
| Firefox | `q13d0315h3_55b375c5d22e_dc5437974b47` |
| Chrome(新連線) | `q13d0312h3_55b375c5d22e_178839b6cec1` |
| Chrome(resumption) | `q13d0314h3_55b375c5d22e_79cc91d6b50c` |

同一個 Chrome 走 TCP 是 `t13d1517h2_8daaf6152771_cb7bf5808d99`:協定、cipher 數、擴充數、ALPN(h2 對 h3)、三段雜湊全都不同。

驗證同樣靠一份獨立實作:`tools/test/quic_ja4.py` 用 Python `cryptography` 解密 pcap 裡每個連線的 Initial、重組、算 JA4,也能輸出逐步的 test vector(金鑰、每個 datagram 的明文與 frame)。`quic_initial_test.c` 在主機上跑這些 vector 加上 RFC 9001/9369 附錄的樣本與 FIPS/HKDF 的公開向量。設備上:Q16 重播 89 個 Firefox 與 25 個 Chrome 連線,JA4 與 oracle 100% 相同;G8S 的值也全部正確,但經 LOOP 埠重播會掉幾個百分比的 hello — 同一連線的 Initial 要在同一核心集合,MIPS 機種靠把 QUIC long header 像 TCP segment 一樣 deschedule 到該流的核心做到這點,而 LOOP 埠做不到;實體埠沒有這個問題。

## 工具一覽

| 工具 | 用途 |
|---|---|
| `tools/test/ja4.py <pcap>` | TCP ClientHello 的 JA4 / JA4_r 參考答案 |
| `tools/test/quic_ja4.py <pcap> [--json out]` | QUIC 的參考答案,`--vectors` 產生 C 測試向量,`--selftest` 自檢 |
| `tools/test/mkhello.py out.pcap h2 http/1.1 hex:0102 none [--ccs]` | 只有 ALPN 不同的合成 hello |
| `tools/test/quic_initial_test.c` | 主機端的解密測試(`gcc … sha256.c`;加 `-DQUIC_CRYPTO_OPENSSL -lcrypto` 可測 OpenSSL 後端) |

兩個參考實作刻意不和設備共用任何程式碼,也彼此只共用 JA4 的字串組合 — 這是它們能當裁判的原因。
