# 磁盘深度清理助手（Windows + macOS 跨平台）

## 触发时机
当用户说到以下任意内容时自动触发：
清理磁盘 / 清理C盘 / 清理D盘 / 磁盘空间不够 / 空间不足 / 硬盘满了 / 帮我清理 / disk clean / 清理一下 / 盘快满了

触发后**立即检测系统类型，不问任何前置问题**，告知用户"正在扫描，请稍候…"，随即执行对应脚本。

---

## 第零步：系统识别

```bash
if [[ "$OSTYPE" == "darwin"* ]]; then echo "MACOS"
elif [[ "$OS" == "Windows_NT" ]] || [[ -n "$WINDIR" ]]; then echo "WINDOWS"
fi
```

- WINDOWS → 走 Windows 支线
- MACOS   → 走 macOS 支线

---

# ══════════════════════════════════
# WINDOWS 支线
# ══════════════════════════════════

## W-1：扫描脚本（动态发现）

**设计原则：扫描所有可能存在的路径，只报告实际存在且 > 1MB 的项目，不预设软件列表。**

> ⚠️ **脚本编码铁则（永远不要违反）**：
> 写入 `.ps1` 文件时，脚本体内**禁止出现任何中文/CJK 字符**（包括注释、字符串、变量值）。
> 所有 label/app/note 字段统一使用 ASCII 英文 key。
> LLM 读取 JSON 结果后，在**展示环节**自行将英文 key 翻译成中文显示给用户。
> 违反此规则会导致 Windows PowerShell 以 ANSI 解析 UTF-8 文件，产生乱码和语法错误。

> ⚠️ **脚本写入铁则（Windows Git Bash 环境下必须遵守）**：
> - **优先复用**：先用 Read 工具尝试读取 `%TEMP%\dc_scan.ps1`。若文件已存在且内容正常（非空、无报错），**直接跳过写入步骤，立即运行**，不要重复写入。
> - **需要写入时（首次或文件损坏）**：用 **Write 工具**写入，Write 工具要求先 Read 过才能写，所以上一步的 Read 同时满足此前提。
> - **正确做法备选**：若 Write 工具受限，改用 Python：`python -c "open(r'C:/path/dc_scan.ps1','w',encoding='utf-8').write(r'''<脚本内容>''')"`
> - **严禁 bash heredoc**（`<< 'EOF'`）：脚本中含单引号会被 bash 提前截断，导致写入不完整
> - **严禁 `powershell -Command "...脚本内容..."`**：bash 会将 `$b.n`、`$j.items` 等 PowerShell 属性访问误展开为空值或路径报错
> - **执行方式**：始终用 `-File` 参数，而非 `-Command`：`powershell -ExecutionPolicy Bypass -File "$env:TEMP\dc_scan.ps1"`

先用 Read 检查 `%TEMP%\dc_scan.ps1` 是否已存在，已存在则直接运行；不存在或损坏则写入后运行：

```powershell
# ENCODING RULE: ASCII ONLY in this script. No Chinese/CJK characters anywhere.
# Labels use English keys. The LLM translates them to Chinese when displaying results.
# PORTABILITY: Never hardcode C:\ - use $env:SystemDrive for the system drive letter.

$local=$env:LOCALAPPDATA; $roaming=$env:APPDATA; $temp=$env:TEMP
$user=$env:USERPROFILE
$sd=$env:SystemDrive          # system drive, e.g. "C:" - may vary on different machines
$windir=$env:SystemRoot       # e.g. "C:\Windows"
$prog="$sd\ProgramData"       # C:\ProgramData equivalent

function Get-Size($p){
    if(-not(Test-Path $p)){return 0}
    $s=0
    Get-ChildItem $p -Recurse -Force -EA SilentlyContinue |
        Where-Object{-not $_.PSIsContainer} | ForEach-Object{$s+=$_.Length}
    $s
}

$items = [System.Collections.Generic.List[hashtable]]::new()

function Add-Entry($cat, $app, $label, $path, $admin=$false, $proc="") {
    $size = Get-Size $path
    if ($size -gt 1048576) {
        $null = $items.Add(@{
            cat=$cat; app=$app; label=$label; path=$path
            size=$size; admin=$admin; proc=$proc
        })
    }
}

# A. System (universal on all Windows machines - use $sd/$windir, never hardcode C:)
Add-Entry "system" "System" "User-Temp"          $temp
Add-Entry "system" "System" "Win-Temp"           "$windir\Temp"                                $true
Add-Entry "system" "System" "CBS-Logs"           "$windir\Logs\CBS"                            $true
Add-Entry "system" "System" "WU-Download"        "$sd\Windows\SoftwareDistribution\Download"   $true
Add-Entry "system" "System" "Thumbnails"         "$local\Microsoft\Windows\Explorer"
Add-Entry "system" "System" "INetCache"          "$local\Microsoft\Windows\INetCache"
Add-Entry "system" "System" "WER"                "$prog\Microsoft\Windows\WER"                 $true
Add-Entry "system" "System" "Prefetch"           "$windir\Prefetch"                            $true
Add-Entry "system" "System" "Minidump"           "$windir\Minidump"                            $true

# B. Browsers (dynamic detection)
$browsers = @(
    @{n="Chrome";    base="$local\Google\Chrome\User Data\Default";                proc="chrome"}
    @{n="Edge";      base="$local\Microsoft\Edge\User Data\Default";               proc="msedge"}
    @{n="Brave";     base="$local\BraveSoftware\Brave-Browser\User Data\Default";  proc="brave"}
    @{n="Opera";     base="$roaming\Opera Software\Opera Stable";                  proc="opera"}
    @{n="Vivaldi";   base="$local\Vivaldi\User Data\Default";                      proc="vivaldi"}
    @{n="Arc";       base="$local\Arc\User Data\Default";                          proc="arc"}
    @{n="360";       base="$local\360Chrome\Chrome\User Data\Default";             proc="360chrome"}
    @{n="QQBrowser"; base="$local\Tencent\QQBrowser\User Data\Default";            proc="QQBrowser"}
)
foreach ($b in $browsers) {
    Add-Entry "browser" $b.n "$($b.n)-Cache"         "$($b.base)\Cache"          $false $b.proc
    Add-Entry "browser" $b.n "$($b.n)-ServiceWorker" "$($b.base)\Service Worker" $false $b.proc
    Add-Entry "browser" $b.n "$($b.n)-CodeCache"     "$($b.base)\Code Cache"     $false $b.proc
}
if (Test-Path "$roaming\Mozilla\Firefox\Profiles") {
    Get-ChildItem "$roaming\Mozilla\Firefox\Profiles" -Directory -EA SilentlyContinue | ForEach-Object {
        Add-Entry "browser" "Firefox" "Firefox-Cache-$($_.Name)" "$($_.FullName)\cache2" $false "firefox"
    }
}

# C. IM apps (dynamic detection - covers CN and international apps)
$imApps = @(
    @{n="WeChat";    proc="WeChat";      paths=@("$roaming\Tencent\WeChat\XPlugin","$roaming\Tencent\WeChat\radium","$roaming\Tencent\WeChat\log")}
    @{n="WeChat-New";proc="WeChatAppEx"; paths=@("$roaming\Tencent\xwechat\XPlugin","$roaming\Tencent\xwechat\radium")}
    @{n="WXWork";    proc="WXWork";      paths=@("$roaming\Tencent\WXWork\cef","$roaming\Tencent\WXWork\wmpf_Applet","$roaming\Tencent\WXWork\patch","$roaming\Tencent\WXWork\WxWorkDocConvert")}
    @{n="WeMeet";    proc="WeMeet";      paths=@("$roaming\Tencent\WeMeet\Cache")}
    @{n="QQ";        proc="QQ";          paths=@("$roaming\Tencent\QQ\Misc","$local\Tencent\QQ\Cache")}
    @{n="Lark";      proc="LarkShell";   paths=@("$roaming\LarkShell\aha\users","$roaming\LarkShell\update\update_downloading","$roaming\LarkShell\CodeCache","$roaming\LarkShell\GrShaderCache")}
    @{n="DingTalk";  proc="DingTalk";    paths=@("$roaming\DingTalk\Cache","$roaming\DingTalk\CodeCache")}
    @{n="Teams";     proc="Teams";       paths=@("$roaming\Microsoft\Teams\Cache","$roaming\Microsoft\Teams\Service Worker")}
    @{n="Slack";     proc="slack";       paths=@("$roaming\Slack\Cache","$roaming\Slack\Service Worker")}
    @{n="Discord";   proc="Discord";     paths=@("$roaming\discord\Cache","$roaming\discord\Code Cache")}
    @{n="Zoom";      proc="Zoom";        paths=@("$roaming\Zoom\data\Cache")}
    @{n="Skype";     proc="Skype";       paths=@("$roaming\Microsoft\Skype for Desktop\Cache")}
    @{n="Telegram";  proc="Telegram";    paths=@("$roaming\Telegram Desktop\tdata\user_data\cache")}
)
foreach ($app in $imApps) {
    foreach ($p in $app.paths) {
        Add-Entry "im" $app.n "$($app.n)-$(Split-Path $p -Leaf)" $p $false $app.proc
    }
}

# D. Other apps (cloud storage, media, etc.)
$otherApps = @(
    @{n="BaiduDisk"; proc="";                     paths=@("$roaming\baidunetdisk\Cache","$roaming\baidunetdisk\Code Cache","$roaming\baidu\BaiduYunKernel")}
    @{n="OneDrive";  proc="OneDrive";             paths=@("$local\Microsoft\OneDrive\logs","$local\Microsoft\OneDrive\StandaloneUpdater")}
    @{n="Dropbox";   proc="Dropbox";              paths=@("$roaming\Dropbox\logs")}
    @{n="Nutstore";  proc="NutstoreClient";        paths=@("$roaming\Nutstore\cache")}
    @{n="Spotify";   proc="Spotify";              paths=@("$roaming\Spotify\Storage")}
    @{n="Steam";     proc="Steam";                paths=@("$local\Steam\htmlcache")}
    @{n="Epic";      proc="EpicGamesLauncher";    paths=@("$local\EpicGamesLauncher\Saved\webcache")}
)
foreach ($app in $otherApps) {
    foreach ($p in $app.paths) {
        Add-Entry "app" $app.n "$($app.n)-$(Split-Path $p -Leaf)" $p $false $app.proc
    }
}

# E. Dev tools (dynamic detection)
$devItems = @(
    @{n="npm";        path="$local\npm-cache"}
    @{n="pip";        path="$local\pip\Cache"}
    @{n="Yarn";       path="$local\Yarn\Cache"}
    @{n="pnpm";       path="$local\pnpm\store"}
    @{n="Gradle";     path="$user\.gradle\caches"}
    @{n="Maven";      path="$user\.m2\repository"}
    @{n="VSCode";     path="$roaming\Code\Cache"}
    @{n="VSCode-log"; path="$roaming\Code\logs"}
    @{n="Cursor";     path="$roaming\Cursor\Cache"}
    @{n="JetBrains";  path="$local\JetBrains"}
    @{n="Docker";     path="$local\Docker\wsl\data"}
    @{n="Azure";      path="$local\AzureFunctionsTools"}
    @{n="AndroidSDK"; path="$local\Android\Sdk\.temp"}
    @{n="NVIDIA-GL";  path="$local\NVIDIA\GLCache"}
    @{n="NVIDIA-DX";  path="$local\NVIDIA\DXCache"}
    @{n="D3DCache";   path="$local\D3DSCache"}
)
foreach ($t in $devItems) {
    Add-Entry "dev" $t.n "$($t.n)-cache" $t.path
}

# F. Optional items (not selected by default)
$optional = [System.Collections.Generic.List[hashtable]]::new()

if (Test-Path "$sd\hiberfil.sys") {
    $null = $optional.Add(@{label="hiberfil.sys"; size=(Get-Item "$sd\hiberfil.sys" -Force).Length; admin=$true; note="powercfg /hibernate off"})
}

@(
    @{p="$prog\Lenovo\UserGuide";        l="Lenovo-UserGuide"}
    @{p="$prog\Lenovo\LDF";              l="Lenovo-LDF"}
    @{p="$prog\Lenovo\LVAPRCU";          l="Lenovo-LVAPRCU"}
    @{p="$prog\Dell\UpdateService";      l="Dell-UpdateService"}
    @{p="$prog\HP\HP Support Framework"; l="HP-SupportFramework"}
    @{p="$prog\ASUS";                    l="ASUS-tools"}
    @{p="$prog\Acer";                    l="Acer-tools"}
) | ForEach-Object {
    if (Test-Path $_.p) {
        $sz = Get-Size $_.p
        if ($sz -gt 10MB) {
            $null = $optional.Add(@{label=$_.l; size=$sz; admin=$true; note="vendor-preinstall"})
        }
    }
}

$pkgSize = Get-Size "$prog\Package Cache"
if ($pkgSize -gt 100MB) {
    $null = $optional.Add(@{label="Package-Cache"; size=$pkgSize; admin=$true; note="repair-reinstall-risk"})
}

$null = $optional.Add(@{label="DISM-ResetBase"; size=0; admin=$true; note="15-40min-no-rollback"})

# Running process detection
$runningProcs = @((Get-Process -EA SilentlyContinue).Name)
$blockedItems = $items | Where-Object { $_.proc -and ($runningProcs -contains $_.proc) }
$runningApps  = ($blockedItems | ForEach-Object { $_.app } | Select-Object -Unique) -join ","

# Disk info
$drv     = Get-PSDrive C
$freeGB  = [math]::Round($drv.Free/1GB, 2)
$usedGB  = [math]::Round($drv.Used/1GB, 2)
$totalGB = [math]::Round(($drv.Used+$drv.Free)/1GB, 2)
$dlMB    = 0
if (Test-Path "$user\Downloads") {
    $dlMB = [math]::Round((Get-ChildItem "$user\Downloads" -Recurse -Force -EA SilentlyContinue |
        Where-Object{-not $_.PSIsContainer} | Measure-Object Length -Sum).Sum/1MB, 0)
}

@{
    items       = @($items)
    optional    = @($optional)
    running     = $runningApps
    free_gb     = $freeGB
    used_gb     = $usedGB
    total_gb    = $totalGB
    dl_mb       = $dlMB
} | ConvertTo-Json -Depth 5 | Out-File "$temp\dc_results.json" -Encoding UTF8

# Verify output; write error marker if missing
if (-not (Test-Path "$temp\dc_results.json")) {
    '{"error":"scan_failed","items":[],"optional":[]}' | Out-File "$temp\dc_results.json" -Encoding UTF8
}
```

运行：`powershell -ExecutionPolicy Bypass -File "$env:TEMP\dc_scan.ps1"`

**读取结果**：PowerShell 的 `Out-File -Encoding UTF8` 会写入含 BOM 的 UTF-8，必须用 Python 解析（禁止用 `powershell -Command "...ConvertFrom-Json..."` 内联读取，原因同上）：

```python
python -c "
import json
with open(r'C:/Users/<用户名>/AppData/Local/Temp/dc_results.json', encoding='utf-8-sig') as f:
    j = json.load(f)
# 访问 j['items'], j['free_gb'], j['running'] 等字段
"
```

> **注意**：路径中 `<用户名>` 需替换为实际用户名，或通过 `%TEMP%` 展开后的绝对路径。

> **编码铁则**：Python 内联脚本在 Windows Bash 下默认以 GBK 输出，`⚑` `✱` 等 Unicode 符号会触发 `UnicodeEncodeError`。
> 解决方案二选一：
> - 脚本开头加 `import sys, io; sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')`
> - **或者（推荐）**：Python 只输出纯 ASCII 数据（用 ADMIN/RUNNING 等英文标记），特殊符号由 LLM 在展示阶段自行拼接，不在 Python 里 print。

> **若 JSON 含 `"error":"scan_failed"`，立即停止并告知用户，不执行任何删除操作。**

---

## W-2：动态分类编号规则

**不预设任何固定列表。读取扫描 JSON 后，按以下规则动态构建清单：**

### 分类原则（大类固定，内容随机器变化）

| 大类 | 对应 `cat` 字段 | 说明 |
|------|-----------------|------|
| A    | system          | 系统缓存/日志/临时文件 |
| B    | browser         | 任何已安装浏览器的缓存 |
| C    | im              | 任何即时通讯/会议类应用 |
| D    | app             | 云存储、媒体、其他应用 |
| E    | dev             | 开发工具、编译器、包管理器 |
| F    | optional        | 可选大项（默认不选） |

### 编号规则
1. 将 `items` 数组按 `cat` 分组
2. 每组内按 `size` **从大到小**排列
3. 顺序编号：A1, A2, A3 ... B1, B2 ...（只给实际存在的项目编号）
4. `optional` 数组的项目编为 F1, F2 ...
5. **该电脑没有的软件 = 扫描结果 size=0 = 不出现在清单里**

### 标记规则
- `admin=true` → 标注 ⚑（需管理员权限）
- `proc` 字段对应的进程正在运行 → 标注 ✱（需关闭该应用）
- F 类 → 标注 ⚠（可选，默认不选）

---

## W-3：授权合并原则（关键）

**目标：全程用户只看到一次 UAC 弹窗，一次应用关闭提示。**

### 执行前合并所有需授权操作
- 展示清单时，同步列出：
  - "⚑ 需要管理员权限的编号：[列出]，确认后弹**一次** UAC"
  - "✱ 需要关闭的应用：[应用名]，关闭后自动继续"
- 将所有 ⚑ 项目的清理命令写入同一个 `dc_admin.ps1`，用**一次** `Start-Process powershell -Verb RunAs -Wait` 执行完毕
- 所有 ✱ 项目：等进程退出后**批量**清理（不逐个询问）

### 执行顺序
1. 先执行：不需要权限、应用未运行的所有项目
2. 同时发起：UAC 弹窗（一次，包含所有 ⚑ 项）
3. 等待：需关闭的应用进程消失，然后自动清理对应项
4. 后台启动：DISM（如用户选了 F 类中的 DISM 项）

---

## W-4：Windows 优化建议触发条件

| 检测条件 | 建议 |
|----------|------|
| 微信/QQ 文件路径在系统盘 | 建议在应用设置中将文件存储路径改到其他盘 |
| 下载目录 > 500MB | 建议将浏览器下载路径改到非系统盘，定期清理 |
| 休眠文件存在且未选 F 类 | powercfg /hibernate off 可释放 [X] GB |
| 任意开发工具缓存 > 1GB | 建议定期运行对应清理命令（npm/pip/gradle） |
| 多个浏览器均有大量缓存 | 建议只保留常用浏览器，其余卸载 |

---

---

# ══════════════════════════════════
# macOS 支线
# ══════════════════════════════════

## M-1：扫描脚本（动态发现）

写入并运行 `/tmp/dc_scan.sh`：

```bash
#!/bin/bash
# ENCODING RULE: ASCII ONLY in this script. No Chinese/CJK anywhere.
# PORTABILITY: No declare -A (requires bash 4+, macOS ships bash 3.2).
#              No python3 dependency. Pure bash JSON building.
#              Works on bash 3.x, bash 5.x, and zsh.

set -o pipefail
H="$HOME"
OUT="/tmp/dc_results.json"

# Size in bytes; returns 0 if path does not exist
get_size() {
    [ ! -e "$1" ] && echo 0 && return
    du -sk "$1" 2>/dev/null | awk '{printf "%d", $1 * 1024}' || echo 0
}

# Pure-bash JSON item builder - no external tools required
ITEMS_JSON=""
add_entry() {
    local cat="$1" app="$2" label="$3" path="$4"
    local admin="${5:-false}" proc="${6:-}"
    local size
    size=$(get_size "$path")
    [ -z "$size" ] && return
    [ "$size" -lt 1048576 ] && return
    # Escape double-quotes and backslashes in path for JSON safety
    local spath
    spath=$(printf '%s' "$path" | sed 's/\\/\\\\/g; s/"/\\"/g')
    local entry
    entry="{\"cat\":\"$cat\",\"app\":\"$app\",\"label\":\"$label\",\"path\":\"$spath\",\"size\":$size,\"admin\":$admin,\"proc\":\"$proc\"}"
    if [ -z "$ITEMS_JSON" ]; then ITEMS_JSON="$entry"
    else ITEMS_JSON="$ITEMS_JSON,$entry"
    fi
}

OPTIONAL_JSON=""
add_optional() {
    local label="$1" size="$2" admin="${3:-false}" note="$4"
    local entry
    entry="{\"label\":\"$label\",\"size\":$size,\"admin\":$admin,\"note\":\"$note\"}"
    if [ -z "$OPTIONAL_JSON" ]; then OPTIONAL_JSON="$entry"
    else OPTIONAL_JSON="$OPTIONAL_JSON,$entry"
    fi
}

# A. System caches
add_entry "system" "System" "User-AppCaches"    "$H/Library/Caches"
add_entry "system" "System" "User-Logs"         "$H/Library/Logs"
add_entry "system" "System" "Trash"             "$H/.Trash"
add_entry "system" "System" "DiagnosticReports" "/Library/Logs/DiagnosticReports" true

# B. Browsers - sequential checks, no associative arrays
AS="$H/Library/Application Support"
add_entry "browser" "Chrome"  "Chrome-Cache"          "$AS/Google/Chrome/Default/Cache"           false "chrome"
add_entry "browser" "Chrome"  "Chrome-ServiceWorker"  "$AS/Google/Chrome/Default/Service Worker"  false "chrome"
add_entry "browser" "Chrome"  "Chrome-CodeCache"      "$AS/Google/Chrome/Default/Code Cache"      false "chrome"
add_entry "browser" "Edge"    "Edge-Cache"             "$AS/Microsoft Edge/Default/Cache"          false "msedge"
add_entry "browser" "Edge"    "Edge-ServiceWorker"     "$AS/Microsoft Edge/Default/Service Worker" false "msedge"
add_entry "browser" "Brave"   "Brave-Cache"            "$AS/BraveSoftware/Brave-Browser/Default/Cache"          false "brave"
add_entry "browser" "Brave"   "Brave-ServiceWorker"    "$AS/BraveSoftware/Brave-Browser/Default/Service Worker" false "brave"
add_entry "browser" "Opera"   "Opera-Cache"            "$AS/com.operasoftware.Opera/Cache"         false "opera"
add_entry "browser" "Arc"     "Arc-Cache"              "$AS/Arc/User Data/Default/Cache"           false "arc"
add_entry "browser" "Safari"  "Safari-Cache"           "$H/Library/Caches/com.apple.Safari"        false "Safari"
add_entry "browser" "Safari"  "Safari-LocalStorage"    "$H/Library/Safari/LocalStorage"            false "Safari"
# Firefox: multiple profiles, loop without associative arrays
if [ -d "$AS/Firefox/Profiles" ]; then
    for profile_dir in "$AS/Firefox/Profiles"/*/; do
        [ -d "${profile_dir}cache2" ] && add_entry "browser" "Firefox" "Firefox-Cache" "${profile_dir}cache2" false "firefox"
    done
fi

# C. IM apps - sequential checks, no associative arrays
# WeChat (sandbox container)
WX="$H/Library/Containers/com.tencent.xinWeChat/Data"
add_entry "im" "WeChat"  "WeChat-Caches"  "$WX/Library/Caches"  false "WeChat"
add_entry "im" "WeChat"  "WeChat-Logs"    "$WX/Library/Logs"    false "WeChat"
# WXWork
WW="$H/Library/Containers/com.tencent.WXWork/Data"
add_entry "im" "WXWork"  "WXWork-Caches"  "$WW/Library/Caches"  false "WXWork"
# WeMeet
WM="$H/Library/Containers/com.tencent.meeting/Data"
add_entry "im" "WeMeet"  "WeMeet-Caches"  "$WM/Library/Caches"  false "WeMeet"
# QQ
QQ="$H/Library/Containers/com.tencent.qq/Data"
add_entry "im" "QQ"      "QQ-Caches"      "$QQ/Library/Caches"  false "QQ"
# Lark/Feishu
add_entry "im" "Lark"    "Lark-AppSupport" "$AS/Lark"            false "Lark"
add_entry "im" "Lark"    "Lark-Caches"     "$H/Library/Caches/Lark" false "Lark"
# DingTalk
add_entry "im" "DingTalk" "DingTalk-Cache" "$AS/DingTalk/Cache"  false "DingTalk"
# International IM
add_entry "im" "Slack"    "Slack-Cache"    "$AS/Slack/Cache"            false "slack"
add_entry "im" "Discord"  "Discord-Cache"  "$AS/Discord/Cache"          false "Discord"
add_entry "im" "Zoom"     "Zoom-Cache"     "$AS/zoom.us/Cache"          false "zoom.us"
add_entry "im" "Teams"    "Teams-Cache"    "$AS/Microsoft Teams/Cache"  false "Teams"
add_entry "im" "Telegram" "Telegram-Cache" "$AS/Telegram Desktop/Cache" false "Telegram"

# D. Other apps
BAIDU="$H/Library/Containers/com.baidu.netdisk/Data"
add_entry "app" "BaiduDisk" "BaiduDisk-Caches"  "$BAIDU/Library/Caches"           false
add_entry "app" "Spotify"   "Spotify-Cache"      "$AS/Spotify/PersistentCache"     false "Spotify"
add_entry "app" "OneDrive"  "OneDrive-Logs"      "$AS/OneDrive/logs"               false "OneDrive"

# E. Dev tools
add_entry "dev" "npm"       "npm-cacache"    "$H/.npm/_cacache"
add_entry "dev" "pip"       "pip-cache"      "$H/Library/Caches/pip"
add_entry "dev" "Yarn"      "Yarn-cache"     "$H/Library/Caches/Yarn"
add_entry "dev" "pnpm"      "pnpm-store"     "$H/Library/pnpm/store"
add_entry "dev" "Homebrew"  "Homebrew-cache" "$H/Library/Caches/Homebrew"
add_entry "dev" "Gradle"    "Gradle-caches"  "$H/.gradle/caches"
add_entry "dev" "Maven"     "Maven-repo"     "$H/.m2/repository"
add_entry "dev" "VSCode"    "VSCode-cache"   "$AS/Code/Cache"
add_entry "dev" "VSCode"    "VSCode-logs"    "$AS/Code/logs"
add_entry "dev" "Cursor"    "Cursor-cache"   "$AS/Cursor/Cache"
add_entry "dev" "JetBrains" "JetBrains-cache" "$H/Library/Caches/JetBrains"
add_entry "dev" "Xcode"     "Xcode-DerivedData" "$H/Library/Developer/Xcode/DerivedData"
add_entry "dev" "Xcode"     "Xcode-SimCaches"   "$H/Library/Developer/CoreSimulator/Caches"
add_entry "dev" "Docker"    "Docker-vms"     "$H/Library/Containers/com.docker.docker/Data/vms"

# F. Optional items
ios_bk=$(get_size "$H/Library/Application Support/MobileSync/Backup")
[ "$ios_bk" -gt 1048576 ] && add_optional "iOS-LocalBackup" "$ios_bk" false "confirm-icloud-backup-first"

xcode_arch=$(get_size "$H/Library/Developer/Xcode/Archives")
[ "$xcode_arch" -gt 1048576 ] && add_optional "Xcode-Archives" "$xcode_arch" false "old-app-archives"

xcode_sdk=$(get_size "$H/Library/Developer/Xcode/iOS DeviceSupport")
[ "$xcode_sdk" -gt 1048576 ] && add_optional "Xcode-iOSDeviceSupport" "$xcode_sdk" false "keep-latest-2-versions"

# Running app detection
running=""
pgrep -x "WeChat"    > /dev/null 2>&1 && running="${running}WeChat,"
pgrep -x "WXWork"    > /dev/null 2>&1 && running="${running}WXWork,"
pgrep -x "Lark"      > /dev/null 2>&1 && running="${running}Lark,"
pgrep -x "DingTalk"  > /dev/null 2>&1 && running="${running}DingTalk,"
pgrep -x "Zoom"      > /dev/null 2>&1 && running="${running}Zoom,"
running="${running%,}"

# Disk info
disk_info=$(df -g / 2>/dev/null | tail -1)
total_gb=$(echo "$disk_info" | awk '{print $2}')
used_gb=$(echo  "$disk_info" | awk '{print $3}')
free_gb=$(echo  "$disk_info" | awk '{print $4}')
dl_mb=$(du -sm "$H/Downloads" 2>/dev/null | awk '{print $1}')
dl_mb="${dl_mb:-0}"

# Write final JSON - pure bash, no python3 required
cat > "$OUT" << JSONEOF
{
  "items": [${ITEMS_JSON}],
  "optional": [${OPTIONAL_JSON}],
  "running": "${running}",
  "free_gb": ${free_gb:-0},
  "used_gb": ${used_gb:-0},
  "total_gb": ${total_gb:-0},
  "dl_mb": ${dl_mb}
}
JSONEOF

# Verify output was created
if [ ! -s "$OUT" ]; then
    echo '{"error":"scan_failed","items":[],"optional":[]}' > "$OUT"
fi
```

运行：`bash /tmp/dc_scan.sh`，读取 `/tmp/dc_results.json`。

---

## M-2：动态分类编号规则

与 Windows W-2 **完全相同的规则**：
- 读取 JSON，按 `cat` 字段分组 → A/B/C/D/E/F
- 每组内按 `size` 降序编号
- 只列出 > 1MB 的项目
- `admin=true` → ⚑（需 sudo/osascript）
- 进程运行中 → ✱（需关闭应用）

## M-3：macOS 授权合并

- 所有 ⚑ 项合并为**一条** `osascript -e 'do shell script "cmd1 && cmd2 && ..." with administrator privileges'`
  → 用户只看到一次系统密码框
- Time Machine 快照：`tmutil deletelocalsnapshots /`（含在上述 osascript 中）

## M-4：macOS 优化建议触发条件

| 检测条件 | 建议 |
|----------|------|
| Homebrew缓存 > 500MB | `brew cleanup --prune=all` |
| Xcode DerivedData > 2GB | Xcode → Product → Clean Build Folder |
| iOS备份 > 1GB 且有iCloud | 确认云端已备份后删除本地备份 |
| npm/pip/gradle > 1GB | 对应清理命令（见各工具文档） |

---

---

# ══════════════════════════════════
# 通用流程（Windows 和 macOS 共用）
# ══════════════════════════════════

## 第三步：向用户呈现清单

读取 JSON，动态构建以下格式的清单（仅显示 > 1MB 的项目）：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 扫描完成 | C盘剩余 XX GB / 共 XX GB
   可清理（默认选中）：约 X.X GB
   [若有运行中应用] ⚠ [应用名] 正在运行，对应项目关闭后自动继续
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🟢 默认全部清理（输入编号可排除）

━━━ A. 系统缓存 ━━━
  A1. [应用·标签]   XXX MB   [一句话说明]   ⚑需管理员
  A2. [应用·标签]   XXX MB   [说明]
  ...

━━━ B. 浏览器缓存 ━━━
  B1. [浏览器名·类型]  XXX MB   书签/密码/历史完全不受影响
  ...

[其他大类同格式]

🟡 可选项（默认不选，输入 +F1 添加）

━━━ F. 可选 ━━━
  F1. [名称]   X.X GB   [说明]   ⚑需管理员/sudo
  ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ 绝对不会清理：聊天记录、收到的文件、书签密码、桌面/文档

[若有以下情况则显示]
⚠️  发现需关注（不自动处理）：
  · 下载目录有 X MB 文件

━━━ 授权汇总（确认后一次性处理）━━━
  ⚑ 需管理员权限：[列出编号]  → Windows 弹一次 UAC / macOS 弹一次密码框
  ✱ 需关闭的应用：[应用名(对应编号)]  → 关闭后自动继续，无需再操作
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 第四步：单次用户确认（全程唯一一次交互）

**必须使用 `AskUserQuestion` 工具**弹出交互式选择框，不要输出纯文字让用户自己回复。

**同时提两个问题**（AskUserQuestion 支持 1-4 个问题，合并在一次调用里）：

**问题一：清理方案**
- header: `"清理方案"`
- question: `"默认清理上方所有 🟢 项目（约 X GB）。如需调整请选「自定义」，在 Other 输入框说明：排除某项→编号如C1 C2 / 添加可选→+F1 / 混合→C1 +F1"`
- options:
  - `{ label: "全部执行（推荐）", description: "清理所有 A-E 类，约 X GB，需关闭 [应用名]，弹一次系统 UAC" }`
  - `{ label: "自定义", description: "在 Other 输入框输入要排除或添加的编号" }`

**问题二：工具执行权限**
- header: `"执行权限"`
- question: `"清理过程需多次调用系统命令。选「自动批准」可全程无打扰；选「逐条确认」每条命令都会弹窗询问。"`
- options:
  - `{ label: "自动批准（推荐）", description: "Claude Code 自动放行所有清理命令，只有 Windows UAC 会弹一次系统窗口" }`
  - `{ label: "逐条确认", description: "每条 Bash/PowerShell 命令都会弹权限确认框，过程会被多次打断" }`

> 用户选「自动批准」后，提示其在 Claude Code 中切换到 Auto-approve 模式（快捷键或设置），然后再继续执行。
> 用户选「逐条确认」则直接继续，告知过程中会多次弹窗属正常现象。

### 输入解析规则
- `全部执行` 或留空 → 清理全部 A-E 类，⚑ 项**一次性授权**
- `C1 C2` → 排除这些，其余执行
- `+F1` → 添加可选项，绿色项照常执行
- `-A3` → 跳过对 A3 的授权，其余授权项照常
- **收到确认后立即执行，不再打扰用户**

---

## 第五步：执行清理

> ⚠️ **清理脚本写入铁则（与 W-1 扫描脚本完全相同的规则）**：
> 清理过程需要写入 `dc_clean_normal.ps1`、`dc_clean_admin.ps1`、`dc_clean_wait.ps1` 等脚本。
> - **正确做法**：用 **Write 工具**直接写入，完全绕开 bash 解析冲突
>   - ⚠️ **Write 工具限制**：写入前必须先用 **Read 工具**尝试读取目标路径（无论文件是否存在），否则文件已存在时会报"File has not been read yet"错误。
> - **严禁 `python -c "...PS脚本内容..."`**：bash 双引号会将 `$local`、`$env`、`$roaming` 等展开为空字符串，写入文件后 PowerShell 变量全部丢失
> - **严禁 bash heredoc**：单引号截断问题同 W-1
> - 读取清理结果 JSON 同样用 Python + `encoding='utf-8-sig'`

### 执行顺序
1. **先执行**：无需权限、应用未运行的所有项目
2. **同步发起**：将所有选中 ⚑ 项写入同一脚本，**一次** UAC / osascript 执行完
3. **自动等待**：检测需关闭的应用，进程消失后**批量**清理（不逐个询问）
4. **后台启动**：DISM / Time Machine 等耗时项，不阻塞其他操作

### 每项完成后立即输出
```
  ✓ [编号] [应用·项目名]    释放 X MB
```

---

## 第六步：汇报结果

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ 清理完成
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
释放空间：X.X GB
磁盘变化：XX GB → XX GB（+X.X GB）

清理明细：
  ✓ [编号] [名称]   X MB
  ...
[若有跳过] ○ 已跳过：[编号]（用户排除）
[若有后台] ⏳ [任务]：后台进行中，预计再释放约 X GB
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 第七步：针对性优化建议

只根据本次扫描实际发现的问题输出对应系统（W-4 或 M-4）的建议，不说废话。

---

## 第八步：更新记忆

清理完成后写入 `~/.claude/projects/.../memory/MEMORY.md`：
- 清理时间、系统类型、释放量、清理前后剩余
- 用户本次排除的项目（下次默认也排除）
- 哪类缓存增长最快（下次重点提示）

下次触发时先读取，告知：
> "上次清理：[日期]，释放 X GB。距今 N 天，增长最快：[类别]"

---

## 执行原则

1. **触发即扫描**，不问前置问题
2. **只打扰用户一次**：扫描→展示→单次确认→全自动执行→汇报
3. **授权严格合并**：Windows 一次 UAC，macOS 一次密码框，全程不重复弹窗
4. **绝不动用户数据**：聊天记录、收到的文件、书签密码、桌面/文档一律不碰
5. **清单内容完全动态**：由扫描结果决定，不预设软件列表，不同电脑自动适配
6. **真实数字**：清理前后实际读磁盘，不写估算值
7. **优化建议有针对性**：只说扫描中实际发现的问题
