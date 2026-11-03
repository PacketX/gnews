# G8S 大量過濾條件的筆數上限

一個 filter 裡放大量的 `ip.addr`(單一位址或 CIDR)、`ipv6.addr`、`dns.qry.name`,能放幾筆?本頁是在一台 G8S 上，以 6.6.261103 版韌體實測的結果。舊韌體(6.5.260910)的結果保留在[舊版頁面](perf-list-limits-6.5.md),上限和失敗的方式都不同，不要混用。

## 這版韌體改了什麼

- **IPv4 CIDR 與 IPv6 清單也即時載入。** 以前只有單一 IPv4 位址和網域名稱會轉成 GDP 即時載入;CIDR 與 IPv6 要整份重新載入，並在每個核心各建一棵樹(放在 Linux 記憶體,G8S 有 4 個核心)。現在這兩種也轉成 GDP,所有核心共用一棵放在 bootmem 的樹。
- **完整 reload 和開機不再逐核心解析清單。** 修改 `run.xml`、送出 instant 設定、重開機，都會重新載入所有設定;以前每份 `runN.xml` 會在每個核心各解析一次，現在能即時載入的清單一律從 GDP 載入。
- **轉換大檔時省一份記憶體。** XML 轉 GDP 時，以前把整個檔案再複製一份到解析器的緩衝區，現在每次只交 1 MB。
- **放不下時會記錄，不再無聲退回。** 轉檔寫入失敗會在 log 記 ERROR 並保留使用中的清單;搬檔不再經過 `system("mv")`,以前記憶體吃緊時它會悄悄失敗，完整 reload 後退回上一份清單(2,100,000 筆 ip.addr 在 6.6.261103.2 就是這樣，修正後通過)。

## 測試環境

| 項目 | 設定 |
|---|---|
| 機型 | **G8S**(MEC 關閉) |
| 韌體 | 6.6.261103.2 與 .3(失敗處理的修正在 .3,ip.addr 的數字是 .3 的) |
| IPv4 flow table | **160,000**(`flowCacheBaseSize` 10000,×16) |
| IPv6 flow table | **50,000**(`flowv6TableSize` 50000) |
| `tryRunXmltoGdp` | 開啟(即時載入) |
| 開機後 Linux 可用記憶體 | 約 190 MiB(舊韌體約 340 MiB:這版每個映像都帶 Python 與 nginx,pywww 多佔約 80 MiB) |
| 其他設定 | `run.xml` 為空，沒有流量 |

## 測試方法

- **一種條件一份設定檔。** 每種條件放在 `run1.xml`,裡面只有一個 filter,filter 裡全是同一種 `<find>`,一行一筆。
- **內容。** `ip.addr` 單一位址用隨機的公網 IPv4;CIDR 用隨機的公網 /24;`ipv6.addr` 用 `2001:db8::/32` 下的隨機 /128;`dns.qry.name` 用類似真實網域的名稱，平均約 24 個字元。
- **更新方式。** 把新檔用 scp 直接覆蓋舊的 `run1.xml`(放進 `ftproot` 再放 `.ok`,和網頁的 submitxml 做的事相同),**不先清掉前一份**。
- **成功的標準。** 同一個大小連續上傳三份不同內容的清單，每一次都要做到:filter 的筆數換成新清單的筆數;引擎 pid 不變，沒有 `Z` 狀態;CPU 沒有持續 100%;log 沒有 out of memory、parse error、fail;網頁 API 持續有回應。**接著還要撐過一次完整 reload(送出空的 instant 設定)和一次重開機**,清單都還在。舊版頁面沒有測這兩項。
- **每換一個大小，先清乾淨。** 刪掉 `run1.xml` 後重開機，再開始下一個大小。工具是韌體樹的 `tools/test/gdp_tree/limits.py`(單一大小)和 `limits_search.py`(二分搜尋)。

## 單一種類的結果

| 條件 | 上限(三次更新、完整 reload、重開機都成功) | 失敗的大小 | XML 大小 | 卡在哪裡 |
|---|---|---|---|---|
| `ip.addr` 單一位址 | **2,100,000** | 2,200,000 | 2,100,000 筆約 128 MB | Linux 記憶體:兩份 XML 同時在 RAM 裡,pywww 先撐不住 |
| `ip.addr` CIDR | **1,700,000** | 1,800,000 | 1,700,000 筆約 106 MB | bootmem:兩棵樹 |
| `ipv6.addr` | **92,000** | 96,000 | 92,000 筆約 8 MB | bootmem:兩棵樹 |
| `dns.qry.name` | **1,300,000** | 1,400,000 | 1,300,000 筆約 99 MB | bootmem:兩張 hash table |

每筆佔的空間:

| 條件 | 放在哪 | 每筆 | 例 |
|---|---|---|---|
| `ip.addr` 單一位址 | bootmem hash table | 8 B × 4/3 | 2,100,000 筆約 22 MB |
| `dns.qry.name` | bootmem hash table | 260 B × 4/3 | 1,300,000 筆 450 MB;1,400,000 筆 485 MB |
| `ip.addr` CIDR | bootmem 樹 | 56 B × 每個 prefix 不與前一個共用的 bit 數;隨機 /24 約 237 B | 2,100,000 個隨機 /24 475 MB |
| `ipv6.addr` /128 | bootmem 樹 | 同上;隨機 /128 約 5 KB | 110,000 個隨機 /128 475 MB;92,000 個約 400 MB |

- **bootmem 的規則：新清單的表加上所有使用中的表都要放得下。** 即時載入會先在另一個 index 建好「使用中的清單 + 新清單」再切換，所以更新一份大清單時，它的新舊兩份表同時存在。這台 G8S 的 flow table 設 10000/50000 時，單一清單的表大約到 450 MB 還能更新(dns 1,300,000),485 MB 就放不下第二份(dns 1,400,000);CIDR 的樹也在同一個範圍。每張表都要一塊連續的空間。
- **Linux 記憶體的規則：兩份 XML 要同時放在 RAM 裡。** `/data` 是 tmpfs,上傳中的新檔和 running-config 裡的舊檔都佔 RAM,轉 GDP 時再多一個 GDP 檔。單一位址的 hash table 很小，所以它卡在這條：128 MB 的 XML(2,100,000 筆)可以;134 MB 時第二次上傳途中 pywww 卡住，網頁 API 不再回應，引擎和清單都正常，但要重開機才能恢復網頁。
- 三種清單同時放時兩條規則都要成立，這次沒有像舊頁面那樣逐一組合實測。

## 失敗時會發生什麼

1. **bootmem 放不下。** log 出現 `failed to alloc from bootmem named ...` 和 `GDP reload failed: staying on hash table N`,新清單沒有生效，**舊清單繼續生效**,系統正常。
2. **RAM 放不下(XML 轉 GDP 時)。** log 出現 `run1.xml: writing /data/ftproot/run1.gdp failed` 之類的 ERROR,新清單被丟掉，**舊清單繼續生效**。6.6.261103.2 以前這種情況不會留下紀錄:寫一半的 GDP 檔被當成成功載入，舊清單留著，三輪更新看起來都成功，完整 reload 之後才發現還是第一份。
3. **開機或完整 reload 時放不下。** 每份清單從 GDP 重新載入，放不下的那份 log 記 `run1.xml.gdp load fail ... not in effect`,**那個 filter 是空的**,其他清單不受影響。有大清單的機器在改完設定後，應該檢查 filter 的筆數。

## 載入要多久

| 動作 | 時間 |
|---|---|
| `ip.addr` 2,100,000 筆即時載入 | 上傳 12 s + 生效 63 s |
| `dns.qry.name` 1,300,000 筆即時載入 | 上傳 9 s + 生效 52 s |
| `ip.addr` CIDR 1,700,000 筆即時載入 | 上傳 10 s + 生效 75 s |
| `ipv6.addr` 92,000 筆即時載入 | 上傳 2 s + 生效 11 s |
| 完整 reload(帶著 2,100,000 筆 ip.addr) | 65 s |
| 重開機 | 約 100 s,清單在開機時從 GDP 載入 |

## 其他注意事項

- **網頁上傳大檔會把 pywww 撐爆。** G8S 上透過 submitxml 送 11 MB 的清單,pywww 吃到 322 MB 記憶體被 OOM 砍掉，而且 MIPS 機器上沒有程式會重啟它。大清單用 scp 放進 `/data/ftproot/run1.xml` 再 `touch run1.xml.ok`,效果和 submitxml 相同。
- **每筆 IPv6 單一位址很貴。** 隨機的 /128 在樹裡約佔 5 KB,同一網段內連號的位址則便宜得多(300 個 /64 各 100 個位址,30,000 筆只要 4.8 MB)。清單如果是整段網段，寫成 CIDR。
