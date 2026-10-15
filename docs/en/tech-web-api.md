# The GRISM web API: the calls there are, and how to use them

Everything Grism Studio can do, it does through the web API on the device; the same API can be scripted, or wired into a system of your own. This note lists every call the device offers today (firmware 6.6.x / 7.6.x) and how to use them.

## Ground rules

- **Address**: `https://<device IP>`, port 443, with a self-signed certificate (`-k` for curl; in a program, turn verification off or import the certificate).
- **Only the listed paths exist.** The nginx in front of the device lets through the paths in this note and answers 404 to everything else.
- **Responses**: most bodies are JSON, yet the `Content-Type` usually says `text/html` -- parse the body as JSON and do not go by the header. A few calls return plain text, XML or a file; the tables say which.
- **Request bodies** come in three kinds, depending on the call:
  - **JSON**: the account calls and the alert calls.
  - **form** (`application/x-www-form-urlencoded`): most of the setting and submitting calls, as in `data=<XML>`.
  - **multipart**: file uploads (firmware, backup, pcap, component); the file field is always named `file`.
- **Success and failure**: 200 on success (the body may be empty, or `{}`); 400 with a one-line reason for a bad parameter; 401 `Unauthorized` when the account's level is too low; and 404 on a path that needs a session when there is **none**.

## Signing in, and the session

```bash
# your own device address, account and password
GRISM=https://DEVICE_IP; ACCOUNT=USERNAME; PASSWORD='PASSWORD'

# sign in: PasswordHash is the SHA-256 of the password, lower-case hex
HASH=$(printf '%s' "$PASSWORD" | sha256sum | cut -d' ' -f1)
curl -k -c cookie.txt -H 'Content-Type: application/json' \
     -d "{\"Username\":\"$ACCOUNT\",\"Password\":\"$PASSWORD\",\"PasswordHash\":\"$HASH\"}" \
     $GRISM/direct_login

# every request after that carries the same cookie
curl -k -b cookie.txt $GRISM/grism/task/get_current_user
# {"username":"USERNAME","priv":15}
```

- A sign-in that works answers `{}` and sets `sessionid` and two more cookies; one that does not answers 400 with the reason (`Username or password is invalid`, for one).
- A local account is checked by `PasswordHash` alone; with TACACS+ or RADIUS on, the device takes `Password` to the server -- so send both.
- The session **expires after 30 minutes without a request**, and every request renews it. A long-running script that meets a 404 signs in again.
- `POST /logout` ends the session.

In Python, with the standard library only:

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
    def get(self, path, **query):                     # the raw body
        return self._send(path + ("?" + urllib.parse.urlencode(query) if query else ""))
    def get_json(self, path, **query):                # strict=False: some strings carry control characters
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

## Account levels

Every account has a level, and the "Level" column of the tables is the lowest one a call accepts.

| Level | Number | May |
|---|---|---|
| view | 1 | read everything, change nothing |
| operator | 5 | + work the packet path: filters and chains, replays and pcap files, clearing counters and the MEC table, bypass, acknowledging alerts |
| config | 10 | + change the device's settings: `submit_config`, time zone, SNMP, services, alert rules, the switch, reboot |
| admin | 15 | + accounts, sign-in servers and the management IP, the management access list, firmware, backup and restore, halt |

"open" means no session is needed.

## The calls

A GET with nothing in its "Parameters" column takes none. "MIPS only" is the 6.6.x models, "arm64 only" the 7.6.x ones.

### Accounts and the session

| Call | Method | Level | Parameters | What it does |
|---|---|---|---|---|
| `/direct_login` | POST | open | JSON `Username` `Password` `PasswordHash` | sign in |
| `/logout` | POST | open | — | sign out |
| `/grism/task/get_current_user` | GET | view | — | `{"username","priv"}`; 404 without a session |
| `/list_user` | GET | admin | — | `{"user_list":[[name, role, level], …]}` |
| `/create_user` | POST | admin | JSON `Username` `PasswordHash` `Priv` (1/5/10/15) | add a local account |
| `/delete_user2` | POST | admin | JSON `Username` | delete an account; not the last administrator |
| `/set_user_priv` | POST | admin | JSON `Username` `Priv` | change a level; the last administrator cannot be demoted |
| `/change_password` | POST | view | JSON `Username` `OldPasswordHash` `NewPasswordHash` | change a password, given the old one |

### The device and its state

| Call | Method | Level | Parameters | What it does |
|---|---|---|---|---|
| `/grism/task/get_version` | GET | open | — | text: the firmware version and its source commit, two lines |
| `/grism/get_machine_id` | GET | open | — | text: the machine id |
| `/grism/get_sn` | GET | open | — | text: the serial number |
| `/grism/task/get_system_status` | GET | view | — | time, `uptime_s`, `loadavg`, `cpu_usage` (overall first, then each core), `mem_usage`, `disk_usage`, temperatures, fans, processes |
| `/grism/task/get_service_versions` | GET | view | — | the version of each component: `grism`, `pywww`, nginx, snmpd, sshd… |
| `/grism/task/get_service_status` | GET | view | — | `service_status`: `state`, `pids` and `ports` of every service |
| `/grism/task/get_status` | GET | view | — | `{"loading","message","gdp_loading"}`: whether a configuration is being loaded |
| `/grism/get_log` | GET | view | — | text: the system log, `log.txt` |
| `/grism/task/get_log_files` | GET | view | — | the system log files (`log.txt` and the `log.txt.1`–`.4` it rotated into): name, size, time |
| `/grism/task/get_log_file` | GET | view | `name` (`log.txt`, `log.txt.1`–`.4`) | plain text: that system log file |
| `/grism/task/download_logs` | GET | view | — | every system log file in one `<model>-log-<date-time>.zip` |
| `/grism/task/get_debug_status` | GET | view | — | whether the packet engine is alive, the core-dump switch, the core files there are |

### Device settings

| Call | Method | Level | Parameters | What it does |
|---|---|---|---|---|
| `/grism/task/get_config` | GET | view | — | the whole configuration as JSON: `args`, `interfaces`, `storages`, `ifcfgs`, logging, heartbeat and the rest |
| `/grism/task/get_config_xml` | GET | view | — | the same as a `<configSet>` XML |
| `/grism/task/submit_config` | POST | config | form `data`=`<configSet>` XML | apply a change; send only what changes. The `<views>` (sign-in servers) and `<ifcfgs>` (management IP) sections need an administrator |
| `/grism/get_time_zone` | GET | view | — | `{"timezone":[{"Asia/Taipei":1}, …]}`; the one with 1 is current |
| `/grism/set_time_zone` | POST | config | form `timezone` | e.g. `Asia/Taipei` |
| `/grism/get_snmp_read_community` | GET | view | — | text |
| `/grism/set_snmp_read_community` | POST | config | form `read_community` (letters and digits) | |
| `/grism/task/set_service_state` | POST | config | form `name` `enable` (1/0) | start or stop a service now: `sshd` `snmpd` `sub-snmpd` `lldpd` `grism_syslogd` `packetx_trap_dispatcher` `telnetd` `pyftpd` `pyhttpd` |
| `/grism/task/get_mgmt_access` | GET | admin | — | the management access list, the ports it guards, the caller's address `client_ip` |
| `/grism/task/set_mgmt_access` | POST | admin | form `allow` | a list of IPv4 addresses or networks; empty means no restriction. 409 when the list leaves the caller's own address out, which would lock the caller out |
| `/grism/task/get_local_login_hosts` | GET | admin | — | the addresses that may still sign in with local accounts while a sign-in server is on |
| `/grism/task/set_local_login_hosts` | POST | admin | form `hosts` | the same kind of list |

### The packet path (filters and chains)

| Call | Method | Level | Parameters | What it does |
|---|---|---|---|---|
| `/grism/task/get_running` | GET | view | — | XML: the `run.xml` in force |
| `/grism/task/get_running_filelist` | GET | view | — | the running configuration files (`run.xml` and any `run<N>.xml`) |
| `/grism/task/get_running_file` | GET | view | `filename` | one of them |
| `/grism/task/del_running` | GET | operator | `name` | delete one of the extra files |
| `/grism/task/submitxml` | POST | operator | form `data`=`<run>` XML, `filename` (`run.xml` or `run<N>.xml`) | **replaces** that file and reloads; it survives a reboot |
| `/grism/task/submit_instant` | POST | operator | form `data`=`<run>` XML | loaded **in addition to** what is there, and not saved; an empty `<run></run>` removes it |
| `/grism/task/get_save_xml_list` | GET | view | — | the saved configurations (the Studio's "Save") |
| `/grism/task/get_save_xml` | POST | view | form `name` | one saved configuration |
| `/grism/task/save_xml` | POST | operator | form `name` `data` `description` | save one |
| `/grism/task/set_xml_description` | POST | operator | form `name` `description` | change its description |
| `/grism/task/del_xml` | POST | operator | form `name` | delete it |

### Traffic statistics and counters

| Call | Method | Level | Parameters | What it does |
|---|---|---|---|---|
| `/grism/task/get_statistics_json` | GET | view ※ | — | `statistics` (per port: link, rates, packet and byte counts, every kind of drop and error), `sessions`, `sessionsv6`, `filter`, `packet_type_counter` |
| `/grism/task/get_lite_statistics_json` | GET | view ※ | — | the per-port part alone: lighter, the one to poll often |
| `/grism/task/get_filter_counter` | GET | view | — | `try_count`, `matched_count` and `matched_per_second` of each filter |
| `/grism/task/get_country_counter` | GET | view | — | traffic by country |
| `/grism/task/get_flow_service` | GET | view | — | traffic grouped by service |
| `/grism/task/get_flow_table` | GET | view | — | the flow table |
| `/grism/task/clear_counters` | GET | operator | — | clear every counter |
| `/grism/task/clear_lite_counters` | GET | operator | — | clear the per-port counters only |

※ On 6.6.261008.4 / 7.6.261008.1 and earlier these two statistics calls answer without a session; later versions want one (a view-level account is enough). A monitoring script that does not sign in gets a 404 after the upgrade: add the sign-in.

### Mapping tables and learned tables

| Call | Method | Level | Parameters | What it does |
|---|---|---|---|---|
| `/grism/task/get_s1ap_items` | GET | view | `offset` `limit` `ue-ipv4` `max-idle` `great-than` `sort` `order` | the MEC mapping table, see below |
| `/grism/task/clear_s1ap_items` | GET | operator | `max-idle` (seconds) | clear the entries idle for longer than that |
| `/grism/task/clear_s1ap_tables` | GET | operator | — | empty the MEC mapping table and its indexes |
| `/grism/task/get_arp_table` | GET | view | — | the ARP table |
| `/grism/task/get_mac_table` | GET | view | — | the learned MAC table |
| `/grism/task/get_port_mac` | GET | view | — | the MAC of every port and of the management ports |
| `/grism/task/get_l2gre_correlation_table` | GET | view | — | the L2 GRE mapping table |
| `/grism/task/get_vxlan_correlation_table` | GET | view | — | the VXLAN mapping table |
| `/grism/task/get_dns_response_ip_table` | GET | view | — | the DNS response address table |
| `/grism/task/get_dns_qry_name_resp_table` | GET | view | — | the DNS query name and response table |
| `/grism/task/get_heartbeat_status` | GET | view | — | `heartbeat_status`: `[[n, up], …]`, n being the position among the *enabled* targets |
| `/grism/task/get_encapsulationEncryptKey` | GET | view | — | text: the encapsulation encryption key |
| `/grism/task/renew_encapsulationEncryptKey` | GET | operator | — | replace the key |

### Storage and capture files

| Call | Method | Level | Parameters | What it does |
|---|---|---|---|---|
| `/grism/task/get_storages` | GET | view | — | the storage volumes and their usage |
| `/grism/task/get_storage_file_list` | POST | view | form `name` (the volume, e.g. `H1`), `dir` | `file_list`: `[kind, name, bytes, modified]`, kind 0 a directory, 1 a file |
| `/grism/task/download_storage_file` | GET | operator | `name` `dir` `filename`, optionally `offset` `limit` | download a file; `offset`/`limit` take a part of it, which is how a capture still being written is followed |
| `/grism/task/del_storage_file` | POST | operator | form `name` `dir` `filename` | delete a file |
| `/grism/task/upload_pcap_file` | POST | operator | query `storage` `dir` `name`; multipart `file` | upload a `.pcap` / `.pcapng`; 413 when there is no room |

### Alerts

| Call | Method | Level | Parameters | What it does |
|---|---|---|---|---|
| `/grism/task/get_alerts` | GET | view | — | `active` (standing) and `events` (lately raised and cleared) |
| `/grism/task/get_alert_rules` | GET | view | — | `rules`, `defaults`, `trap_targets` |
| `/grism/task/set_alert_rules` | POST | config | JSON `rules`, optionally `trap_targets` | a rule: `id` `enable` `threshold` `sustain` `clear`; a trap target: `enable` `host` `port` `community` |
| `/grism/task/ack_alert` | POST | operator | JSON `key` (`*` for all) | acknowledge |
| `/grism/task/test_alert_trap` | POST | config | JSON `trap_targets` | send a test trap |

### Firmware, components, backup and restore

| Call | Method | Level | Parameters | What it does |
|---|---|---|---|---|
| `/grism/task/update` | POST | admin | multipart `file`=`grism-bin.tgz` | upload a firmware image and start the update; settings are carried over |
| `/grism/task/update_check` | GET | config | — | ask the update server whether there is a newer version |
| `/grism/task/update_download` | GET | admin | `version` | have the device fetch that version itself |
| `/grism/task/update_download_check` | GET | view | — | text `downloaded,total` (bytes) |
| `/grism/task/update_download_update` | GET | admin | — | apply the downloaded image |
| `/grism/task/upload_component` | POST | admin | form `target` (`grism-studio` or `pywww`); multipart `file` | replace one component only |
| `/grism/task/backup` | POST | admin | — | make a backup |
| `/grism/task/download_backup` | GET | admin | — | download the backup just made |
| `/grism/task/restore_from_file` | POST | admin | multipart `file`=`grism-backup*.tgz` | restore from a backup |
| `/grism/task/restore` | POST | admin | — | **back to factory settings** |

### Operations

| Call | Method | Level | Parameters | What it does |
|---|---|---|---|---|
| `/grism/task/reboot` | POST | config | — | reboot |
| `/grism/task/halt` | POST | admin | — | halt |
| `/grism/task/clear_log` | POST | config | — | empty the system log |
| `/grism/task/set_debug_core` | GET | config | `enable` (1/0) | core dumps on or off |
| `/grism/task/get_core_file` | GET | config | `name` | download a core file |
| `/grism/task/del_core_file` | GET | config | `name` | delete a core file |
| `/grism/task/restart_grism` | GET | config | — | restart the packet engine alone. arm64 only; MIPS answers 501, reboot instead |

### Bypass and port speed (MIPS only)

`data` is `1` (bypass) / `0` (normal) on the G8S and the T12S, and `7` (bypass) / `9` (normal) on the G8. The status calls answer with the same values.

| Call | Method | Level | Parameters | What it does |
|---|---|---|---|---|
| `/grism/task/get_g8s_hwbypass1_status`<br>`/grism/task/get_g8s_hwbypass2_status` | GET | view | — | G8S: pair 1 (P0/P1), pair 2 (P4/P5) |
| `/grism/task/set_g8s_hwbypass1_mode`<br>`/grism/task/set_g8s_hwbypass2_mode` | POST | operator | form `data` | |
| `/grism/task/get_t12s_hwbypass1_status`<br>`/grism/task/get_t12s_hwbypass2_status` | GET | view | — | T12S: pair 1 (P8/P9), pair 2 (P10/P11) |
| `/grism/task/set_t12s_hwbypass1_mode`<br>`/grism/task/set_t12s_hwbypass2_mode` | POST | operator | form `data` | |
| `/grism/task/get_hwbypass_status` | GET | view | — | G8: P6/P7 |
| `/grism/task/set_hwbypass_mode` | POST | operator | form `data` | |
| `/grism/task/set_t12s_speed` | POST | config | form `qlm5_6` (P0–P3), `qlm3` (P4–P7), `qlm2` (P8–P11), each `1000` or `10000` | T12S port speed, four ports a group; **reboots the device** |
| `/grism/task/set_im_speed` | POST | config | form: each group's name = `1000` or `10000` | port speed on the T20 family; **reboots the device** |

### The front switch (arm64 only, and only the Q16)

**Every write restarts the switch service: about twenty seconds with every port down.**

| Call | Method | Level | Parameters | What it does |
|---|---|---|---|---|
| `/grism/task/get_switch_interface` | GET | view | — | each port: enabled, link, speed, FEC, transceiver details |
| `/grism/task/set_switch_interface` | POST | config | JSON: the ports' settings | edit the normal configuration port by port |
| `/grism/task/get_switch_interface_custom` | GET | view | — | text: the custom configuration |
| `/grism/task/submit_switch_interface_custom` | POST | config | the body is the configuration | write the custom configuration |
| `/grism/task/get_cpss_config_list` | GET | view | — | the configurations there are, and which is in use |
| `/grism/task/get_cpss_config` | GET | view | `path` | text: one configuration |
| `/grism/task/submit_cpss_config` | POST | config | form `path` `data` | write one configuration (no restart) |
| `/grism/task/set_cpss_config_default` | GET | config | `path` | switch to another configuration |
| `/grism/task/get_cpss_service` | GET | view | — | whether the switch service is running |
| `/grism/task/set_cpss_restart` | POST | config | — | restart the switch service by hand |
| `/grism/task/get_cpss_status` | GET | view | — | text: the switch's port status |
| `/grism/task/get_cpss_statistics` | GET | view | — | text: the switch's port counters |

## Things one does often

### Change one setting

`submit_config` takes a `<configSet>` holding only what is to change. The names are those of `args` in what `get_config` returns; `get_config_xml` shows how the whole of it is written.

```python
g.post_form("/grism/task/submit_config",
            data='<configSet reboot="no"><args><hostname>GRISM-LAB</hostname></args></configSet>')
```

Most settings take effect at once. One that changes how memory is laid out -- the size of a table, say -- only takes proper effect after a reboot: set `reboot` to `yes`, or call `reboot` afterwards.

### Load filters and chains, and read what they matched

| | `submitxml` | `submit_instant` |
|---|---|---|
| Effect | replaces `run.xml`: the device's configuration | loaded beside the configuration |
| After a reboot | still there | gone |
| For | the real thing | a test, a capture, something temporary |

```python
run = '''<run>
  <filter id="7001" name="dns" sessionBase="no">
    <or><find name="udp.dstport" relation="==" content="53" /></or>
  </filter>
  <chain><in>P0</in><fid>F7001</fid><out>0</out></chain>
</run>'''
g.post_form("/grism/task/submit_instant", data=run)        # in addition; <out>0</out> counts without forwarding

for f in g.get_json("/grism/task/get_filter_counter")["filter_counter"]:
    print(f["id"], f["try_count"], f["matched_count"], f["matched_per_second"])

g.post_form("/grism/task/submit_instant", data="<run></run>")   # take the addition out again
```

With `submit_instant`, give the filters ids the device is not already using (`get_running` shows what it has). After a submit, poll `get_status` until `loading` is `false` again.

### Query the MEC mapping table

The table can run to hundreds of thousands of rows: page through it.

| Parameter | Meaning |
|---|---|
| `offset`, `limit` | the page; 200 rows or fewer is a good size |
| `ue-ipv4` | one address or network only (`10.45.0.0/16`); `all` lists the rows whose UE address is not known yet as well |
| `max-idle` | only rows idle for no longer than this many seconds; with `great-than=1` the other way round, only those idle for longer |
| `sort`, `order` | the column to order by (`ue-ipv4` `idle` `ul-bytes` `dl-bytes` `ul-packets` `dl-packets`) and `asc` / `desc` |

```python
page = g.get_json("/grism/task/get_s1ap_items", offset=0, limit=200,
                  **{"ue-ipv4": "10.45.0.0/16", "sort": "dl-bytes", "order": "desc"})
print(page["s1ap_items_count"])        # [rows now, capacity]
print(page["s1ap_items_matched"])      # rows that pass the filters, for working out the number of pages
for row in page["s1ap_items"]:
    print(row["ue-ipv4"], row["cellid"], row["ran-ipv4"], row["idle"], row["ul-bytes"], row["dl-bytes"])
```

The answer also carries `s1ap_items_teid_count` and `s1ap_items_sip_count`: `[used, capacity]` of the TEID index and of the UE-address index.

### Download a capture

```python
files = json.loads(g.post_form("/grism/task/get_storage_file_list", name="H1", dir="out"))["file_list"]
data = g.get("/grism/task/download_storage_file", name="H1", dir="out", filename="capture.pcap")
```

A capture still being written has a name ending in `.tmp`; with `offset` (the bytes already read) only what was added since comes back.

### Upload a pcap, update the firmware

```bash
# a pcap: where it goes is in the query string
curl -k -b cookie.txt -F file=@sample.pcap \
     "$GRISM/grism/task/upload_pcap_file?storage=H1&dir=in&name=sample.pcap"

# firmware: the file must be named grism-bin.tgz (or grism-bin-<model>.tgz), and be built for this model
curl -k -b cookie.txt -F file=@grism-bin.tgz $GRISM/grism/task/update
```

Once uploaded, the device applies the image by itself: a MIPS model reboots, an arm64 model restarts the services concerned.

## Things to mind

- **Calls that interrupt traffic**: `reboot`, `halt`, `restore`, `update`, the port-speed calls, and every write to the switch on a Q16. `submitxml` and `submit_instant` reload the configuration.
- **The clearing calls are GETs** (`clear_counters`, `clear_s1ap_tables`, `del_running`…): keep crawlers and link pre-fetchers away from a signed-in session.
- `get_statistics_json` is a large answer; poll `get_lite_statistics_json` when polling often.
- With a large MEC table, every `get_s1ap_items` call walks the whole table, about 0.3 s a call: do not sweep the table in very small pages.
