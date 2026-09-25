# SONiC OT-VS 建置修改分析(對齊包・完整版)

- 目的:釐清 `tom` 分支相對 `origin/otn_pre_202411` 的全部修改——為何存在、歸屬哪類、如何與 Joe 對齊
- 分析方式:實證調查(git diff 全量比對、實網 URL 測試、slave 容器內 apt 解析實驗、雙方 log 比對)
- 版本:v2(2026-09-24,併入 Joe 對 10 題問題清單的回答)
- 配套文件:`SONiC-OT-VS-Build-Contract.md`(契約)、`SONiC-OT-VS-Joe-Alignment-Brief.md`(會議版)、`SONiC-OT-VS-Build-Troubleshooting.md`(排解全文)

## 1. 前提修正:三個關鍵事實

| # | 事實 | 證據 |
|---|------|------|
| 1 | 實際建置樹是 WSL `~/sonic-buildimage`(ext4);`D:\goto\SONiC_vm\sonic-buildimage` 未參與成功建置 | 文檔 §0 鐵律 2;D: 樹僅含早期失敗建置殘留與過期版 mirror script |
| 2 | Joe 的 WIP commit(`189b747f7`,2026-09-01)不是完整的最小修改集,且其本機修改未全部進 commit | `build_kvm_image.sh` 留有 CI `exit 0` stub(他親證同一行);`src/sonic-linux-kernel/Makefile` 的 URL 修正以**未 commit patch** 形式存在(2026-09-24 Joe 證實)——message 宣告過、實作在樹外 |
| 3 | sonicstorage 403 是全域性事件,雙方當時與現在都撞到 | 雙方實測均 403(Azure Network Security Perimeter);Joe WIP message 自證 |

## 2. 兩棵樹的實證關係

| 項目 | D: 樹 | WSL 樹(權威) |
|------|-------|---------------|
| 實質檔案修改 | 僅 `scripts/build_mirror_config.sh` 過期版(mtime 09-21 17:26,無 backports append) | 同檔 09-21 15:50,內容為 D: 版本超集 |
| 44 個 submodule M | Windows git 讀不了 WSL symlink 的幻影 | git 視角乾淨;SHA 與 superproject 記錄一致 |
| 結論 | 已被 WSL 完整覆蓋,可封存 | 全部有效修改所在,commit 已在此樹完成 |

時間線:WSL 15:50 改 mirror script → D: 17:26 改(較晚、內容為子集)→ 建置完成 09-22。D: 的修改從未流入建置,也不需要。

## 3. 修改分類與決策對照表(v2)

分類定義:

- **④ 分支缺陷/跨機必要**:任何人在此 commit 編「真映像」都需要,與環境、時間無關
- **② 時間因素**:雙方共有的外部狀態在 09-01(Joe)與 09-21(本方)之間改變
- **③ 環境因素**:本機特有的條件
- **① 多餘/可選**:相對特定建置範圍不需要

> v2 修正:v1 把測試跳過歸「③ WSL2 特有」——**錯誤**,Joe 是原生 Fedora 也撞到同樣失敗(見 §7)。③ 只剩主機能力差異與 Windows 傳輸偽修改。

### 3.1 類別 ④:跨機實證必要

| 修改 | 排解編號 | 證據 | 處置 |
|------|---------|------|------|
| `build_kvm_image.sh` 刪 `exit 0` + 條件 KVM 加速 | E18/E21 | CI commit `67a8f6676`(2024-01-31)插入 stub;Joe 親證同一行;不刪 = 435-byte stub | 已 commit `tom` |
| `build_image.sh` 主映像改用 4-asic ONIE recovery ISO | E19 | 白名單不含單 ASIC 機器;Joe 手動安裝踩到等價問題、採同解法(下載 4asic recovery ISO + FAT32 payload) | 已 commit `tom` |
| `config-setup` + `syncd_common.sh` runtime 修正 | F23–F27 | 多重 redis 使 `PING` 解析壞;per-asic 旗標設不上;不修開不了機(§8 排解 + 重開機持久驗證) | 已 commit `tom`;⚠️ 待與 Joe 的 session log 交叉比對(他的 VM 只見 17 容器 vs 我方 19,他可能在 VM 內做過等價修法) |
| `rules/sonic-config.mk` `_TEST = n` | D14 | 跨機證實:我方 log(make-build7.log:1635)與 Joe 的 `DEB_BUILD_OPTIONS=nocheck` 各自撞到同類 dh_auto_test 失敗 | 已 commit `tom`(python wheel knob) |
| `$*_DEB_BUILD_OPTIONS = nocheck`(systemd-sonic-generator、swss) | D15/D16 | 跨機證實(同上);採上游既有 knob(先例 rules/otairedis.mk、sairedis.mk、syncd.mk),取代 debian/rules hack | 手術後 commit `tom` |
| `sonic-bmp` debian/rules exec bit 644→755 | — | 機制:`dpkg-buildpackage -rfakeroot` 直呼 `debian/rules`(slave.mk:808),git index 為 644,新 clone 必炸;**Joe 機器雙機實證(2026-09-25)**:index 同為 644 但磁碟 755(`ls-files -s` vs `ls -l`,`core.filemode=true`),其建置成功全賴磁碟意外 exec bit,submodule 處於 mode-only dirty | 已 commit;⚠️ 待 Joe 確認當初是否手動 `chmod +x`(同源確認) |

### 3.2 類別 ②:時間因素

| 修改 | 排解編號 | 證據 | 處置 |
|------|---------|------|------|
| `build_mirror_config.sh` snapshot 釘 `20260601` + backports `20241001` append | C9/C10/C11 | 今日實測:deb.debian.org security **pool 檔 404**(dists 索引 200——Joe 測的是索引)、backports 404;security pool 於 9/2 後被清空,Joe 建置時尚在 | 已 commit `tom` |
| `sonic-slave-bullseye/Dockerfile.j2` + `backports-pin` | C12 | **E5 實證(2026-09-24,slave 容器內)**:無 pin 時 `apt-get build-dep linux` EXIT=100,`libtracefs-dev (>= 1.3)` 無法解析——即使 backports 1.5.0 就在 priority 100 版本表;apt 不回退低優先級。kernel 本體在 bookworm slave 建(bookworm main 1.6.4 免 pin),但 **bullseye slave 映像建置**也有 `build-dep linux`(Dockerfile.j2:522) | 已 commit `tom`;pin 必要(疑慮關閉)。Joe 為何無 pin 可過,待其 `docker-base-bullseye.gz.log` 解釋 |
| `sonic-linux-kernel/Makefile` URL → deb.debian.org | D17 | sonicstorage 403 全域(雙方);deb.debian.org 200 驗證;Joe 的替代解(snapshot 20240621)已記錄為備援,去重留我方 | 已 commit(`tom` submodule) |
| 基底映像 `debian:bullseye` 漂移 | C11 | Joe 當時即以 digest 建置(`da5c2dc5...`,其 log Step 1/41);我方 9/21 re-pull 到較新 base | 已釘 digest(手術後 commit `tom`,pull 已驗證) |

### 3.3 類別 ③:環境因素(v2 範圍縮小)

| 修改 | 排解編號 | 證據 | 處置 |
|------|---------|------|------|
| `install_sonic.py` timeout 1200→3600s + `KVM_ACCEL` | E21 | 主機能力差異(KVM 有無/速度);條件化無副作用 | 已 commit `tom`(保險絲,建議保留) |
| Windows clone → rsync 傳輸偽修改 | — | symlink/autocrlf 汙染(bmp mode 為唯一有功能意義者,已併入 §3.1) | 已還原/本地 exclude,未進 commit |

### 3.4 類別 ①:多餘/可選

| 項目 | 說明 | 處置 |
|------|------|------|
| ~~`device/virtual-ot/x86_64-ot_kvm_x86_64_6_asic-r0/`~~ | **已定案移除(2026-09-24,選項 a)**:實查該目錄為 4asic 完整複製品(`asic.conf` 仍是 `NUM_ASIC=4`,`diff -rq` 零差異)——掛 6asic 名實為 4 卡機,名實不符且無 6 卡需求 | 已從 `tom` 移除 |
| 建置殘留(fsroot、installer/platforms、core dump 等 34 項) | 建置副產物,可再生 | 本地 `.git/info/exclude` 靜音,永不 commit |
| D: 樹整體 | 過期鏡像 | 封存;之後從 WSL 回存刷新 |

## 4. 三假說結論(v2)

| 假說 | 結論 |
|------|------|
| 1. 修改是多餘的 | 開放項全數關閉(2026-09-24):bmp 保留(機制必要)、6asic 移除(名實不符,目錄實為 4asic 複製品);C12 pin 經 E5 實證必要 |
| 2. 共有外部因素隨時間改變 | 成立:security pool 於 9/2 後清空、backports 退役、基底映像漂移——皆已釘死 |
| 3. 環境不同引致 | 大幅縮小:D15/D16 非 WSL2 特有(跨機證實);剩 KVM 主機能力與 Windows 傳輸偽修改 |
| (列表外)4. 分支本身不完整 | 成立(雙方證據):Joe 樹留 stub、其 kernel patch 未 commit、runtime bug 雙方各自處理 |

## 5. 已執行動作記錄

### 5.1 分支與 commit(branch `tom`;2026-09-24 手術後 SHA 以 fork 現值為準)

| commit(主題) | 內容 | 對應編號 |
|--------------|------|---------|
| build: pin bullseye apt sources to snapshot.debian.org | mirror snapshot 釘選 + backports append + pin(C12) | C9–C12 |
| build: make KVM image generation work for local builds | E18/E19/E21 | E 系列 |
| build: skip broken unit tests via per-package knobs | D14/D15/D16 統一 knob(手術後形式) | D 系列 |
| runtime: fix per-asic config init and DB wait | config-setup + syncd_common | F 系列 |
| submodules: bump sonic-linux-kernel, sonic-bmp | D17 + exec bit(swss 修正已由 knob 取代,submodule 回歸上游基點) | D17/bmp |
| build: pin bullseye slave base image to digest | `debian:bullseye@sha256:da5c2dc5...`(C11 anchor) | C11 |
| docs: add OT-VS build fixes analysis, contract, ... | 對齊包文檔 | — |

submodule 分支 `tom`:`sonic-linux-kernel` `8ce2a4987`、`sonic-bmp` `2e07a74a2`;`sonic-swss` 已回歸上游基點 `4eb74f008`(修正移至 superproject knob)。

### 5.2 fork 與 bundle

| 對象 | 狀態 |
|------|------|
| `hoholee8991/{sonic-buildimage,sonic-linux-kernel,sonic-bmp}` | 分支 `tom` 已推 |
| `hoholee8991/sonic-swss` | fork 已建;`tom` 分支已刪(無差異 commit) |
| `ot-vs-tom-*.bundle`(D:\goto\SONiC_vm\) | 離線備援;super 已隨手術重製,swss bundle 已移除 |

### 5.3 排障記錄:`git add --all` 失敗事件

| 原因 | 處理 |
|------|------|
| `fsroot-ot-vs/`(root 所有的建置殘留)使 add 中途 fatal | 改精準 add;殘留寫入本地 exclude |
| submodule 內未 commit 的修改,superproject 的 add 看不見 | 先在 submodule 內各自 commit,再 bump gitlink |
| `.git/objects` 有 3180 個 root 擁有的物件 | `sudo chown -R user:user .git` |

## 6. 未決事項(v2)

| # | 事項 | 狀態 |
|---|------|------|
| 1 | commit 作者 email | ✅ 已修正(2026-09-24):`tomleungwork1998@gmail.com`,superproject + 2 submodule reset-author 並 force-push |
| 2 | bmp exec bit 定案 | ✅ 已定案(2026-09-24):保留——機制上必要,不再等 Joe |
| 3 | 6asic 範圍 | ✅ 已定案(2026-09-24):移除(選項 a),範圍 = 主 + 4asic;6asic 目錄實為 4asic 複製品(`NUM_ASIC=4`) |
| 4 | F23–F27 交叉比對、17 vs 19 容器 | 待 Joe session-ses_f7fd*.md |
| 5 | Joe 無 pin 過關之謎 | 待 docker-base-bullseye.gz.log(build-dep linux 段) |
| 6 | `DEFAULT_CONTAINER_REGISTRY` 雙方值對齊 | Joe 用 `""`(容器全部本地建);我方值待確認 |
| 7 | `Troubleshooting.md` §3-C12 補 E5 實證註記 | 待使用者批准修訂 |
| 8 | builder 容器(Dockerfile + build.sh + run.sh) | 下一工程項目,方向已共識 |

## 7. Joe 回答摘要(2026-09-24)與新追問

### 7.1 十題要點

| # | Joe 的回答 | 對判定的影響 |
|---|-----------|-------------|
| 1 | 2026-09-01 02:40 UTC 起 ~28h;Fedora 44 原生(非 WSL2)、16GB、Docker legacy builder;target = 主映像 + `SONIC_BUILD_JOBS=8` + `DEFAULT_CONTAINER_REGISTRY=""`,後期加 `DEB_BUILD_OPTIONS=nocheck` | ③ 類改標;nocheck 側證測試失敗跨機 |
| 2 | `.img.gz` = 435B stub 從未開機;但 850MB `sonic-ot-vs.bin` 為真——手動下載 4asic recovery ISO + FAT32 安裝,開機成功,9/8–9/16 有 live VM session(17 容器、~2.0GB) | 「分支不完整」證實;E19 等價解法跨機印證 |
| 3 | kernel Makefile patch 還在(未 commit,snapshot 20240621) | 去重:留我方 deb.debian.org,他的作為備援記錄 |
| 4 | 建置時已釘 `amd64/debian:bullseye@sha256:da5c2dc5...` | C11 歸因實錘;契約 anchor |
| 5 | sonicstorage 403 當時+現在皆然 | D17 必要,全域 |
| 6 | 他的 mirror 組合當時可用;「現在皆 200」 | 他測的是 Release 索引;pool 檔已 404(我方實測)——時間線一致 |
| 7 | 未踩 C12:kernel 在 bookworm slave 建,bookworm main 1.6.4 免 pin | C12 疑似冗餘 → **E5 實證後確認 pin 仍必要**(bullseye slave 映像建置自身有 `build-dep linux`);他為何可過待 log |
| 8 | UT 不在預設路徑;後期 `nocheck` 繞過打包測試 | D14/D15/D16 跨機證實 |
| 9 | 傾向各自 fork + PR,bundle 離線備援 | 與現狀一致 |
| 10 | 完整 log 可分享(img.gz.log、20+ docker-*.gz.log、debs logs、versions logs、session md) | 追問清單見 §7.2 |

### 7.2 追問清單(給 Joe)

| # | 追問 | 定案什麼 |
|---|------|---------|
| 1 | `git -C src/sonic-bmp ls-files -s debian/rules;ls -l src/sonic-bmp/debian/rules;git -C src/sonic-bmp config core.filemode;ls target/debs/bookworm/ \| grep -i bmp` | bmp exec bit 去留 |
| 2 | `session-ses_f7fd*.md` 全文 | F23–F27 交叉、17 vs 19 容器 |
| 3 | `docker-base-bullseye.gz.log` 中 build-dep linux 段 | 他無 pin 過關之謎 |
| 4 | 一起實測:`curl -I` deb.debian.org security 的 **pool 檔**(非索引) | mirror 認知對齊 |

## 8. 參考來源

- 排解編號(C/D/E/F 系列):`SONiC-OT-VS-Build-Troubleshooting.md` §3、§8
- Joe WIP commit:`git show 189b747f7`;Joe 對問題清單的回答:2026-09-24(全文見對話記錄)
- sonicstorage 403 實測:2026-09-24 雙方各自;deb.debian.org `linux_6.1.94-1.dsc` HTTP 200
- security pool 實測:2026-09-24,`dists/.../Packages.gz` 200 vs `pool/.../libssl1.1_1.1.1n-0+deb11u5_amd64.deb` 404
- E5 實證:2026-09-24,`sonic-slave-bullseye:6b67c9c214e` 容器內無 pin `apt-get build-dep -s linux` → EXIT=100
- 映像指紋:2026-09-24 計算,見 `SONiC-OT-VS-Build-Contract.md` §7
