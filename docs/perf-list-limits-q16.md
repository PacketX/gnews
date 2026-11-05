# Q16 大量過濾條件的筆數上限

一個 filter 裡放大量的 `ip.addr`(單一位址或 CIDR)、`ipv6.addr`、`dns.qry.name`,Q16 能放幾筆?本頁是在一台 Q16(DPDK 系列)上,以 7.6.261104 版韌體、出廠設定實測的結果。G8S 的結果在[這裡](perf-list-limits.md),T12S 的在[這裡](perf-list-limits-t12s.md);三台的瓶頸各不相同。

## 這台機器的記憶體怎麼分

Q16 和 MIPS 機型最大的差別是清單放的地方不只一個:

- **hash 表(`dns.qry.name`、單一位址的 `ip.addr`)放在 hugepage。** 機器有 48 GB RAM,開機時保留 10 GB 的 hugepage(20 頁 × 512 MB)給封包引擎;引擎自己的 flow table 等用掉約 4.5 GB,剩下 **約 5.5 GB(11 頁)** 給清單。每張 hash 表要一塊連續的空間。
- **樹(`ip.addr` CIDR、`ipv6.addr`)放在引擎程序的一般記憶體**,不佔 hugepage。扣掉 hugepage 與其他程式,大約 20 GB 可用。
- **上傳的清單放在 `/data`**,一個 4 GB 的 tmpfs(也是 RAM):上傳中的檔、running-config 裡的那一份,以及轉出來的 GDP 檔都在這裡。

## 測試環境

| 項目 | 設定 |
|---|---|
| 機型 | **Q16**(16 個 1G/10G 埠,前端有 Marvell 交換器) |
| 韌體 | 7.6.261104.2 |
| IPv4 flow table | 出廠值(`flowCacheBaseSize` 23000) |
| IPv6 flow table | 出廠值(`flowv6TableSize` 30000) |
| `tryRunXmltoGdp` | 開啟(即時載入;這版起預設就是開) |
| 其他設定 | 出廠設定,MEC 關閉,`run.xml` 為空,沒有流量;閒置時 hugepage 剩 11 頁 |

## 測試方法

和 G8S 那頁相同:每種條件一份 `run1.xml`、一個 filter、一行一筆,內容為隨機的公網 IPv4、隨機 /24、`2001:db8::/32` 下的隨機 /128、約 24 字元的網域名稱;用 scp 把新檔覆蓋到 `ftproot` 再放 `.ok`(和網頁的 submitxml 做的事相同),**不先清掉前一份**。同一個大小要連續上傳三份不同內容、每次都換成新清單、引擎不重啟、log 沒有錯誤,**然後完整 reload(空的 instant 設定)和重開機都還保有清單**才算通過;每換一個大小先刪掉清單並重開機。工具是韌體樹的 `tools/test/gdp_tree/limits.py` 與 `limits_search.py`(×1.5 往上找到失敗,再二分到 15% 以內;單一位址與 CIDR 設了上限就不再往上)。Q16 重開機約 2 分鐘。

## 單一種類的結果

| 條件 | 上限(三次更新、完整 reload、重開機都成功) | 失敗的大小 | XML 大小 | 卡在哪裡 |
|---|---|---|---|---|
| `ip.addr` 單一位址 | **至少 10,000,000**(測到這裡為止) | — | 611 MB | 沒碰到:hash 表只有 107 MB,/data 還有一半 |
| `ip.addr` CIDR | **至少 12,000,000**(測到這裡為止) | — | 752 MB | 沒碰到:樹約 2.9 GB,放在一般記憶體 |
| `ipv6.addr` | **2,200,000** | 3,000,000 | 193 MB | Linux 記憶體:兩棵樹合計約 29 GB,**機器記憶體耗盡,管理服務被殺掉**(見下) |
| `dns.qry.name` | **7,600,000** | 8,400,000 | 578 MB | hugepage:兩張 2.9 GB 的表放不進 5.5 GB |

每筆佔的空間:

| 條件 | 放在哪 | 每筆 | 例 |
|---|---|---|---|
| `ip.addr` 單一位址 | hugepage hash table | 8 B × 4/3 | 10,000,000 筆約 107 MB |
| `dns.qry.name` | hugepage hash table | 260 B × 4/3 ≈ 347 B | 7,600,000 筆 2.6 GB;8,400,000 筆 2.9 GB;10,000,000 筆 3.5 GB |
| `ip.addr` CIDR | 一般記憶體的樹 | 隨機 /24 約 240 B | 12,000,000 筆約 2.9 GB |
| `ipv6.addr` /128 | 一般記憶體的樹 | 隨機 /128 約 4.3 KB | 2,200,000 筆約 9.4 GB;3,000,000 筆約 12.9 GB |

- **hugepage 的規則:使用中的每張 hash 表,再加上正在更新的那張的第二份,要一次放進 5.5 GB。** 即時載入會先建好「使用中的清單 + 新清單」再切換,所以一張 dns 表最大約 2.6 GB(7,600,000 筆);2.9 GB 的第二份就配不到(`failed to alloc from bootmem named FCDATA_HASH_TABLE_0_9101 size: 2912000000`),log 記 `GDP reload failed: staying on hash table N`,舊清單繼續生效,機器正常。
- **一般記憶體的規則:使用中的樹,再加上正在更新那棵的第二份,要放進約 20 GB。** 兩棵 9.4 GB 的 IPv6 樹(2,200,000 筆)可以;3,000,000 筆時第二次上傳要建第二棵 12.9 GB 的樹,**沒有被拒絕,而是把 Linux 的記憶體用光**:OOM killer 把 sshd、nginx、pywww、snmpd 都殺了,封包引擎還在轉送,但 ssh 和網頁都連不上,只能斷電重開。這和 MIPS 機型不同——MIPS 的樹在 bootmem,配不到就失敗保留舊清單;Q16 的樹是一般的 malloc,Linux 會先答應再在用到時出事。**7.6.261105 起的引擎(2.9.45)會先算出樹的大小,可用記憶體不夠(保留 2 GB 給其他程式)就拒絕,log 記 `filter N: its M-prefix tree would take X GB ... not built` 與 `GDP reload failed: staying on hash table N`,舊清單保留、機器正常**(實測:第一棵 3,000,000 筆的樹 12.05 GB 建好,第二棵在剩 7.5 GB 時被拒)。在那之前的版本,**隨機的 IPv6 單一位址清單請控制在 2,000,000 筆以內**;整段網段寫成 CIDR 便宜得多。
- **`/data` 的規則:兩份 XML(上傳中的與 running-config 裡的)要同時放進 4 GB**;本頁的大小都離這條很遠。
- 幾種清單同時放時三條規則都要成立,下面的小工具把它算出來。

## 同時放幾種清單

小工具用本頁量到的單筆成本計算:hash 表(dns、單一位址,合計 H)在 hugepage 裡要放得下 2H;樹(CIDR、IPv6,合計 T)在一般記憶體裡要放得下 T 加上正在更新那棵的第二份;完整 reload 或開機則是 H 與 T 各一份。拉動四種的筆數,每一列顯示其他維持不動時這一種最多能放幾筆,表格列出每個動作的餘裕,兩條長條分別是 hugepage 與一般記憶體的用量。Q16 只量了單一清單,組合是模型推算;樹的大小取決於網段怎麼分佈,自己的清單可以在「模型參數」裡改每筆 bytes。

<iframe src="tools/list-limits-66.html?model=q16" data-autoheight style="width:100%;height:900px;border:1px solid #e5e8eb;border-radius:6px" title="清單同時放的上限(Q16)"></iframe>

## 失敗時會發生什麼

1. **hugepage 放不下(dns、單一位址)。** log 出現 `failed to alloc from bootmem named ...` 和 `GDP reload failed: staying on hash table N`,新清單沒有生效,**舊清單繼續生效**,系統正常;告警規則「過濾清單未套用」會發出。
2. **一般記憶體放不下(CIDR、IPv6 的樹)。** 7.6.261104 與之前:沒有安全的失敗,記憶體被用光,管理服務被 OOM killer 結束,需要斷電重開。7.6.261105 起:建樹前先算,放不下就和 hugepage 一樣失敗、舊清單保留、告警。無論哪一版,請用上面的小工具先算,不要讓兩棵樹加起來接近 20 GB。
3. **開機或完整 reload 時放不下。** 每份清單從 GDP 重新載入,hugepage 放不下的那份 log 記 `run1.xml.gdp load fail ... not in effect`,**那個 filter 是空的**,其他清單不受影響。

## 載入要多久

| 動作 | 時間 |
|---|---|
| `dns.qry.name` 7,600,000 筆即時載入 | 上傳 5 s + 生效 45 s |
| `ip.addr` CIDR 12,000,000 筆即時載入 | 上傳 7 s + 生效 84 s |
| `ipv6.addr` 2,200,000 筆即時載入 | 上傳 2 s + 生效 28 s |
| `ip.addr` 10,000,000 筆即時載入 | 上傳 6 s + 生效 45 s |
| 完整 reload(帶著 7,600,000 筆 dns) | 40 s |
| 重開機 | 約 2 分鐘;清單在開機時從 GDP 重建(3,000,000 筆 IPv6 的樹 8 s) |

## 其他注意事項

- 上傳大檔請用 scp 放到 `/data/ftproot/run1.xml` 再 `touch run1.xml.ok`;網頁的 submitxml 要把整份 XML 當表單欄位送,幾百 MB 的檔不適合。
- 清單會存在 `/usr/local/grism/etc/running-config/`,重開機時再載入一次;一份開機時放不下的清單,每次開機都會再試一次。
