# SONiC OT-VS 對齊排查 Prompt(給 Joe 機器上的 AI agent)

- 用法:在 `sonic-buildimage` repo 根目錄,把下方圍欄內整段貼給你機器上的 AI agent
- 前置:取得 `tom` 分支(含 docs/ 文檔):
  `git remote add fork https://github.com/hoholee8991/sonic-buildimage.git && git fetch fork tom`
  submodule 的修正 commit 在對應 fork(`https://github.com/hoholee8991/{sonic-linux-kernel,sonic-swss,sonic-bmp}.git`,分支 `tom`)
- 離線備援:bundle 檔(`ot-vs-tom-*.bundle`),先 `git bundle verify <檔>` 確認前置 commit 存在,再 `git fetch <bundle 檔> tom`;submodule 同理
- 配套文檔會由 `tom` 分支的 `docs/` 帶入,agent 應先讀再動手

```
你是 SONiC 建置排查 agent。任務:查清「同一條 ot-vs 分支,在不同機器/時間點建置,為何各機器出現不同的本地修改」,並產出你這台機器的修改分類報告,供與 tom 對齊。

## 鐵律
1. 本次為唯讀排查:禁止 reset / checkout / clean 丟掉任何現有狀態;所有結論必須附指令輸出佐證,禁止臆測
2. 若需重跑建置:一律 setsid 背景執行,log 寫 /home/<user>/,不寫 /tmp
3. 分類必須沿用 tom 的四類定義(見文檔 §3),不要發明新類別

## 第 0 步:讀文檔(repo 內 docs/)
- docs/SONiC-OT-VS-Build-Fixes-Analysis.md — tom 側完整歸因:四類分類法、證據、已 commit 清單
- docs/SONiC-OT-VS-Build-Contract.md — 已釘死的變數(Debian snapshot 時戳、submodule SHA、驗收標準、映像 sha256)
- docs/SONiC-OT-VS-Joe-Alignment-Brief.md — tom 方要問你的 10 題與 4 個決策點,邊查邊準備答案
- docs/SONiC-OT-VS-Build-Troubleshooting.md — 排解全文(C/D/E/F 編號被以上文檔引用)

## 第 1 步:定位你的基準
git log -3 --format='%h %ad %an %s%n%b---'
git branch -vv
git status --porcelain=v1
git log --oneline -1 origin/otn_pre_202411        # 應為 189b747f7(Joe WIP)
git show --stat HEAD
特別驗證:git show HEAD --stat | grep sonic-linux-kernel
→ 若為空,即「commit message 宣告修 sonic-linux-kernel/Makefile 但未包含」在你機器上同樣成立

## 第 2 步:盤點你機器的本地修改
git diff --stat
git status --porcelain=v1 | grep '^??'
git submodule foreach --recursive 'git status --porcelain'
對每個 M 分類:
- superproject 檔案 M → 真修改,git diff 逐檔讀
- submodule M → 進入判別:git -C <sub> diff --name-status(真內容修改)vs 僅 untracked(建置殘留)vs index mode 120000(symlink 幻影)
- Windows 端 clone 遺毒檢查:git ls-files --eol 查 CRLF;git ls-files -s | awk '$1==120000' 對照 symlink 是否變純文字檔

## 第 3 步:與 tom 分支比對
git remote add fork https://github.com/hoholee8991/sonic-buildimage.git      # 已存在則略過
git fetch fork tom
git log origin/otn_pre_202411..fork/tom --oneline             # 逐條 commit
git diff origin/otn_pre_202411..fork/tom --stat               # 檔案級
git diff origin/otn_pre_202411..fork/tom --submodule=log      # submodule 級(修正 commit 在 hoholee8991 的三個 submodule fork)
對每個 commit git show <sha>,對照你機器有無等價修改:
- 有 → 確認同樣必要(歸類應相同)
- 無 → 記錄「tom 有、你沒有」→ 時間/環境因素候選

## 第 4 步:時間因素實證
- curl -sI https://sonicstorage.blob.core.windows.net/debian-security/pool/updates/main/l/linux/linux_6.1.94-1.dsc
  → 403 network security perimeter = 全域事件,非單機問題
- curl -sI https://deb.debian.org/debian/pool/main/l/linux/linux_6.1.94-1.dsc → 200 = 替代源可用
- docker images --digests | grep -E 'debian|slave' → 你當時的 slave 基底映像 digest
- 你的建置日期 vs bullseye EOL(2026-08-31)→ 決定你當時的 mirror 狀態

## 第 5 步:環境因素實證
- 建置 log 中的測試結果:grep -c 'FAIL LOG END' <log>;grep -iE 'sonic-config-engine|ssg_main|mock_tests' <log>
- ls -la /dev/kvm → 編譯有無 KVM 加速;無則 TCG,E21(timeout)才是必要修改
- cat /etc/wsl.conf 2>/dev/null;uname -r → 是否 WSL2 環境

## 第 6 步:產出報告
1. 表:你的每個本地修改 × 四類(④分支缺陷 / ②時間 / ③環境 / ①多餘)× 指令證據 × 與 tom 對應 commit 的關係(相同/等價/無)
2. 回答 Alignment-Brief §2 的 10 題(逐題,附證據)
3. 「雙方都主張必要但作法不同」清單(例:kernel URL 你用 snapshot.debian.org、tom 用 deb.debian.org)→ 對齊會議議程
4. 你機器上「tom 沒有、你有」的獨有修改(如有)
```
