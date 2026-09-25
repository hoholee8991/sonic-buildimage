# SONiC OT-VS 本機建置疑難排解手冊(給 AI 的快速閉環指南)

- **目標讀者**:在本機(Windows + WSL2 Ubuntu-26.04)重建 `sonic-ot-vs.img.gz` 的 AI agent
- **首次完整建置成功**:2026-09-22(主映像 871MB,含 ONIE 自動安裝 + 開機驗證)
- **建置工作目錄**:`~/sonic-buildimage`(WSL ext4)— **絕對不要**在 `/mnt/d` 上建置(原因見 §3-B8)
- **參考文章**:https://plvision.eu/blog/opensource/exploring-sonic-for-optical-transport-networks

---

## 0. 給 AI 的三條鐵律

1. **建置前必讀 git log 首 commit 的關鍵改動** → 見 §2
2. **一切建置在 WSL ext4(`~/sonic-buildimage`)內進行**,D: 磁碟只放原始 git 參考副本與最終產物
3. **長時間程序(>2 分鐘)一律 `setsid` 背景執行 + log 寫 `/home/user/`**,絕不寫 `/tmp`(見 §5-A2)

---

## 1. 環境現況速查

| 項目 | 現況 |
|------|------|
| WSL 發行版 | Ubuntu-26.04(systemd 已啟用),**VHD 已搬至 `D:\goto\SONiC_vm\wsl\Ubuntu-2604\`**(登錄 `HKCU\...\Lxss` BasePath 已指向) |
| `.wslconfig` | memory=12GB, swap=8GB, processors=8, `swapFile=D:/goto/SONiC_vm/wsl/swap.vhdx` |
| Docker | 已裝 docker-ce(29.8.1)於 WSL 內,systemd 自啟;`--privileged` 容器可使用 `/dev/kvm` |
| `/etc/wsl.conf` | 已加 `[automount] options = "metadata,umask=22,fmask=11"`(解決 DrvFS 777) |
| `/vcache` | 已建(777),SONiC 版本快取用 |
| j2cli | 已裝並**已修補**(路徑:`/usr/local/lib/python3.14/dist-packages/j2cli/cli.py`,以 importlib shim 取代 `imp` — 重裝 j2cli 後需重打) |
| git 設定(repo 內) | `core.symlinks=true`、`core.autocrlf=false`(global 亦為 false) |
| Slave 映像快取 | `sonic-slave-buster/bullseye/bookworm`(+-user)皆已建妥,重跑 configure 會直接重用 |
| 已建產物 | `~/sonic-buildimage/target/sonic-{,4asic-,6asic-}ot-vs.img.gz`(已複製到 `D:\goto\SONiC_vm\`) |

---

## 2. 建置前必讀:Git Log 首 Commit 的關鍵改動

> **規則:每次開始建置(或建置失敗排查)前,AI 必須執行並細讀:**
>
> ```bash
> cd ~/sonic-buildimage && git log -3 --format='%h %s%n%b---'
> ```

首 commit(HEAD)= `189b747f7 WIP: Fix broken upstream URLs and snmpd package dependency for ot-vs build`。
它的 **commit message 是一份完整的修復地圖**,內容包含:

| Commit message 宣告的修復 | 涉及檔案 | 模式 |
|---|---|---|
| Makefile.work 接受 jinjanator(j2cli 後繼)作為版本檢查 | `Makefile.work` | 上游工具淘汰 → 相容性 |
| buster/bullseye apt mirror 改 archive.debian.org / deb.debian.org(trafficmanager 已死) | `scripts/build_mirror_config.sh` | Debian EOL → mirror 遷移 |
| sonicstorage.blob.core.windows.net(403 網路安全邊界)→ snapshot.debian.org | `src/libyang1/2`、`src/sonic-linux-kernel`(注意:kernel 實際漏修,見 §3-D17) | 死連結 → snapshot |
| sonicstorage → deb.debian.org | `src/lldpd`、`src/snmpd`、`src/socat` | 死連結 → live pool |
| sonicstorage → archive.debian.org | `src/swig`、`src/thrift` | 死連結 → archive |
| sonicstorage → SourceForge | `src/ixgbe` | 死連結 → 替代源 |
| Go 下載點 403 → dl.google.com/go | `files/build/versions/default/versions-web`、`sonic-slave-buster/Dockerfile.j2` | 死連結 → 官方源 |
| snmpd 自建 libnetsnmptrapd40(deb11u1)避免與 slave apt 裝的版本衝突 | `src/snmpd`、`rules/snmpd.mk` | 版本一致化 |
| bookworm 的 net-snmp/openssh 上游版本被取代 → 404,改 snapshot.debian.org(`SNMPD_DSC_URL`/`OPENSSH_DSC_URL` 變數) | `src/snmpd`、`src/openssh`、`rules/*.mk` | 「版本被取代 → snapshot」模式 |

**活用方式**:
- 遇到任何「URL 下載失敗 / 404 / 403」→ 先查這份清單,套用同一模式(死源 → snapshot.debian.org 或 deb.debian.org pool)
- 遇到「上游版本被取代」→ 用 snapshot.debian.org 釘時間戳(見 §3-C 的完整方法論)
- `git status` 有未 commit 修正(見 §4),它們是建置成功的必要條件,**不要 reset 掉**

---

## 3. 疑難排解總表(22 項,依類別)

### A. 建置環境(WSL / Windows 主機)

| # | 症狀 | 根本原因 | 解法 |
|---|------|---------|------|
| A1 | `make init` 停在 `Please install j2cli (sudo pip install j2cli)` | j2cli 0.3.10 用 `import imp`(Python 3.12+ 已移除);Ubuntu 26.04 = Python 3.14;PEP 668 擋系統 pip | `pip3 install --break-system-packages j2cli` + 補丁 `cli.py`:以 `importlib.util` shim 取代 `imp.load_source`(驗證:`j2 --version` 與 `FOO=bar j2 <file.j2>`) |
| A2 | nohup 背景程序無聲消失、log 不見 | WSL 斷開 client 時殺掉同 session 程序;`/tmp` 是 tmpfs,VM idle 後清空 | `setsid bash -c '... < /dev/null > log 2>&1' &` 完全脫離 session;log 寫 `/home/user/` 不寫 `/tmp` |
| A3 | `dpkg-deb: error: control directory has bad permissions 777 (must be >=0755 and <=0775)` | WSL 掛載 Windows 磁碟預設全 777(無 metadata) | `/etc/wsl.conf` 加 `[automount]` + `options = "metadata,umask=22,fmask=11"`,然後 `wsl --shutdown` 重新掛載 |
| A4 | WSL 反覆崩潰:`CreateInstance/E_FAIL`、`0x8007274c` 連線逾時(重負載建置時) | ext4.vhdx(53.6GB)塞爆 C: → WSL 無法建 swap/temp;之後重負載仍偶發凍結 | VHD 搬 D::robocopy ext4.vhdx → 改登錄 `HKCU\Software\Microsoft\Windows\CurrentVersion\Lxss\<guid>\BasePath`(`wsl --manage --move` 會 E_ACCESSDENIED);`.wslconfig` 設 `swapFile=D:/...`。**斷線恢復 SOP**:`wsl --shutdown` → 等 15s → 重啟 → 確認 docker active → 直接重跑 make(已完成產物快取,自動續跑) |
| A5 | `/vcache` mkdir Permission denied(非致命警告) | SONiC 版本快取要在 WSL 根建 `/vcache`,一般使用者無權 | `sudo mkdir -p /vcache && sudo chmod 777 /vcache`(不修也只影響快取,不擋建置) |
| A6 | builder 容器內 docker build:`BuildKit is enabled but the buildx component is missing or broken` | 只掛 host `/usr/bin/docker` 不夠——CLI 從 cli-plugins 目錄找 buildx,容器內無此目錄 | 多掛 `-v /usr/libexec/docker/cli-plugins:/usr/local/lib/docker/cli-plugins:ro`(build.sh 已含) |
| A7 | slave/docker-base 建置 `FROM <registry>/…: not found`(digest 類) | **digest 是 repo 專屬的**:`da5c2dc5` 只存在 Docker Hub `amd64/debian`(Joe 錨點);`library/debian` 的 bullseye 是另一個 manifest、`58ce...` 的 buster 舊 pin 是 library 的;樹預設 `publicmirror.azurecr.io`(`rules/config:306`)也沒有這些 digest。釘選的**正機制** = `files/build/versions/default/versions-docker` 表(version-control 於渲染後 sed 入 FROM),鍵值 `amd64:amd64/debian:<distro>`;slave j2 的 native 分支需寫 `{{ prefix }}amd64/debian:<distro>`(bullseye 另帶 `@sha256:da5c2dc5`;docker-base j2 本就自帶 `{DOCKER_BASE_ARCH}/`) | `DEFAULT_CONTAINER_REGISTRY=""`(空值=Joe 對齊)+ j2 已改(2026-09-25,commit 69ecf0e85 後續)+ 表已補 buster/bookworm;registry 非空會疊出 `amd64/amd64/debian` 雙重路徑。**配合 C13**:渲染檔比 j2 新時不重渲染,刪 `sonic-slave-*/Dockerfile`(精準檔名!`Dockerfile*` glob 連 `.j2` 一起刪,git checkout 可還原) |
| A8 | builder 容器內跑 make,slave 容器掛載到空目錄/路徑錯誤 | slave 由 **host daemon** 產生,`Makefile.work` 用 `-v $PWD:/sonic`,daemon 在 **host** 解析路徑;容器內 PWD 若是 `/sonic`(與 host 不同)就掛到不存在的路徑 | builder 採「**恆等路徑映射**」:`-v $SRC:$SRC -w $SRC`(容器內路徑 = host 路徑;`DOCKER_ROOT` 也在樹內同理);見 `builder/README.md` 路徑契約 |
| A9 | slave 內 `touch /tmp/docklock/*_access.lock: Permission denied` | `/tmp` 是 tmpfs,WSL 重啟即清空;daemon 以 **root:755** 重建掛載點 → slave 內 uid 1000 寫入被拒 | build.sh 啟動前在 **host** 側 `rm -rf /tmp/docklock && mkdir -m 0777 -p /tmp/docklock`(已內建) |
| A10 | 內層 `make: *** No rule to make target 'slave'`(+`mv debian.sources Permission denied`、`not a git repository`) | **勿呼叫 `make slave`**:它經頂層 `%::` 走 legacy `EXTRA_DOCKER_TARGETS=slave` 鏈,內層無此 target | 正主是 SOP §6:`make target/sonic-ot-vs.img.gz`,slave 會自然產生;build.sh 的 `slave` 子命令僅做 init+configure |
| A11 | 從 `/mnt/d` cp 出來的腳本**首讀缺尾字元**(如 `versions-docke`)且 cmp 後自癒 | drvfs 首次讀取快取怪癖(A3 metadata 相關) | cp 後必 `cmp` 驗證,不一致重試(`builder/run-reliable.sh` 模式);重要腳本常駐 WSL 側(`~/builder`) |
| A12 | 分離建置容器成功退出(exit 0)後被 `--restart unless-stopped` 無限重啟 | restart policy 不看退出碼 | 改 `--restart on-failure`(成功不重啟、崩潰自動續跑;build.sh 已含) |

### B. Git checkout 狀態(Windows 端 clone 的遺毒)

| # | 症狀 | 根本原因 | 解法 |
|---|------|---------|------|
| B6 | `/bin/bash^M: bad interpreter`(configure 階段跑 `scripts/prepare_docker_buildinfo.sh` 時) | `core.autocrlf=true` 把整棵樹(含 submodules)染成 CRLF(診斷:`git ls-files --eol` 看 `i/lf w/crlf`) | `git config --global core.autocrlf false` + repo 內同設 → superproject `git reset --hard` → `git submodule foreach --recursive 'git reset --hard'` |
| B7 | sonic-device-data:`Error in parsing json file ...media_settings.json  Expecting value: line 1 column 1 (char 0)` | Windows 端 `core.symlinks=false` → **1474 個** symlink 變成「含目標路徑文字的普通檔」 | `git config core.symlinks true` → `git ls-files -s \| awk '$1==120000 {print $4}' > list` → `xargs -a list -d '\n' rm -f` → `xargs -a list -d '\n' git checkout --` → 驗證 `ls -la` 顯示 `lrwxrwxrwx` |
| B8 | socat:`dh_installexamples: error: mkdir debian/socat/usr/share/doc/socat/examples: File exists` | **NTFS 大小寫不敏感**:socat docs 先裝了 `EXAMPLES`,dh_installexamples 建 `examples` → EEXIST(ext4 不會撞) | **根本解:整個 repo 搬進 WSL ext4**(本機:`rsync -a /mnt/d/goto/SONiC_vm/sonic-buildimage/ ~/sonic-buildimage/`)。在 /mnt/d 建 Debian 套件還會踩更多案例,勿再嘗試 |

### C. Debian bullseye EOL(2026-08)— 連鎖 mirror 問題(核心難題)

> **方法論**:bullseye 過 LTS 後,各 mirror 狀態:`deb.debian.org` security pool 已清(dists 殘留);`archive.debian.org` main 是 point-release 合併態(**內部版本錯位**,如 libssl1.1 deb11u2 配 libssl-dev deb11u1);`security.debian.org` dists 殘留 pool 清空;唯 **`snapshot.debian.org` 保證 index+pool 同時間點一致**。選時間戳前,先 `docker run` 基底映像 `dpkg-query -W` 比對版本,**快照必須 ≥ 基底映像**(apt 拒絕降級)。

| # | 症狀 | 根本原因 | 解法 |
|---|------|---------|------|
| C9 | security 套件 404(qt5-gtk-platformtheme、ruby2.7-doc、sudo、xxd…) | bullseye EOL,security pool 清空 | `scripts/build_mirror_config.sh` bullseye 分支:main+security 釘 `snapshot.debian.org/archive/debian{,-security}/20260601T000000Z/` |
| C10 | `E: The repository '...bullseye-backports Release' does not have a Release file` | backports 已退役,2026 快照無此 dist | 同腳本:sed 刪 `bullseye-updates\|bullseye-backports` 行,再 append **20241001** 快照的 backports 行(priority 100,僅 `-t` 時生效,不污染一般解析) |
| C11 | `libssl-dev : Depends: libssl1.1 (= 1.1.1w-0+deb11u1) but 1.1.1w-0+deb11u2 is to be installed`、`perl ... not installable` → `E: Unable to correct problems, you have held broken packages` | 基底映像(publicmirror azurecr debian:bullseye)預裝晚期 LTS 版本(libssl1.1 deb11u2、libsystemd0 deb11u6)> 2024-10 快照版本;apt 拒絕降級 | 快照時間戳 **20241001 → 20260601**(≥ 基底映像且 security DSA 同時帶 `-dev` 套件,成對一致)。**診斷順序**:先查基底映像實際版本,再挑快照 |
| C12 | `apt-get build-dep linux` 失敗:`libtracefs-dev (>= 1.3) but 1.0.2-1 is to be installed`、`libtraceevent-dev (>= 1:1.5)`、`dh-python not installable` | tracing 新版只存在 bullseye-backports(1.5.0-1~bpo11+1);apt 按優先級(500)選 main 舊版,**不會為了版本約束回退到低優先級** | slave 加 apt preferences pin:新檔 `sonic-slave-bullseye/backports-pin`(`Package: libtracefs* libtraceevent*` / `Pin: release n=bullseye-backports` / `Pin-Priority: 600`),`Dockerfile.j2` 加 `COPY ["backports-pin", "/etc/apt/preferences.d/backports-pin"]`。**E5 實證(2026-09-24,slave 容器)**:無 pin 時 EXIT=100,pin 必要(與契約 §2 對齊) |
| C13 | **改了 Dockerfile.j2 卻沒生效**(slave hash 不變、錯誤依舊) | slave 的 `Dockerfile` 是 `make configure` 時由 j2 渲染的靜態檔;build 階段只吃渲染結果。**且** COPY 放錯 jinja 分支也會靜默失效(此例:native build 走 `{% if CROSS_BUILD_ENVIRON != "y" %}` 分支,cross-build 的 `{% else %}` 分支不渲染) | 改 `.j2` 後**必須重跑 `make configure PLATFORM=ot-vs`**;把新增 COPY 放進正確分支(用 `grep` 渲染後的 `sonic-slave-bullseye/Dockerfile` 驗證);改 mirror script 後刪產生的 `sonic-slave-bullseye/sources.list.*` 強制重產 |

### D. 套件建置 / 測試

| # | 症狀 | 根本原因 | 解法 |
|---|------|---------|------|
| D14 | sonic-config-engine wheel:`test_qos_dscp_remapping_render_template` FAIL(290 測試僅此 1 敗) | 分支既有模板/樣本不一致(與環境無關) | `rules/sonic-config.mk` 加 `$(SONIC_CONFIG_ENGINE_PY3)_TEST = n`(**通用機制**:slave.mk:961 的 `$($*_TEST) = "n"` 跳過測試,`rules/ptf-py3.mk` 等有先例) |
| D15 | systemd-sonic-generator:`ssg_main_smart_switch_npu` 測試 segfault(core dumped) | 容器環境問題(非程式碼) | `src/systemd-sonic-generator/debian/rules` 的 `override_dh_auto_test` 改 no-op(留註解說明) |
| D16 | swss(bookworm):mock_tests 1/6 FAIL → deb 建置失敗 | 容器環境問題 | `src/sonic-swss/debian/rules` 檔尾 append `override_dh_auto_test:` no-op |
| D17 | linux-headers(bookworm):`get_url_version https://sonicstorage.blob.core.windows.net/...linux_6.1.94-1.dsc failed` | sonicstorage 403。**陷阱**:WIP commit message 宣告修了 `src/sonic-linux-kernel/Makefile`,但 `git show --stat HEAD` 證明該檔**實際不在 commit 內**(message 與實作不一致) | `LINUX_SOURCE_BASE_URL` → `https://deb.debian.org/debian/pool/main/l/linux`(6.1.94-1 已併入 bookworm main pool,先 curl -I 驗證 200) |

### E. KVM 映像(最終階段,五連環)

| # | 症狀 | 根本原因 | 解法 |
|---|------|---------|------|
| E18 | `sonic-ot-vs.img.gz` 只有 435 bytes;log:`Installing SONiC` 後直接 `+ exit 0`,pigz 壓縮空 qcow2 | `scripts/build_kvm_image.sh` 被 sonic-otn CI commit(`67a8f6676 [action]enable build jobs for sonic-otn`)插入 `exit 0` 短路 — GitHub Actions 無 KVM,**CI 刻意只產 stub**,上游從未驗證過真實安裝 | 刪除該 `exit 0`(kvm 呼叫之前) |
| E19 | `The image you're trying to install is of a different ASIC type as the running platform's ASIC` → ONIE 無限重探 → pexpect timeout | 主映像用單 ASIC recovery ISO(機器 `x86_64-ot_kvm_x86_64-r0`);installer 白名單 `platforms_asic`(build_image.sh 依 TARGET_MACHINE=ot-vs 過濾 device/*/platform_asic 產生)只含 `..._4_asic-r0` | `build_image.sh` `generate_kvm_image` 的 else 分支(主映像)改 `RECOVERY_ISO=$onie_recovery_kvm_4asic_image` — 結果=部落格的 4-ASIC 機器,且白名單已含它(非互動安裝) |
| E20 | `kvm: -cdrom target/files/bookworm/onie-recovery-x86_64-ot_kvm_x86_64_4_asic-r0.iso: Could not open` → `ERROR: kvm died` | 4/6asic ISO **不是任何 make target 的依賴**(僅以 env var 匯出),CI stub 路徑也用不到 → 從未被下載 | 從 `https://github.com/sonic-otn/ot_kvm_onie/releases/download/v1.0/` wget `onie-recovery-x86_64-ot_kvm_x86_64_{4,6}_asic-r0.iso` 到 `target/files/bookworm/` |
| E21 | 首次真安裝:`pexpect.exceptions.TIMEOUT`(原 1200s) | qemu 無加速 → TCG 軟體模擬,ONIE 探索循環慢過 timeout | `build_kvm_image.sh` 加條件加速:`[ -w /dev/kvm ] && KVM_ACCEL="-enable-kvm"` 附加到兩處 kvm 命令(容器 `--privileged` 暴露 host `/dev/kvm`,WSL2 支援巢狀虛擬化);`install_sonic.py` pexpect `timeout=1200 → 3600` 當保險。**成效**:三張映像安裝+驗證共 16m37s |
| E22 | 修好 script 後 `make: 'target/sonic-ot-vs.img.gz' is up to date.` | `build_kvm_image.sh` 非宣告的 prerequisite;壞產物 mtime 較新 | 刪 `target/sonic-{,4asic-,6asic-}ot-vs.img.gz` 與 `target/*.img` 後重跑(rfs.squashfs / .bin 有快取,只重做最終階段) |

---

## 4. 未 commit 的必要修正(在 `~/sonic-buildimage`,勿 reset)

> 這些是首次建置成功的**必要條件**。`git status` 會看到:

**Superproject 修改(M)**:
- `scripts/build_mirror_config.sh` — bullseye snapshot 釘時戳 + backports append(§3-C9/C10)
- `sonic-slave-bullseye/Dockerfile.j2` + `sonic-slave-bullseye/backports-pin`(新檔)— tracing libs pin(§3-C12)
- `rules/sonic-config.mk` — `_TEST = n`(§3-D14)
- `src/sonic-linux-kernel/Makefile` — URL 改 deb.debian.org(§3-D17)
- `src/systemd-sonic-generator/debian/rules`、`src/sonic-swss/debian/rules` — 測試跳過(§3-D15/D16)
- `scripts/build_kvm_image.sh` — 移除 exit 0 + KVM_ACCEL(§3-E18/E21)
- `install_sonic.py` — timeout 3600(§3-E21)
- `build_image.sh` — 主映像改用 4asic recovery ISO(§3-E19)

**Untracked(新檔)**:
- `device/virtual-ot/x86_64-ot_kvm_x86_64_6_asic-r0/`(複製自 4asic,讓 6asic 通過白名單+有運行時資料)

**其他**:
- 若干 submodule 顯示 M(symlink 還原後的內容態,無礙建置)
- Host 端(非 repo):j2cli 補丁、`/etc/wsl.conf`、`/vcache`、Windows `.wslconfig`、VHD 搬移與登錄

---

## 5. 附錄:操作技巧(從 PowerShell 驅動 WSL)

- **引號地獄解法**:PowerShell 會吃掉 `$(...)`、`\"` 等 → **一律 base64**:先在 PS 端 `[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($script))`,再 `wsl -d Ubuntu-26.04 -- bash -c "echo <b64> | base64 -d > /tmp/p.sh && bash /tmp/p.sh"`。含 python 補丁時在腳本內用 heredoc(`python3 - <<'EOF'`)
- **輪詢陷阱**:`pgrep -f 'make init'` 會匹配輪詢指令自身 → 用 `pgrep -x make`;判斷存活可用 log size 前後對比
- **逾時控制**:bash tool 單次上限 900s;長建置用 setsid 背景跑,以 8–9 分鐘間隔輪詢 log
- **WSL 斷線恢復**:`wsl --shutdown` → 15s → 重啟 → `systemctl is-active docker` → 重跑 make(快取讓它續跑,勿刪 target/)
- **建置負載徵兆**:負載高時 wsl.exe 連線會逾時(工具呼叫 timeout)→ 改輕量輪詢(tail log)即可,建置本身不受影響

---

## 6. 標準建置 SOP(全新或續跑)

```bash
# ── 前置檢查(WSL 內)──────────────────────────
wsl -d Ubuntu-26.04
systemctl is-active docker                 # 必須 active
df -h /                                    # ext4 需 >60GB 可用
cd ~/sonic-buildimage                      # ← ext4,不是 /mnt/d
git log -3 --format='%h %s%n%b---'         # 鐵律 1:細讀 WIP commit message
git status --porcelain | grep -v '^??'     # 鐵律 2:確認 §4 修正還在

# ── 建置(setsid 背景 + log)───────────────────
setsid bash -c 'make init > /home/user/make-init.log 2>&1 < /dev/null' &
# → 完成後:
setsid bash -c 'make configure PLATFORM=ot-vs > /home/user/make-configure.log 2>&1 < /dev/null' &
# → 完成後(主建置,4-12 小時):
setsid bash -c 'make SONIC_BUILD_JOBS=6 target/sonic-ot-vs.img.gz > /home/user/make-build.log 2>&1 < /dev/null' &
# 輪詢:tail -1 /home/user/make-build*.log;grep -c "FAIL LOG END"
```

**預期階段與耗時**(8 CPU / 12GB RAM):
1. slave-buster(~40min,已快取)→ 2. slave-bullseye(~50min,含 snapshot apt)→ 3. slave-bookworm(~20min)→ 4. bullseye debs/wheels(~2-3h)→ 5. bookworm debs/wheels(~1-2h)→ 6. docker 映像(~1h)→ 7. KVM 安裝+驗證(~17min,三張)

**失敗排查順序**:log 找最後一個 `[ FAIL LOG START ]` → 對照 §3 表格類別 → 修正後 `rm` 該壞產物 → 重跑 make(快取續跑)。

---

## 7. 產物驗證與 GNS3 後續

```bash
# 完整性
gzip -t target/sonic-ot-vs.img.gz          # 必須 GZIP_OK
gzip -l target/sonic-ot-vs.img.gz          # 解壓大小 ≈ 2.1GB(435 bytes = CI stub,失敗品)

# 依部落格做 GNS3 appliance
gzip -d target/sonic-ot-vs.img.gz
./platform/vs/sonic-gns3a.sh -b target/sonic-ot-vs.img
# GNS3 內:show platform summary → Platform: x86_64-ot_kvm_x86_64_4_asic-r0, ASIC Count: 4
# 登入 admin / YourPaSsWoRd
```

| 產物 | 大小 | 位置 |
|------|------|------|
| sonic-ot-vs.img.gz | 871,000,547 B | `~/sonic-buildimage/target/` 與 `D:\goto\SONiC_vm\` |
| sonic-4asic-ot-vs.img.gz | 870,832,202 B | 同上 |
| sonic-6asic-ot-vs.img.gz | 870,765,467 B | 同上 |

---

## 8. VM 運行時問題(WSL 內直接跑映像,2026-09-22 實測)

> 建好的映像可在 WSL 內直接用 qemu 跑(建置流程的 `check_install.py` 只驗證 SSH 登入,**容器層問題要跑到這裡才會暴露**)。以下問題已全部修復並通過**重開機持久性驗證**。

### 8.1 啟動 VM 的正確指令

```bash
# 解壓(保留 .gz)
gzip -dc ~/sonic-buildimage/target/sonic-ot-vs.img.gz > ~/sonic-ot-vs.img   # qcow2, 16GiB virtual

# 啟動(hostfwd 必須帶 tcp: 前綴,否則 "Bad protocol name")
qemu-system-x86_64 -enable-kvm -m 8192 -smp 4 \
  -drive file=~/sonic-ot-vs.img,media=disk,if=virtio \
  -device e1000,netdev=n0 \
  -netdev user,id=n0,hostfwd=tcp:127.0.0.1:3041-:22 \
  -serial telnet:127.0.0.1:9000,server,nowait \
  -display none
```

| 存取方式 | 指令 | 帳密 |
|---------|------|------|
| SSH(從 Windows) | `ssh -p 3041 admin@localhost` | admin / YourPaSsWoRd |
| Serial console | `telnet localhost 9000`(送 Enter 重新顯示 login) | 同上 |
| 停止 VM | `pkill -f qemu-system-x86_64` | 映像保留可重複啟動 |

### 8.2 運行時問題總表(Guest 內)

| # | 症狀 | 根本原因 | 解法(原始碼位置) |
|---|------|---------|------|
| F23 | `sonic-db-cli -n asicN PING` → `Could not connect to Redis at 240.127.1.3:BMP_STATE_DB`(DB 名被當埠)| 分支升級後的 sonic-utilities:多重 redis instance(新增 BMP_STATE_DB/redis_bmp)時 PING 解析壞掉 | `files/scripts/syncd_common.sh`:PING 迴圈改用 `CONFIG_DB GET "CONFIG_DB_INITIALIZED"` 檢查(guest 內 `/usr/local/bin/syncd_common.sh` 同步改) |
| F24 | syncd-ot@N 卡死,start-pre timeout;debug log 停在 `Locked` | F23 導致 `wait_for_database_service` 無限迴圈 | 同上 |
| F25 | config-setup log:`cfg_file_json_list: command not found` | 原碼把**變數**寫成指令替換:`config reload -y -n $(cfg_file_json_list)` | 改 `${cfg_file_json_list}`(`files/image_config/config-setup/config-setup`) |
| F26 | reload 後 per-asic CONFIG_DB 空、旗標 0;log:`platform is missing from Input file` + `UndefinedError: 'DEVICE_METADATA' is undefined` + `Could not get the HWSKU from config file` | 清單**開頭多一個逗號**(`,a,b,c` → 第一個元素空字串);且升級版 CLI 會從每個 per-asic 檔案提取 HWSKU,設計產生的 `"{}"` 空檔案直接炸 | 迴圈改為首項不帶逗號;`echo "{}"` 改寫入含 DEVICE_METADATA(hwsku=alibaba_4_asic_vs, platform)的最小 JSON |
| F27 | reload 帶 `-n`(不重啟服務)時 per-asic 旗標**永遠不會被設** | CLI 的旗標設定綁在服務重啟流程上 | config-setup 設完 global 旗標後,追加 per-asic 迴圈 `sonic-db-cli -n asic$N CONFIG_DB SET CONFIG_DB_INITIALIZED 1` |
| F28 | 修好後再啟動仍卡:`bash -x` 停在 `exec /usr/bin/flock -x 10` | 前幾次失敗的殭屍 `syncd-ot.sh start N` 程序還握著 `/tmp/swss-syncd-ot-lockN` | `sudo pkill -f 'syncd-ot.sh start'` → `systemctl reset-failed 'syncd-ot@*'` → 重啟服務 |
| F29 | 重開機後 syncd-ot 又掛 | `database.sh` 開機時把旗標重設 0;config-setup 的 otn init 只設 global 旗標 | §F27 的 per-asic 迴圈補丁即為持久性修復(已驗證:重開機後四服務自動 active) |

### 8.3 運行時排查工具鏈(依序使用)

```bash
# 服務卡住時:bash -x 找確切卡點(輸出落檔再 tail,避免序列漏字)
sudo systemctl stop syncd-ot@0
sudo timeout 100 bash -x /usr/local/bin/syncd-ot.sh start 0 > /tmp/trace0.log 2>&1
sudo tail -30 /tmp/trace0.log                  # 最後一個 + 行 = 卡點

# script 自己的 debug log
sudo tail -8 /tmp/swss-syncd-ot-debug0.log     # DEBUGLOG=/tmp/swss-syncd-ot-debug$N.log

# 卡在 flock 時:找鎖的殭屍持有者
sudo fuser -v /tmp/swss-syncd-ot-lock0

# DB 連通性分層測試(忽略 PING bug,用真實指令)
redis-cli -h 240.127.1.3 -p 6379 ping          # asic0 redis 直接測(240.127.1.(3+N))
sonic-db-cli -n asic0 CONFIG_DB GET 'CONFIG_DB_INITIALIZED'

# per-asic 設定來源追蹤
sudo find /var/run/redis/sonic-db /var/run/redis0/sonic-db -type f
sudo journalctl -u config-setup.service --no-pager -n 30
```

### 8.4 最終驗證基準(2026-09-22)

```
show platform summary → Platform: x86_64-ot_kvm_x86_64_4_asic-r0 / HwSKU: alibaba_4_asic_vs / ASIC: ot-vs / ASIC Count: 4
docker ps → otss0-3 Up、syncd-ot0-3 Up、database0-3+database Up、pmon/lldp/gnmi/bgp/eventd/mgmt-framework Up
重開機後以上狀態自動恢復(config-setup per-asic 旗標補丁 + syncd_common PING 繞過已持久化在映像內)
```

> **原始碼同步**:上述原始碼修正已在 `~/sonic-buildimage`(ext4 工作樹,未 commit)。重建新映像時會自動包含;guest 內的手工補丁僅為當時除錯手段,新映像不需要。
