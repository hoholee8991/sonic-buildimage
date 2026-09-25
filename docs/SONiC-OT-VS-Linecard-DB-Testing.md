# SONiC-OT VS 線卡插拔與 DB 查閱實測記錄

> 2026-09-23 實測。環境:WSL2 內 qemu 運行 `sonic-ot-vs.img`(Platform `x86_64-ot_kvm_x86_64_4_asic-r0`,4 ASIC),所有容器 Up。
> 配套文件:`SONiC-OT-VS-Build-Troubleshooting.md`(建置與 VM 運行時問題 §1–8)。

---

## 1. 前置條件

### 1.1 VM 已運行

```bash
ss -tlnp | grep -E ':9000|:3041'   # 兩 port 監聽即 VM 活著
ssh -p 3041 admin@localhost        # 密碼 YourPaSsWoRd
```

### 1.2 加板腳本未隨映像安裝 — 需手動帶入(發現 #1)

`config_sonic_otn_linecard.sh` 只存在於 build 樹根目錄(git `fcab41927`),**無任何安裝機制**,映像內沒有。從 WSL 帶入:

```bash
# WSL 側(~/sonic-buildimage/config_sonic_otn_linecard.sh)
sshpass -p YourPaSsWoRd scp -P 3041 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
  ~/sonic-buildimage/config_sonic_otn_linecard.sh admin@localhost:/tmp/
ssh -p 3041 admin@localhost 'sudo mv /tmp/config_sonic_otn_linecard.sh /usr/local/bin/ && sudo chmod +x /usr/local/bin/config_sonic_otn_linecard.sh'
```

### 1.3 板卡型號

| 型號 | 目錄(映像內 `/usr/share/sonic/device/$PLATFORM/linecards/`) | 內容 |
|------|------|------|
| E110C | `E110C/` | config_db.json.j2 + cli_capability.json + flexcounter.json |
| P230C | `P230C/` | 同上 + L1_400G_CA_100GE 目錄 |

插槽 N → ASIC N-1(slot 1 → asic0,slot 4 → asic3)。

---

## 2. 腳本用法與語法陷阱(發現 #2:大小寫 bug)

```bash
sudo config_sonic_otn_linecard.sh <slot> <TYPE>    # 加板
sudo config_sonic_otn_linecard.sh <slot> none      # 刪板
```

**⚠️ TYPE 必須用大寫 `E110C` / `P230C`。**

腳本內部(可從下列兩行對照看出):

```bash
LINECARD_TYPE_UPPERCASE=${LINECARD_TYPE^^}         # line 9:算了大寫變數
sudo sh -c "SLOT_ID=.. ASIC_ID=.. j2 /etc/sonic/linecards/$LINECARD_TYPE/config_db.json.j2 > ..."  # line 31:卻用「原始輸入」拼路徑
```

- 映像內目錄名是大寫 `E110C`;若照腳本註解範例輸入小寫 `e110c`,j2 渲染不存在的路徑 → **產出 0 byte 的 `config_dbN.json`** → `sonic-cfggen` 對空檔 traceback(靜默,腳本不會停)。
- 實測小寫輸入:STATE_DB hset 與 linkup 照樣執行(表面成功),但 **CONFIG_DB 完全沒有線卡表**。

### 2.1 靜默失敗的檢測法(發現 #3)

加板後 30 秒內檢查,兩者皆須成立:

```bash
sudo ls -la /etc/sonic/config_db0.json                  # 0 bytes = 失敗;~9777 bytes = 成功
sonic-db-cli -n asic0 CONFIG_DB keys 'LINECARD*'        # 應回 LINECARD|LINECARD-1-1
```

---

## 3. 加板後的 DB 狀態(E110C 成功基準,slot 1)

### 3.1 asic0 CONFIG_DB — 11 張模板表

| 表 | 筆數 | 備註 |
|----|------|------|
| LINECARD | 1 | `LINECARD-1-1` → linecard-type=E110C |
| AMPLIFIER | 2 | CONSTANT_GAIN,target-gain 15.0 |
| APS / APS_PORT | 1 / 6 | OLP 保護組 |
| PORT | 28 | EDFA×4、OLP×6、MD×16(33/38/43/48/53/58)、LINE×4 |
| ATTENUATOR | 3 | attenuation 15.0 |
| MUX | 1 | oper-status ACTIVE |
| OSC / INTERFACE / LLDP | 各 1 | OSC 通道 |
| DEVICE_METADATA | 1 | hwsku=E110C(覆寫) |

加上原有 FEATURE×13 + LOGGER×24。

### 3.2 asic0 STATE_DB — `LINECARD|LINECARD-1-1`

寫入後由 otss 模擬長出 **23 欄位運行狀態**:`oper-status=ACTIVE`、`slot-status=Ready`、`power-admin-state=POWER_ENABLED`、`led-color=RED`、`baud-rate=9600`、`cpld-version/fpga-version/ucd-version=1.0`、溫度告警門檻等。

### 3.3 APPL_DB / ASIC_DB — OTAI 全鏈路活起來

- APPL_DB:52 keys
- ASIC_DB:47 keys,含 `ASIC_STATE:OTAI_OBJECT_TYPE_PORT`、`_OSC`、`_ATTENUATOR`、`_APSPORT` 等 OTAI 物件(syncd-ot + otss 建立)

### 3.4 其他驗證點

```bash
sudo docker exec syncd-ot0 ls /tmp/linkup    # 腳本 touch 的 linkup 標記
sudo docker ps | grep -E 'otss0|syncd-ot0'   # 容器 Up
```

---

## 4. 刪板行為(發現 #4:`flushall` 是 redis 實例級)

`config_sonic_otn_linecard.sh 1 none` 執行:

```bash
sudo systemctl stop otss@0 syncd-ot@0
sonic-db-cli -n asic0 STATE_DB flushall        # ← 實例級!整個 asic0 的 db0~9 全清
sonic-db-cli -n asic0 CONFIG_DB SET CONFIG_DB_INITIALIZED 1
sudo rm /etc/sonic/config_db0.json
sudo systemctl start syncd-ot@0 otss@0
```

實測結果(全數確認):

| 檢查項 | 結果 |
|--------|------|
| STATE_DB `LINECARD*` | ✅ 清空 |
| asic0 CONFIG_DB | ⚠️ **只剩 `CONFIG_DB_INITIALIZED`** — FEATURE×13、LOGGER×24 也被 flushall 清掉 |
| APPL_DB / ASIC_DB | ✅ OTAI 物件歸零(52→1、47→1) |
| `/etc/sonic/config_db0.json` | ✅ 已刪 |
| global CONFIG_DB(db4) | ✅ 無 LINECARD 表(線卡只在 per-asic DB) |

> 註:service 重啟後 LOGGER×24 會由 otss/syncd 啟動流程補回,**FEATURE×13 不會**(需乾淨開機才恢復);實測服務與容器仍正常運作。

---

## 5. 快速插拔循環會觸發 systemd StartLimit(發現 #5)

「刪板 → 立刻加板」的循環測試中:

```
Job for syncd-ot@0.service failed because start of the service was attempted too often.
```

- 連續 stop/start 踩中 `StartLimitBurst` → 服務被封鎖 → **otss0 / syncd-ot0 容器直接消失**(`docker ps` 看不到)
- 腳本對此無處理:後續 hset/linkup 照跑,linkup 因容器不在而 `Container ... is not running`
- 失敗的重啟嘗試期間,STATE_DB/CONFIG_DB 曾短暫出現 LINECARD key 殘留(暫態,乾淨重啟後消失)

**恢復程序(必做 workaround):**

```bash
sudo systemctl reset-failed
sudo systemctl start syncd-ot@0.service otss@0.service
# 等容器 Up(實測 <5s),再重跑加板腳本
```

恢復後驗證:`show platform summary` 正常、DB 回乾淨 baseline、全容器 Up。

---

## 6. DB 查閱指令清單

```bash
# sonic-db-cli(推薦,自動處理 namespace/port/index)
sonic-db-cli -n asic0 CONFIG_DB keys 'LINECARD*'         # 列 key
sonic-db-cli -n asic0 CONFIG_DB hgetall 'LINECARD|LINECARD-1-1'
sonic-db-cli -n asic0 STATE_DB hgetall 'LINECARD|LINECARD-1-1'
sonic-db-cli -n asic0 APPL_DB keys '*'                   # 應用層(OTAI 表)
sonic-db-cli -n asic0 ASIC_DB keys 'ASIC_STATE*'         # 硬體物件(OTAI oid)
sonic-db-cli CONFIG_DB keys '*'                          # 全域(host,不加 -n)

# redis-cli 直查(進階)
sudo docker exec database0 redis-cli -n 4 keys 'LINECARD*'   # asic0 的 CONFIG_DB=db4
sudo docker exec database0 redis-cli -n 6 keys 'LINECARD*'   # asic0 的 STATE_DB=db6
sudo docker exec database redis-cli -n 4 keys '*'            # 全域 database 容器
redis-cli -h 240.127.1.3 -p 6379 ping                        # asic0 redis 直連(asicN → 240.127.1.(3+N))

# 表筆數統計
sonic-db-cli -n asic0 CONFIG_DB keys '*' | cut -d'|' -f1 | sort | uniq -c
```

DB index 對照:APPL_DB=0、ASIC_DB=1、CONFIG_DB=4、STATE_DB=6。

---

## 7. 安全插拔 SOP(綜合 workaround)

```bash
# 加板(slot 1,E110C)
sudo config_sonic_otn_linecard.sh 1 E110C
sleep 30 && sudo ls -la /etc/sonic/config_db0.json           # 確認非 0 byte
sonic-db-cli -n asic0 CONFIG_DB hgetall 'LINECARD|LINECARD-1-1'
sonic-db-cli -n asic0 ASIC_DB keys 'ASIC_STATE*' | wc -l     # 應 >40

# 刪板
sudo config_sonic_otn_linecard.sh 1 none
sudo systemctl reset-failed                                   # ← 防下次啟動被 StartLimit 封鎖
sudo systemctl start syncd-ot@0.service otss@0.service        # ← 腳本啟動失敗時的保險
sudo docker ps | grep -E 'otss0|syncd-ot0'

# 若容器消失(循環插拔後)
sudo systemctl reset-failed && sudo systemctl start syncd-ot@0.service otss@0.service
```

---

## 8. 測試結論

| 項目 | 結果 |
|------|------|
| 加板(大寫 `E110C`) | ✅ 全鏈路成功:11 張 CONFIG_DB 表 + STATE_DB 23 欄位 + ASIC_DB 47 OTAI 物件 + linkup |
| 加板(小寫 `e110c`,照腳本註解用法) | ❌ 靜默失敗(大小寫 bug,j2 路徑不存在) |
| 刪板 | ✅ 清得乾淨(比預期更狠:實例級 flushall,連 FEATURE/LOGGER 都清) |
| 快速插拔循環 | ❌ 觸發 systemd StartLimitBurst → 容器消失;`reset-failed` + 手動 start 可恢復 |
| 平台整體健康 | ✅ 測試全程 global 服務(database/bgp/mgmt 等)不受影響,`show platform summary` 恆常 |

**腳本缺陷清單**(若日後要 upstream):未安裝進映像、line 31 大小寫 bug、失敗不中止、刪板 flushall 過廣、無 StartLimit 處理。
