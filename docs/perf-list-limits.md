# G8S 大量過濾條件的筆數上限

一個 filter 裡放大量的 `ipv6.addr`、`ip.addr`、`dns.qry.name`,能放幾筆?本頁是在一台 G8S 上實測的結果，以及三種清單同時存在時互相影響的方式。最下方的小工具可以直接調整三種的筆數，看會不會超過。

## 測試環境

| 項目 | 設定 |
|---|---|
| 機型 | **G8S** |
| 韌體 | 6.5.260910 |
| IPv4 flow table | **160,000**(`flowCacheBaseSize` 10000,×16) |
| IPv6 flow table | **50,000**(`flowv6TableSize` 50000) |
| 開機後 Linux 可用記憶體 | 約 340 MiB |
| 其他設定 | `run.xml` 為空，沒有流量 |

## 測試方法

- **一種條件一份設定檔。** 每種條件放在自己的 `runN.xml`,裡面只有一個 filter,filter 裡全是同一種 `<find>`。
- **內容。** `ipv6.addr` 用 `2001:db8::/32` 下的隨機 /128;`ip.addr` 用隨機 IPv4;`dns.qry.name` 用類似真實網域的名稱，例如 `cdn74.27dcfee4f4.io`,平均 24 個字元。
- **更新方式。** 把新檔直接覆蓋舊的 `runN.xml`(放進 `ftproot` 再放 `.ok`),**不先清掉前一份**。這是實際換清單時的情況。
- **成功的標準。** 同一個大小連續上傳三份不同內容的清單，每一次都要做到：
  - filter 的筆數換成新清單的筆數;
  - 其他 filter 的筆數不變;
  - 引擎 pid 不變，沒有出現 `Z` 狀態的 `grism-linux_64`;
  - CPU 沒有持續 100%;
  - `/grism/get_log` 沒有錯誤。
- **每換一個大小，先清乾淨。** 刪掉測試用的 `runN.xml` 後重開機，再開始下一個大小。

## 單一種類的結果

| 條件 | 上限(連續 3 次都成功) | 失敗的大小 | XML 大小 |
|---|---|---|---|
| `ipv6.addr` | **12,000** | 12,500、13,000 在第 2 次失敗 | 12,000 筆約 1 MB |
| `ip.addr` | **1,250,000** | 1,375,000 前兩次成功、第 3 次失敗;1,500,000 第 2 次失敗;2,000,000 第 1 次就失敗 | 1,250,000 筆約 76 MB |
| `dns.qry.name` | **1,100,000** | 1,200,000 在第 2 次失敗 | 1,100,000 筆約 84 MB |

- **`ipv6.addr` 吃的是 Linux 記憶體。** IPv6 條件在每個核心的引擎行程裡各存一份逐 bit 的 radix tree,一筆約 18.8 KiB。只載入一次的話 15,000 筆還放得下;16,000 筆會讓一個引擎行程被 OOM 殺掉，變成 `Z`,只能重開機。
- **`ip.addr` 和 `dns.qry.name` 卡在 XML 的大小。** 這兩種會轉成 GDP 即時載入 bootmem 的 hash table,hash table 本身在這個大小還不是限制。限制在 Linux 這邊：新上傳的檔和 running-config 裡的舊檔都放在 `/data` 的 tmpfs,也就是 RAM,watcher 還要再配一塊記憶體來解析新檔。
- **約 84 MB 的 XML 是臨界點。** `ip.addr` 在這個大小時好時壞,`dns.qry.name` 剛好成功。要留餘裕的話，單一清單的 XML 建議在 76 MB 以內。

## 三種同時放

三種清單共用同一塊 Linux 記憶體，所以一種放得多，另外兩種能放的就少。下面每一組都是先清乾淨重開，再依序上傳、輪流更新。

| 組合 | 結果 |
|---|---|
| `ip.addr` 1,000,000 + `dns.qry.name` 800,000 | 輪流各更新 3 次，全部成功 |
| `ip.addr` 1,200,000 + `dns.qry.name` 960,000 | 放 dns 時成功;再更新 ip 時失敗(out of memory),舊的 ip 清單留著 |
| `dns.qry.name` 1,000,000 + `ipv6.addr` 3,000 → 4,000 | 每次更新 IPv6 都成功 |
| `dns.qry.name` 1,000,000 + `ipv6.addr` 5,000 | 再更新一次 IPv6 時，**dns 那個 filter 消失** |
| `dns.qry.name` 1,000,000 + `ipv6.addr` 6,000 | 同上,dns 消失 |
| `ip.addr` 500,000 + `dns.qry.name` 500,000 + `ipv6.addr` 7,500 | 更新 dns 時失敗(out of memory),舊清單留著 |
| `ip.addr` 500,000 + `dns.qry.name` 500,000 + `ipv6.addr` 6,500 | 三種輪流各更新 3 次，全部成功(事先由模型預測) |

## 失敗時會發生什麼

失敗有三種，嚴重程度不同:

1. **更新 ip / dns 清單時放不下。** log 出現 `parse error at line 1: out of memory`,新清單沒有生效，**舊清單繼續生效**。系統正常。
2. **更新 IPv6 清單時放不下。** log 裡沒有錯誤，照樣寫 `reload grism`,但新檔沒有被搬進 running-config,**舊清單繼續生效**。要從 filter 筆數才看得出來。
3. **完整 reload 時重新解析大清單放不下。** 更新 IPv6 清單、修改 `run.xml`,都會讓引擎重新解析每一個 `runN.xml`。這時記憶體不夠，log 會出現 `runN.xml parsing fail`,**那份 ip / dns 清單的 filter 直接消失**,直到下一次放得下的 reload。這是唯一會讓設定實際變少的情況，有大清單的機器在改完設定後，應該檢查 filter 的筆數。

另外，只載入一次的 IPv6 超過約 15,000 筆時，引擎行程會被 OOM 殺掉(第 4 種),要重開機。

## 互相影響的模型

以開機後可用記憶體 F(約 340 MiB)為準，三條規則都要成立:

- **更新 ip / dns 清單:** F − IPv6 表 − 所有清單的 XML − 新檔 ≥ 36 MiB + 1.8 × 新檔。
- **更新 IPv6 清單:** F − IPv6 表 − 所有清單的 XML ≥ 110 MiB。
- **完整 reload:** F − IPv6 表 − 所有清單的 XML ≥ 2.5 × 最大那份 ip / dns XML。

其中 IPv6 表每筆約 18.8 KiB。XML 每行的大小:`ip.addr` 約 61 bytes,`dns.qry.name` 約 52 + 名稱長度,`ipv6.addr` 約 88。這三條規則是用上面 16 個實測成功和失敗的點擬合出來的，最後一組三種同時放的組合，是先用模型預測再實測，結果相符。

模型只適用於上面的測試環境。換了機型、flow table 大小，或者 `run.xml` 本身很大，開機後可用的記憶體就不同，可以在工具的「模型參數」裡調整 F。

## 小工具：三種同時放的上限

拉動或輸入三種清單的筆數，每一列下方會顯示「其他兩項維持目前設定時，這一項最多能放幾筆」,下面的表格列出每個動作還剩多少餘裕。

<iframe src="tools/list-limits.html" data-autoheight style="width:100%;height:900px;border:1px solid #e5e8eb;border-radius:6px" title="三種清單同時放的上限"></iframe>

## 其他注意事項

- **這版韌體的網頁上傳有 10 MB 限制。** 6.5.260910 的 Go 網頁程式，對超過約 10 MB 的 `submitxml` 回 502,所以大清單要用 scp 放進 `/data/ftproot`,再放 `.ok`。
- **開機時也會解析每一份 `runN.xml`。** 本頁每次換大小前都先刪掉清單再重開機，所以沒有測過「帶著接近上限的清單開機」。
