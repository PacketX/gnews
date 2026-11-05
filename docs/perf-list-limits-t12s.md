# T12S 大量過濾條件的筆數上限

一個 filter 裡放大量的 `ip.addr`(單一位址或 CIDR)、`ipv6.addr`、`dns.qry.name`,T12S 能放幾筆?本頁是在一台 T12S 上,以 6.6.261104 版韌體、出廠的 flow table 實測的結果。G8S 的結果在[這裡](perf-list-limits.md),Q16 的在[這裡](perf-list-limits-q16.md)。

## 這台機器的記憶體怎麼分

T12S 和 G8S 是同一條產品線(MIPS),清單放的地方相同——bootmem 裡的 hash 表與樹——只是大得多:

- **DRAM 32 GB**,Linux 分到 4 GB,其餘是封包引擎的 bootmem。出廠的 flow table(IPv4 `flowCacheBaseSize` 150000、IPv6 `flowv6TableSize` 500000)用掉 4.6 GB,量到**可供清單使用的約 13.8 GB**(以兩棵 IPv6 樹同時存在的最大值回推)。每張表、每棵樹都要一塊連續的空間。
- **上傳的清單放在 `/data`**,一個 2 GB 的 tmpfs(也是 Linux 的 RAM):上傳中的檔、running-config 裡的那一份,以及轉出來的 GDP 檔都在這裡。開機後 Linux 剩約 2.7 GB。

## 測試環境

| 項目 | 設定 |
|---|---|
| 機型 | **T12S**(MEC 關閉) |
| 韌體 | 6.6.261104.3 |
| IPv4 flow table | 出廠值(`flowCacheBaseSize` 150000,×16 = 2,400,000) |
| IPv6 flow table | 出廠值(`flowv6TableSize` 500000) |
| `tryRunXmltoGdp` | 開啟(即時載入;這版起預設就是開) |
| 其他設定 | `run.xml` 為空,沒有流量 |

## 測試方法

和 G8S 那頁相同:每種條件一份 `run1.xml`、一個 filter、一行一筆,內容為隨機的公網 IPv4、隨機 /24、`2001:db8::/32` 下的隨機 /128、約 24 字元的網域名稱;用 scp 把新檔覆蓋到 `ftproot` 再放 `.ok`(和網頁的 submitxml 做的事相同),**不先清掉前一份**。同一個大小要連續上傳三份不同內容、每次都換成新清單、引擎不重啟、log 沒有錯誤,然後完整 reload(空的 instant 設定)還保有清單才算通過。**帶著清單重開機的驗證做了 dns 11,000,000 與 4,800,000 筆**(開機時把清單重轉成 GDP 載入,11,000,000 筆約 4 分鐘、4,800,000 筆約 90 s,期間 CPU 100%,之後 filter 筆數完整):這台測試機的管理埠(USB 網卡)有時在開機時起不來,要人到現場斷電,所以其他三種沒有像 G8S 那頁一樣逐一重開;開機載入清單的路徑和 reload 相同,大小只影響時間。工具是韌體樹的 `tools/test/gdp_tree/limits.py` 與 `limits_search.py`(×1.5 往上找到失敗,再二分到 15% 以內;單一位址設了 10,000,000 的上限就不再往上)。T12S 重開機約 2.5 分鐘,加上開機時重轉清單的時間(見下)。

## 單一種類的結果

| 條件 | 上限(三次更新與完整 reload 都成功) | 失敗的大小 | XML 大小 | 卡在哪裡 |
|---|---|---|---|---|
| `ip.addr` 單一位址 | **至少 10,000,000**(測到這裡為止) | — | 611 MB | 沒碰到:hash 表只有 107 MB,/data 用了 583 MB |
| `ip.addr` CIDR | **至少 12,000,000**(測到這裡為止) | — | 752 MB | 沒碰到:樹約 2.9 GB,/data 用了 717 MB |
| `ipv6.addr` | **1,600,000** | 1,800,000 | 140 MB | bootmem:兩棵 7.7 GB 的樹放不進 13.8 GB |
| `dns.qry.name` | **11,000,000** | 12,000,000 | 837 MB | /data:兩份 XML 加上轉出來的 GDP 檔放不進 2 GB(`writing run1.gdp failed: No space left on device`) |

每筆佔的空間:

| 條件 | 放在哪 | 每筆 | 例 |
|---|---|---|---|
| `ip.addr` 單一位址 | bootmem hash table | 8 B × 4/3 | 10,000,000 筆約 107 MB |
| `dns.qry.name` | bootmem hash table | 260 B × 4/3 ≈ 347 B | 6,000,000 筆 1.9 GB;11,000,000 筆 3.6 GB |
| `ip.addr` CIDR | bootmem 樹 | 隨機 /24 約 242 B | 4,500,000 筆 1.0 GB;12,000,000 筆約 2.9 GB |
| `ipv6.addr` /128 | bootmem 樹 | 隨機 /128 約 4.3 KB(筆數越多越省一點:500,000 筆 4.4 KB,2,400,000 筆 4.28 KB) | 1,600,000 筆 6.9 GB;1,800,000 筆 7.7 GB;2,400,000 筆 10.3 GB |

- **bootmem 的規則和 G8S 一樣:使用中的每張 hash 表與每棵樹,加上正在更新的那一份的第二份,要一次放進約 13.8 GB。** 兩棵 6.9 GB 的 IPv6 樹(1,600,000 筆)可以;1,800,000 筆時第二棵 7.7 GB 配不到(`failed to alloc from bootmem named FCDATA_TREE_9101_1 size: 7747469312`),log 記 `GDP reload failed: staying on hash table N`,舊清單繼續生效,機器正常。其他三種在測到的大小都離這條很遠。
- **`/data` 的規則:上傳中的 XML、running-config 裡的上一份 XML,加上轉出來的 GDP 檔,要同時放進 2 GB。** dns 就是卡在這條:11,000,000 筆(XML 837 MB,GDP 約 350 MB)可以,12,000,000 筆(XML 913 MB)第二次上傳時 GDP 檔寫不進去,log 記 `writing /data/ftproot/run1.gdp failed: No space left on device -- the list in use is kept, the new one is dropped`,舊清單繼續生效;14,000,000 筆(XML 1.07 GB)連第二份 XML 都放不下,scp 就失敗了。單一位址與 CIDR 的 XML 較小,到測試的上限都還有餘裕。
- 幾種清單同時放時兩條規則都要成立,下面的小工具把它算出來。

## 同時放幾種清單

小工具用本頁量到的單筆成本計算:hash 表(dns、單一位址,合計 H)與樹(CIDR、IPv6,合計 T)都在 bootmem,更新 hash 清單需要 2H + T、更新一棵樹需要 2H + T + 那棵樹、完整 reload 或開機需要 H + T,都要放進 13.8 GB。拉動四種的筆數,每一列顯示其他維持不動時這一種最多能放幾筆,表格列出每個動作的餘裕,長條圖是四張表加上「即時更新的第二份」(斜紋)佔掉 bootmem 的情形。T12S 只量了單一清單,組合是模型推算;樹的大小取決於網段怎麼分佈,自己的清單可以在「模型參數」裡改每筆 bytes。

<iframe src="tools/list-limits-66.html?model=t12s" data-autoheight style="width:100%;height:900px;border:1px solid #e5e8eb;border-radius:6px" title="清單同時放的上限(T12S)"></iframe>

## 失敗時會發生什麼

和 G8S 相同:

1. **bootmem 放不下。** log 出現 `failed to alloc from bootmem named ...` 和 `GDP reload failed: staying on hash table N`,新清單沒有生效,**舊清單繼續生效**,系統正常;告警規則「過濾清單未套用」會發出。
2. **RAM 放不下(XML 轉 GDP 時)。** log 出現 `run1.xml: writing /data/ftproot/run1.gdp failed` 之類的 ERROR,新清單被丟掉,**舊清單繼續生效**。
3. **開機或完整 reload 時放不下。** 每份清單從 GDP 重新載入,放不下的那份 log 記 `run1.xml.gdp load fail ... not in effect`,**那個 filter 是空的**,其他清單不受影響。

## 載入要多久

| 動作 | 時間 |
|---|---|
| `dns.qry.name` 11,000,000 筆即時載入 | 上傳 40 s + 生效 270 s(6,000,000 筆:22 s + 148 s) |
| `ip.addr` CIDR 12,000,000 筆即時載入 | 上傳 36 s + 生效 330 s(8,000,000 筆:24 s + 219 s) |
| `ipv6.addr` 1,600,000 筆即時載入 | 上傳 7 s + 生效 72 s |
| `ip.addr` 10,000,000 筆即時載入 | 上傳 29 s + 生效 196 s |
| 完整 reload(帶著 11,000,000 筆 dns) | 244 s(6,000,000 筆:139 s;12,000,000 筆 CIDR:311 s) |
| 重開機 | 約 2.5 分鐘,再加上開機時把清單轉成 GDP 載入的時間:4,800,000 筆 dns 約 90 s、11,000,000 筆約 4 分鐘,期間 CPU 100%(封包引擎還沒開始轉送) |

## 其他注意事項

- 上傳大檔請用 scp 放到 `/data/ftproot/run1.xml` 再 `touch run1.xml.ok`;網頁的 submitxml 要把整份 XML 當表單欄位送,G8S 上 11 MB 就把 pywww 撐爆過,T12S 的 Linux 記憶體較多但同樣不適合幾百 MB 的檔。
- 清單會存在 `/usr/local/grism/etc/running-config/`,重開機時再轉檔、載入一次;開機時間因此隨清單大小增加。
