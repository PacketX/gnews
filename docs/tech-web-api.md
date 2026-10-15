# GRISM Web API:可用的呼叫與使用方式

Grism Studio 做得到的事,都是透過設備上的 Web API 做的;同一組 API 也可以直接拿來寫腳本或接到自己的系統。本文列出目前(韌體 6.6.x / 7.6.x)設備實際提供的全部呼叫,以及怎麼用。

## 基本規則

- **位址**:`https://<設備 IP>`,443 埠,憑證是自簽的(curl 加 `-k`,程式裡關掉憑證驗證或匯入憑證)。
- **只有清單內的路徑存在**。設備前端的 nginx 只放行本文列出的路徑,其餘一律回 404。
- **回應格式**:大多數回應的內容是 JSON,但 `Content-Type` 多半標成 `text/html`,請直接把內容當 JSON 解析,不要依賴標頭。少數回純文字、XML 或檔案,表格中會註明。
- **請求本文**有三種,依呼叫而定:
  - **JSON**:帳號相關與告警相關的呼叫。
  - **表單**(`application/x-www-form-urlencoded`):大多數的設定與送出類呼叫,例如 `data=<XML>`。
  - **multipart**:上傳檔案(韌體、備份、pcap、元件),檔案欄位名稱一律是 `file`。
- **成功與失敗**:成功回 200(內容可能是空字串或 `{}`);參數錯誤回 400 並附一句英文原因;權限不足回 401 `Unauthorized`;**尚未登入**時,需要登入的路徑回 404。

## 登入與工作階段

```bash
# 換成自己的設備位址、帳號與密碼
GRISM=https://DEVICE_IP; ACCOUNT=USERNAME; PASSWORD='PASSWORD'

# 登入:PasswordHash 是密碼的 SHA-256(十六進位小寫)
HASH=$(printf '%s' "$PASSWORD" | sha256sum | cut -d' ' -f1)
curl -k -c cookie.txt -H 'Content-Type: application/json' \
     -d "{\"Username\":\"$ACCOUNT\",\"Password\":\"$PASSWORD\",\"PasswordHash\":\"$HASH\"}" \
     $GRISM/direct_login

# 之後每個請求帶同一個 cookie
curl -k -b cookie.txt $GRISM/grism/task/get_current_user
# {"username":"USERNAME","priv":15}
```

- 登入成功回 `{}`,並設定 `sessionid` 等 cookie;失敗回 400 與原因(例如 `Username or password is invalid`)。
- 本機帳號只比對 `PasswordHash`;啟用 TACACS+ 或 RADIUS 時,設備會拿 `Password` 去問伺服器,所以兩個欄位都要送。
- 工作階段 **30 分鐘沒有請求就過期**,每次請求都會續期。長時間執行的腳本遇到 404 時重新登入即可。
- `POST /logout` 結束工作階段。

Python 範例(只用標準函式庫):

```python
import json, ssl, hashlib, urllib.request, urllib.parse, http.cookiejar

class Grism:
    def __init__(self, host, user, password):
        ctx = ssl.create_default_context(); ctx.check_hostname = False; ctx.verify_mode = ssl.CERT_NONE
        self.base = f"https://{host}"
        self.http = urllib.request.build_opener(urllib.request.HTTPSHandler(context=ctx),
                                                urllib.request.HTTPCookieProcessor(http.cookiejar.CookieJar()))
        self.post_json("/direct_login", {"Username": user, "Password": password,
                                         "PasswordHash": hashlib.sha256(password.encode()).hexdigest()})

    def _send(self, path, data=None, headers={}):
        return self.http.open(urllib.request.Request(self.base + path, data=data, headers=headers), timeout=60).read()
    def get(self, path, **query):                     # 回傳原始內容
        return self._send(path + ("?" + urllib.parse.urlencode(query) if query else ""))
    def get_json(self, path, **query):                # strict=False:有些字串裡帶控制字元
        return json.loads(self.get(path, **query).decode("utf-8", "replace"), strict=False)
    def post_form(self, path, **fields):
        return self._send(path, urllib.parse.urlencode(fields).encode())
    def post_json(self, path, obj):
        return self._send(path, json.dumps(obj).encode(), {"Content-Type": "application/json"})

g = Grism("DEVICE_IP", "USERNAME", "PASSWORD")
print(g.get_json("/grism/task/get_config")["args"]["model"])
for port in g.get_json("/grism/task/get_statistics_json")["statistics"]:
    print(port["name"], port["linkStatus"], port["inPps"], port["inMbps"], port["inDrops"])
```

## 帳號等級

每個帳號有一個等級,表格中的「等級」欄是呼叫該端點所需的最低等級。

| 等級 | 數值 | 可以做的事 |
|---|---|---|
| 檢視 view | 1 | 讀取所有資訊,不能更改 |
| 操作 operator | 5 | 加上封包路徑的操作:篩選器與鏈結、重播與封包檔、清除計數器與 MEC 表、旁路、確認告警 |
| 設定 config | 10 | 加上設備設定:`submit_config`、時區、SNMP、服務、告警規則、交換器、重開機 |
| 管理 admin | 15 | 加上帳號、登入伺服器與管理 IP、管理存取清單、韌體、備份還原、關機 |

「公開」表示不需要登入。

## 端點一覽

以下未特別註明的 GET 都沒有參數。「僅 MIPS」指 6.6.x 機種,「僅 arm64」指 7.6.x 機種。

### 帳號與工作階段

| 端點 | 方法 | 等級 | 參數 | 說明 |
|---|---|---|---|---|
| `/direct_login` | POST | 公開 | JSON `Username` `Password` `PasswordHash` | 登入 |
| `/logout` | POST | 公開 | — | 登出 |
| `/grism/task/get_current_user` | GET | 檢視 | — | `{"username","priv"}`;未登入時回 404 |
| `/list_user` | GET | 管理 | — | `{"user_list":[[名稱, 角色, 等級], …]}` |
| `/create_user` | POST | 管理 | JSON `Username` `PasswordHash` `Priv`(1/5/10/15) | 新增本機帳號 |
| `/delete_user2` | POST | 管理 | JSON `Username` | 刪除帳號;最後一個管理員不能刪 |
| `/set_user_priv` | POST | 管理 | JSON `Username` `Priv` | 變更等級;最後一個管理員不能降級 |
| `/change_password` | POST | 檢視 | JSON `Username` `OldPasswordHash` `NewPasswordHash` | 變更密碼,需提供舊密碼 |

### 裝置資訊與狀態

| 端點 | 方法 | 等級 | 參數 | 說明 |
|---|---|---|---|---|
| `/grism/task/get_version` | GET | 公開 | — | 純文字:韌體版本與來源 commit,兩行 |
| `/grism/get_machine_id` | GET | 公開 | — | 純文字:機器 ID |
| `/grism/get_sn` | GET | 公開 | — | 純文字:序號 |
| `/grism/task/get_system_status` | GET | 檢視 | — | 時間、`uptime_s`、`loadavg`、`cpu_usage`(第一個是總體,其後各核心)、`mem_usage`、`disk_usage`、溫度、風扇、行程清單 |
| `/grism/task/get_service_versions` | GET | 檢視 | — | 各元件版本:`grism`、`pywww`、nginx、snmpd、sshd… |
| `/grism/task/get_service_status` | GET | 檢視 | — | `service_status`:每個服務的 `state`、`pids`、`ports` |
| `/grism/task/get_status` | GET | 檢視 | — | `{"loading","message","gdp_loading"}`:設定是否正在載入 |
| `/grism/get_log` | GET | 檢視 | — | 純文字:系統記錄 `log.txt` |
| `/grism/task/get_log_files` | GET | 檢視 | — | 列出系統記錄檔(`log.txt` 及輪替出的 `log.txt.1`–`.4`)的名稱、大小、時間 |
| `/grism/task/get_log_file` | GET | 檢視 | `name`(`log.txt`、`log.txt.1`–`.4`) | 純文字:指定的系統記錄檔 |
| `/grism/task/download_logs` | GET | 檢視 | — | 下載全部系統記錄檔,打包成 `<型號>-log-<日期時間>.zip` |
| `/grism/task/get_debug_status` | GET | 檢視 | — | 封包引擎是否存活、core dump 開關、既有的 core 檔 |

### 設備設定

| 端點 | 方法 | 等級 | 參數 | 說明 |
|---|---|---|---|---|
| `/grism/task/get_config` | GET | 檢視 | — | 整份設定(JSON):`args`、`interfaces`、`storages`、`ifcfgs`、記錄與 heartbeat 等 |
| `/grism/task/get_config_xml` | GET | 檢視 | — | 同一份設定的 `<configSet>` XML 形式 |
| `/grism/task/submit_config` | POST | 設定 | 表單 `data`=`<configSet>` XML | 套用設定變更,只需送要改的部分。`<views>`(登入伺服器)與 `<ifcfgs>`(管理 IP)區段需要管理等級 |
| `/grism/get_time_zone` | GET | 檢視 | — | `{"timezone":[{"Asia/Taipei":1}, …]}`,值為 1 的是目前時區 |
| `/grism/set_time_zone` | POST | 設定 | 表單 `timezone` | 例如 `Asia/Taipei` |
| `/grism/get_snmp_read_community` | GET | 檢視 | — | 純文字 |
| `/grism/set_snmp_read_community` | POST | 設定 | 表單 `read_community`(英數字) | |
| `/grism/task/set_service_state` | POST | 設定 | 表單 `name` `enable`(1/0) | 立即啟動或停止服務:`sshd` `snmpd` `sub-snmpd` `lldpd` `grism_syslogd` `packetx_trap_dispatcher` `telnetd` `pyftpd` `pyhttpd` |
| `/grism/task/get_mgmt_access` | GET | 管理 | — | 管理存取清單、受保護的埠、呼叫端位址 `client_ip` |
| `/grism/task/set_mgmt_access` | POST | 管理 | 表單 `allow` | IPv4 位址或網段清單;空字串代表不限制。清單不含自己的位址時回 409,避免把自己鎖在外面 |
| `/grism/task/get_local_login_hosts` | GET | 管理 | — | 啟用登入伺服器時仍可用本機帳號登入的來源位址 |
| `/grism/task/set_local_login_hosts` | POST | 管理 | 表單 `hosts` | 同上清單 |

### 封包處理設定(篩選器與鏈結)

| 端點 | 方法 | 等級 | 參數 | 說明 |
|---|---|---|---|---|
| `/grism/task/get_running` | GET | 檢視 | — | XML:目前的 `run.xml` |
| `/grism/task/get_running_filelist` | GET | 檢視 | — | 執行中的設定檔清單(`run.xml` 與額外的 `run<N>.xml`) |
| `/grism/task/get_running_file` | GET | 檢視 | `filename` | 取回其中一個檔案 |
| `/grism/task/del_running` | GET | 操作 | `name` | 刪除一個額外的執行設定檔 |
| `/grism/task/submitxml` | POST | 操作 | 表單 `data`=`<run>` XML、`filename`(`run.xml` 或 `run<N>.xml`) | **取代**該檔並重新載入,重開機後仍在 |
| `/grism/task/submit_instant` | POST | 操作 | 表單 `data`=`<run>` XML | 在現有設定之外**額外**載入,不存檔;送空的 `<run></run>` 即清除 |
| `/grism/task/get_save_xml_list` | GET | 檢視 | — | 已儲存的設定(Studio 的「儲存」) |
| `/grism/task/get_save_xml` | POST | 檢視 | 表單 `name` | 取回一份已儲存的設定 |
| `/grism/task/save_xml` | POST | 操作 | 表單 `name` `data` `description` | 儲存一份設定 |
| `/grism/task/set_xml_description` | POST | 操作 | 表單 `name` `description` | 修改說明 |
| `/grism/task/del_xml` | POST | 操作 | 表單 `name` | 刪除 |

### 流量統計與計數器

| 端點 | 方法 | 等級 | 參數 | 說明 |
|---|---|---|---|---|
| `/grism/task/get_statistics_json` | GET | 檢視 ※ | — | `statistics`(每埠:連線狀態、速率、封包與位元組數、各種丟棄與錯誤)、`sessions`、`sessionsv6`、`filter`、`packet_type_counter` |
| `/grism/task/get_lite_statistics_json` | GET | 檢視 ※ | — | 只有每埠統計,較輕量,適合高頻輪詢 |
| `/grism/task/get_filter_counter` | GET | 檢視 | — | 每個篩選器的 `try_count`、`matched_count`、`matched_per_second` |
| `/grism/task/get_country_counter` | GET | 檢視 | — | 依國家的流量計數 |
| `/grism/task/get_flow_service` | GET | 檢視 | — | 依服務分組的流量 |
| `/grism/task/get_flow_table` | GET | 檢視 | — | 流量表內容 |
| `/grism/task/clear_counters` | GET | 操作 | — | 清除所有計數器 |
| `/grism/task/clear_lite_counters` | GET | 操作 | — | 只清除每埠計數器 |

※ 6.6.261008.4 / 7.6.261008.1 以及更早的版本,這兩個統計端點不需要登入就能讀取;之後的版本需要登入(檢視等級即可)。既有的監控腳本若沒有登入,升級後會收到 404,請加上登入步驟。

### 對應表與學習表

| 端點 | 方法 | 等級 | 參數 | 說明 |
|---|---|---|---|---|
| `/grism/task/get_s1ap_items` | GET | 檢視 | `offset` `limit` `ue-ipv4` `max-idle` `great-than` `sort` `order` | MEC 對應表,見下方說明 |
| `/grism/task/clear_s1ap_items` | GET | 操作 | `max-idle`(秒) | 清除閒置超過指定秒數的項目 |
| `/grism/task/clear_s1ap_tables` | GET | 操作 | — | 清空整個 MEC 對應表與索引 |
| `/grism/task/get_arp_table` | GET | 檢視 | — | ARP 表 |
| `/grism/task/get_mac_table` | GET | 檢視 | — | MAC 學習表 |
| `/grism/task/get_port_mac` | GET | 檢視 | — | 各埠與管理埠的 MAC |
| `/grism/task/get_l2gre_correlation_table` | GET | 檢視 | — | L2 GRE 對應表 |
| `/grism/task/get_vxlan_correlation_table` | GET | 檢視 | — | VXLAN 對應表 |
| `/grism/task/get_dns_response_ip_table` | GET | 檢視 | — | DNS 回應 IP 表 |
| `/grism/task/get_dns_qry_name_resp_table` | GET | 檢視 | — | DNS 查詢名稱與回應表 |
| `/grism/task/get_heartbeat_status` | GET | 檢視 | — | `heartbeat_status`:`[[序號, 是否正常], …]`,序號是「已啟用目標」中的位置 |
| `/grism/task/get_encapsulationEncryptKey` | GET | 檢視 | — | 純文字:封裝加密金鑰 |
| `/grism/task/renew_encapsulationEncryptKey` | GET | 操作 | — | 更換金鑰 |

### 儲存空間與封包檔

| 端點 | 方法 | 等級 | 參數 | 說明 |
|---|---|---|---|---|
| `/grism/task/get_storages` | GET | 檢視 | — | 儲存空間清單與使用量 |
| `/grism/task/get_storage_file_list` | POST | 檢視 | 表單 `name`(儲存空間,如 `H1`)、`dir` | `file_list`:`[類型, 名稱, 位元組, 修改時間]`,類型 0 是目錄、1 是檔案 |
| `/grism/task/download_storage_file` | GET | 操作 | `name` `dir` `filename`,可選 `offset` `limit` | 下載檔案;`offset`/`limit` 可只取一段,用來跟讀仍在寫入的擷取檔 |
| `/grism/task/del_storage_file` | POST | 操作 | 表單 `name` `dir` `filename` | 刪除檔案 |
| `/grism/task/upload_pcap_file` | POST | 操作 | 查詢字串 `storage` `dir` `name`;multipart `file` | 上傳 `.pcap` / `.pcapng`;空間不足回 413 |

### 告警

| 端點 | 方法 | 等級 | 參數 | 說明 |
|---|---|---|---|---|
| `/grism/task/get_alerts` | GET | 檢視 | — | `active`(進行中)與 `events`(最近的觸發與解除) |
| `/grism/task/get_alert_rules` | GET | 檢視 | — | `rules`、`defaults`、`trap_targets` |
| `/grism/task/set_alert_rules` | POST | 設定 | JSON `rules`,可選 `trap_targets` | 規則欄位:`id` `enable` `threshold` `sustain` `clear`;trap 目標:`enable` `host` `port` `community` |
| `/grism/task/ack_alert` | POST | 操作 | JSON `key`(`*` 代表全部) | 確認告警 |
| `/grism/task/test_alert_trap` | POST | 設定 | JSON `trap_targets` | 送一個測試 trap |

### 韌體、元件、備份還原

| 端點 | 方法 | 等級 | 參數 | 說明 |
|---|---|---|---|---|
| `/grism/task/update` | POST | 管理 | multipart `file`=`grism-bin.tgz` | 上傳韌體並開始更新;設定會沿用 |
| `/grism/task/update_check` | GET | 設定 | — | 向更新伺服器查詢有無新版 |
| `/grism/task/update_download` | GET | 管理 | `version` | 讓設備自行下載指定版本 |
| `/grism/task/update_download_check` | GET | 檢視 | — | 純文字 `已下載,總大小`(位元組) |
| `/grism/task/update_download_update` | GET | 管理 | — | 套用已下載的韌體 |
| `/grism/task/upload_component` | POST | 管理 | 表單 `target`(`grism-studio` 或 `pywww`);multipart `file` | 只更換單一元件 |
| `/grism/task/backup` | POST | 管理 | — | 產生備份 |
| `/grism/task/download_backup` | GET | 管理 | — | 下載剛產生的備份檔 |
| `/grism/task/restore_from_file` | POST | 管理 | multipart `file`=`grism-backup*.tgz` | 從備份還原 |
| `/grism/task/restore` | POST | 管理 | — | **回復原廠設定** |

### 維運

| 端點 | 方法 | 等級 | 參數 | 說明 |
|---|---|---|---|---|
| `/grism/task/reboot` | POST | 設定 | — | 重開機 |
| `/grism/task/halt` | POST | 管理 | — | 關機 |
| `/grism/task/clear_log` | POST | 設定 | — | 清空系統記錄 |
| `/grism/task/set_debug_core` | GET | 設定 | `enable`(1/0) | 開關 core dump |
| `/grism/task/get_core_file` | GET | 設定 | `name` | 下載 core 檔 |
| `/grism/task/del_core_file` | GET | 設定 | `name` | 刪除 core 檔 |
| `/grism/task/restart_grism` | GET | 設定 | — | 只重啟封包引擎。僅 arm64;MIPS 回 501,請改用重開機 |

### 旁路與埠速(僅 MIPS)

`data` 的值:G8S 與 T12S 是 `1`(旁路)/ `0`(正常);G8 是 `7`(旁路)/ `9`(正常)。狀態查詢回同樣的值。

| 端點 | 方法 | 等級 | 參數 | 說明 |
|---|---|---|---|---|
| `/grism/task/get_g8s_hwbypass1_status`<br>`/grism/task/get_g8s_hwbypass2_status` | GET | 檢視 | — | G8S:第 1 對(P0/P1)、第 2 對(P4/P5) |
| `/grism/task/set_g8s_hwbypass1_mode`<br>`/grism/task/set_g8s_hwbypass2_mode` | POST | 操作 | 表單 `data` | |
| `/grism/task/get_t12s_hwbypass1_status`<br>`/grism/task/get_t12s_hwbypass2_status` | GET | 檢視 | — | T12S:第 1 對(P8/P9)、第 2 對(P10/P11) |
| `/grism/task/set_t12s_hwbypass1_mode`<br>`/grism/task/set_t12s_hwbypass2_mode` | POST | 操作 | 表單 `data` | |
| `/grism/task/get_hwbypass_status` | GET | 檢視 | — | G8:P6/P7 |
| `/grism/task/set_hwbypass_mode` | POST | 操作 | 表單 `data` | |
| `/grism/task/set_t12s_speed` | POST | 設定 | 表單 `qlm5_6`(P0–P3)、`qlm3`(P4–P7)、`qlm2`(P8–P11),值 `1000` 或 `10000` | T12S 埠速,四埠一組;**會重開機** |
| `/grism/task/set_im_speed` | POST | 設定 | 表單:各組名稱 = `1000` 或 `10000` | T20 系列的埠速;**會重開機** |

### 前置交換器(僅 arm64,且僅 Q16)

**每次寫入都會重啟交換器服務,所有埠約中斷二十秒。**

| 端點 | 方法 | 等級 | 參數 | 說明 |
|---|---|---|---|---|
| `/grism/task/get_switch_interface` | GET | 檢視 | — | 各埠:啟用、連線、速率、FEC、光模組資訊 |
| `/grism/task/set_switch_interface` | POST | 設定 | JSON:各埠設定 | 逐埠修改一般設定檔 |
| `/grism/task/get_switch_interface_custom` | GET | 檢視 | — | 純文字:自訂設定檔 |
| `/grism/task/submit_switch_interface_custom` | POST | 設定 | 請求本文即設定檔內容 | 寫入自訂設定檔 |
| `/grism/task/get_cpss_config_list` | GET | 檢視 | — | 可用的設定檔與目前使用的是哪一個 |
| `/grism/task/get_cpss_config` | GET | 檢視 | `path` | 純文字:讀取一個設定檔 |
| `/grism/task/submit_cpss_config` | POST | 設定 | 表單 `path` `data` | 寫入一個設定檔(不重啟) |
| `/grism/task/set_cpss_config_default` | GET | 設定 | `path` | 切換使用的設定檔 |
| `/grism/task/get_cpss_service` | GET | 檢視 | — | 交換器服務是否在執行 |
| `/grism/task/set_cpss_restart` | POST | 設定 | — | 手動重啟交換器服務 |
| `/grism/task/get_cpss_status` | GET | 檢視 | — | 文字:交換器各埠狀態 |
| `/grism/task/get_cpss_statistics` | GET | 檢視 | — | 文字:交換器各埠計數 |

## 常用流程

### 修改一項設定

`submit_config` 收的是 `<configSet>`,只需要寫出要改的項目。項目名稱與 `get_config` 回傳的 `args` 相同;要看完整的寫法可以先呼叫 `get_config_xml`。

```python
g.post_form("/grism/task/submit_config",
            data='<configSet reboot="no"><args><hostname>GRISM-LAB</hostname></args></configSet>')
```

多數項目立即生效;會改變記憶體配置的項目(例如各種表的大小)要重開機才會正確生效,請把 `reboot` 設成 `yes` 或隨後呼叫 `reboot`。

### 套用篩選器與鏈結,並讀取命中數

| | `submitxml` | `submit_instant` |
|---|---|---|
| 效果 | 取代 `run.xml`,成為設備的設定 | 在現有設定之外額外載入 |
| 重開機後 | 仍在 | 消失 |
| 適合 | 正式套用 | 暫時性的測試、擷取 |

```python
run = '''<run>
  <filter id="7001" name="dns" sessionBase="no">
    <or><find name="udp.dstport" relation="==" content="53" /></or>
  </filter>
  <chain><in>P0</in><fid>F7001</fid><out>0</out></chain>
</run>'''
g.post_form("/grism/task/submit_instant", data=run)        # 額外載入;<out>0</out> 只計數不轉送

for f in g.get_json("/grism/task/get_filter_counter")["filter_counter"]:
    print(f["id"], f["try_count"], f["matched_count"], f["matched_per_second"])

g.post_form("/grism/task/submit_instant", data="<run></run>")   # 清除剛才額外載入的部分
```

用 `submit_instant` 時,篩選器的 `id` 不要和設備現有的重複(先用 `get_running` 看現有設定)。送出後可輪詢 `get_status`,等 `loading` 變回 `false`。

### 查詢 MEC 對應表

表可能有數十萬列,務必分頁:

| 參數 | 意義 |
|---|---|
| `offset`、`limit` | 分頁;一頁建議 200 列以內 |
| `ue-ipv4` | 只看某個位址或網段(如 `10.45.0.0/16`);`all` 連尚未學到 UE 位址的列也列出 |
| `max-idle` | 只看閒置不超過這個秒數的列;加上 `great-than=1` 則反過來,只看閒置超過的 |
| `sort`、`order` | 排序欄位(`ue-ipv4` `idle` `ul-bytes` `dl-bytes` `ul-packets` `dl-packets`)與 `asc` / `desc` |

```python
page = g.get_json("/grism/task/get_s1ap_items", offset=0, limit=200,
                  **{"ue-ipv4": "10.45.0.0/16", "sort": "dl-bytes", "order": "desc"})
print(page["s1ap_items_count"])        # [目前列數, 容量]
print(page["s1ap_items_matched"])      # 符合條件的總列數,用來算頁數
for row in page["s1ap_items"]:
    print(row["ue-ipv4"], row["cellid"], row["ran-ipv4"], row["idle"], row["ul-bytes"], row["dl-bytes"])
```

回應另有 `s1ap_items_teid_count` 與 `s1ap_items_sip_count`,是 TEID 索引與 UE 位址索引的 `[已用, 容量]`。

### 下載擷取檔

```python
files = json.loads(g.post_form("/grism/task/get_storage_file_list", name="H1", dir="out"))["file_list"]
data = g.get("/grism/task/download_storage_file", name="H1", dir="out", filename="capture.pcap")
```

擷取進行中的檔案名稱結尾是 `.tmp`;帶上 `offset`(已讀到的位元組數)可以只取新增的部分。

### 上傳 pcap、更新韌體

```bash
# pcap:目的地寫在查詢字串
curl -k -b cookie.txt -F file=@sample.pcap \
     "$GRISM/grism/task/upload_pcap_file?storage=H1&dir=in&name=sample.pcap"

# 韌體:檔名須為 grism-bin.tgz(或 grism-bin-<機型>.tgz),機型必須與設備相符
curl -k -b cookie.txt -F file=@grism-bin.tgz $GRISM/grism/task/update
```

韌體上傳完成後設備會自行套用:MIPS 機種會重開機,arm64 機種只重啟相關服務。

## 注意事項

- **會中斷流量的呼叫**:`reboot`、`halt`、`restore`、`update`、埠速設定,以及 Q16 上所有交換器的寫入。`submitxml` 與 `submit_instant` 會重新載入設定。
- **清除類的呼叫用的是 GET**(`clear_counters`、`clear_s1ap_tables`、`del_running`…),不要讓爬蟲或預先載入機制碰到已登入的工作階段。
- `get_statistics_json` 內容較大,高頻輪詢請用 `get_lite_statistics_json`。
- MEC 表很大時,`get_s1ap_items` 每次呼叫都要走過整張表,單次約 0.3 秒,不要用很小的頁逐頁掃完整張表。
