# SONiC OT-VS 對齊會議簡報(給 Joe)

- 目的:30 分鐘內完成雙方建置條件對齊——結論先行、問題清單、待決策點
- 日期:2026-09(待約)
- 配套:全文分析見 `SONiC-OT-VS-Build-Fixes-Analysis.md`;排解全文見 `SONiC-OT-VS-Build-Troubleshooting.md`

## 0. 開始前:取得 tom 分支與文檔

| 步驟 | 指令 |
|------|------|
| 1. superproject 取分支 | `git remote add fork https://github.com/hoholee8991/sonic-buildimage.git && git fetch fork tom` |
| 2. 檢出 | `git checkout FETCH_HEAD` 或 `git worktree add ../tom-check FETCH_HEAD`(唯讀排查建議 worktree,不動現有工作樹) |
| 3. submodule 修正 commit | `sonic-linux-kernel`、`sonic-bmp`:`git -C src/<repo> fetch https://github.com/hoholee8991/<repo>.git tom`(sonic-swss 已無差異 commit,不需) |
| 4. 讀文檔 | repo 內 `docs/`:Build-Fixes-Analysis(歸因)、Build-Contract(釘死變數)、Joe-Investigation-Prompt(餵 AI agent)、Build-Troubleshooting(排解全文) |

- 離線備援:4 個 `ot-vs-tom-*.bundle`,先 `git bundle verify <檔>` 再 `git fetch <bundle> tom`;submodule 同理
- 排查動作:把 `Joe-Investigation-Prompt.md` 圍欄內整段貼給你機器上的 AI agent

## 1. TL;DR(三句話)

1. `189b747f7`(你的 WIP)只能編出 435-byte stub 映像——`build_kvm_image.sh` 內有 CI 插入的 `exit 0`;我方在其上補了 6 個 superproject commit + 3 個 submodule commit(分支 `tom`),才編出可開機的真映像
2. 補的修改分四類:分支缺陷(任何人都需要)、Debian EOL 時間因素(已用 snapshot 釘死)、本機環境因素(WSL2/Windows)、可選項(6asic 目錄等)
3. 對齊方式:按 §0 取得 `tom` 分支後逐條 review commit + 回答 §2 問題清單 + 對 §3 決策點與 Build Contract 達成共識

## 2. 問題清單(請 Joe 提供)

> **2026-09-24:Joe 已逐題回答**,要點與判定影響見 `SONiC-OT-VS-Build-Fixes-Analysis.md` §7;本章保留原文供追溯。

| # | 問題 | 驗證什麼 |
|---|------|---------|
| 1 | 你實際建置的日期、機器(OS/CI?)與 target(主映像 or `sonic-4asic-ot-vs`)? | 歸因「時間因素」的時間窗;E19 是否你根本沒踩到 |
| 2 | 你的映像有實際開機過嗎? | 你的樹留有 `exit 0` stub——若從未開機,「分支不完整」推論直接成立 |
| 3 | `sonic-linux-kernel/Makefile` 修復宣告在 message 但不在 commit 裡——你的本機未 commit patch 還在嗎? | 關閉「message 與實作不一致」疑點 |
| 4 | 你建置時 `debian:bullseye` 基底映像的 digest? | 基底映像漂移(C11)歸類時間因素 |
| 5 | 你的網路直連 `sonicstorage.blob.core.windows.net` 可達嗎(當時/現在)? | 403 perimeter 是否全域(實測我方仍 403) |
| 6 | 你建置時 `archive.debian.org` main + `deb.debian.org` security 的組合可用嗎? | mirror 狀態時間線;你的 commit 用的正是這組合 |
| 7 | 編 bullseye kernel 時 libtracefs/libtraceevent >= 1.3 你怎麼解的? | C12(backports pin)是否你也要 |
| 8 | sonic-config-engine / systemd-sonic-generator / swss mock 的單元測試,你有跑過嗎?結果? | D14/D15/D16 是分支缺陷還是環境差異 |
| 9 | submodule 修正(sonic-linux-kernel / sonic-swss / sonic-bmp)的推送流程偏好:各自 fork + PR?還是 bundle 進 superproject? | submodule commit 目前只在我方本機,Joe 無法取得 |
| 10 | 有建置 log 可以分享嗎? | 對照雙方 build 條件 |

## 3. 決策點(會議上要拍板)

| # | 決策 | 選項 |
|---|------|------|
| 1 | 建置目標範圍 | ✅ 已定案(2026-09-24):主 + 4asic(6asic 移除——其平台目錄實為 4asic 複製品,`NUM_ASIC=4`,名實不符) |
| 2 | 測試政策 | ✅ 已執行(2026-09-24):統一 knob——python wheel 用 `_TEST = n`、deb 用 `_DEB_BUILD_OPTIONS = nocheck`(上游先例 otairedis/sairedis/syncd) |
| 3 | 修改合併方式 | `tom` 分支 push 到 origin / 個人 fork / PR 回 sonic-otn 上游 |
| 4 | 基底映像釘選 | ✅ 已執行(2026-09-24):bullseye 釘 `da5c2dc5...`(amd64);bookworm/buster 待辦 |

## 4. Troubleshooting 摘要(與 Joe 相關的 12 條)

全文與原始碼落點見 `SONiC-OT-VS-Build-Troubleshooting.md` §3、§8。

| 編號 | 症狀 | 根因 | 解法 |
|------|------|------|------|
| C9/C10 | bullseye security/backports 套件 404 | EOL 後 pool 清空、backports 退役 | snapshot 釘 `20260601` + backports `20241001` append |
| C11 | libssl/perl 版本錯位,broken packages | archive 內部版本不一致 | 同上 snapshot 釘選 |
| C12 | libtracefs-dev >= 1.3 裝不到 | bullseye main 太舊、backports 已退役 | `backports-pin` + 舊 snapshot |
| D17 | linux-headers 下載 404 | sonicstorage 403(Azure perimeter,全域) | URL 改 `deb.debian.org`(bookworm main pool 已收 6.1.94-1) |
| E18 | `.img.gz` 只有 435 bytes | CI commit `67a8f6676` 插入 `exit 0` stub | 刪除之 |
| E19 | ONIE 無限重探 | 主映像 recovery ISO 機器不在安裝白名單 | 主映像改用 4asic recovery ISO |
| E20 | 4/6asic recovery ISO 缺檔 | 非任何 make target 的依賴 | 自 `ot_kvm_onie` release 下載 |
| E21 | pexpect timeout | 無 KVM 時 TCG 過慢 | `-enable-kvm` 條件加速 + timeout 3600s |
| E22 | make 說 up to date | 壞產物 mtime 較新 | 刪產物重跑 |
| F23/F24 | `sonic-db-cli PING` 連不上、syncd-ot 卡死 | 多重 redis 下 PING 把 DB 名當埠 | `syncd_common.sh` 改查 `CONFIG_DB_INITIALIZED` |
| F25–F27 | per-asic CONFIG_DB 空/旗標不設 | `$(var)` 當指令、`{}` 空檔炸 HWSKU 提取、`-n` 跳過旗標設定 | `config-setup` 三處修正 |
| F28/F29 | flock 殭屍、重開機又掛 | 前次失敗程序握鎖;開機重設旗標 | 清程序;per-asic 旗標迴圈(已持久化於映像) |

## 5. 參考來源

- 修改歸因與證據:`SONiC-OT-VS-Build-Fixes-Analysis.md`
- 釘死變數與驗收:`SONiC-OT-VS-Build-Contract.md`
- 排解全文:`SONiC-OT-VS-Build-Troubleshooting.md`
