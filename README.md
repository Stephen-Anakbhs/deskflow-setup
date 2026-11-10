# Deskflow：Windows 主机 + Mac 从机配置

目标：用 Windows 电脑上的键盘和鼠标同时控制 Mac。Windows 是**服务端（主机）**，Mac 是**客户端（从机）**。

> **给在 Mac 上执行配置的 AI 助手**：主机已经配好，你只需要完成下面的「Mac 从机配置」部分。
> 请按顺序做，每做完一步就核对它的「验证」项。遇到需要用户点击授权的地方（系统设置里的权限开关），停下来请用户操作。

---

## 1. 主机现状（Windows，已完成，无需改动）

| 项目 | 值 |
|---|---|
| Deskflow 版本 | **1.27.0** |
| 角色 | 服务端（Server） |
| 主机计算机名 | `WINBOX` |
| 主机局域网 IP | **`192.168.110.174`**（Wi‑Fi 网卡；若连不上先确认这个 IP 是否变化，见第 4 节） |
| 端口 | `24800`（TCP，防火墙已放行） |
| TLS 加密 | 开启 |
| 指纹校验 | 关闭（`checkPeerFingerprints=false`） |
| 剪贴板共享 | 开启 |
| 期望的 Mac 计算机名 | **`MACBOOK`** |
| Mac 摆放位置 | Windows 屏幕的**上方** |

主机的配置文件副本在 [`windows-server/Deskflow.conf`](windows-server/Deskflow.conf)，仅供参考。

---

## 2. Mac 从机配置

### 2.1 安装 Deskflow 1.27.0

**两端版本必须一致**，Mac 上也要装 **1.27.0**（如果已经装了旧版，直接覆盖升级）。

方式 A（推荐，若 Mac 装有 Homebrew）：

```bash
brew install --cask deskflow   # 已安装旧版时改用：brew upgrade --cask deskflow
```

方式 B：从官方发布页下载 macOS 安装包，按芯片选择 `arm64`（Apple 芯片）或 `x86_64`（Intel）：
<https://github.com/deskflow/deskflow/releases/tag/v1.27.0>

查看芯片类型：

```bash
uname -m   # arm64 = Apple 芯片；x86_64 = Intel
```

把 `Deskflow.app` 拖进「应用程序」文件夹。若首次打开提示“无法验证开发者”，到「系统设置 → 隐私与安全性」底部点「仍要打开」。

**验证**：

```bash
defaults read /Applications/Deskflow.app/Contents/Info.plist CFBundleShortVersionString
# 应输出 1.27.0（或以 1.27 开头）
```

### 2.2 确认 Mac 能连到主机

Mac 和 Windows 必须在**同一个局域网**（同一个路由器 / Wi‑Fi）。

```bash
nc -vz 192.168.110.174 24800
```

**验证**：输出含 `succeeded` / `open`。

如果失败：
- 先 `ping 192.168.110.174`，不通说明不在同一网络（或网络开启了客户端隔离，比如校园网/访客 Wi‑Fi）。可以把两台机器连到同一个手机热点，或用网线直连，然后请用户在 Windows 上重新查看 IP（见第 4 节）。
- Mac 上若开着代理 / VPN 的 TUN（全局接管）模式，可能拦截局域网流量，先关掉再试。

### 2.3 写入客户端配置

先**完全退出** Deskflow（菜单栏图标 → Quit；或执行 `pkill -x Deskflow; pkill -f deskflow-core`），否则它退出时会覆盖你写的文件。

配置文件位置（Deskflow 1.27 在 macOS 上的搜索路径）：`~/Library/Deskflow/Deskflow.conf`

```bash
pkill -x Deskflow 2>/dev/null; pkill -f deskflow-core 2>/dev/null
mkdir -p ~/Library/Deskflow
[ -f ~/Library/Deskflow/Deskflow.conf ] && cp ~/Library/Deskflow/Deskflow.conf ~/Library/Deskflow/Deskflow.conf.bak
cat > ~/Library/Deskflow/Deskflow.conf <<'EOF'
[core]
coreMode=1
computerName=MACBOOK
port=24800

[client]
remoteHost=192.168.110.174

[gui]
startCoreWithGui=true
closeToTray=true

[security]
tlsEnabled=true
checkPeerFingerprints=false
EOF
```

说明：
- `coreMode=1` = 客户端模式。
- `computerName=MACBOOK` **必须**和主机布局里的名字完全一致（区分大小写）。
- `tlsEnabled=true` 必须和主机一致，否则握手失败。
- 文件里没写到的项，Deskflow 会自动补成默认值，这是正常的。

如果 Deskflow 启动后界面里显示的不是上面这些值（说明这台 Mac 上它从别的位置读取设置），就直接在图形界面里设置，效果相同：
1. 选「Use another computer's mouse and keyboard (client)」／「使用另一台计算机的鼠标和键盘」；
2. 服务器地址填 `192.168.110.174`；
3. 菜单 Edit → Preferences（或 Deskflow → Settings）里，把 Computer name 改为 `MACBOOK`，勾选 TLS / Enable TLS encryption，端口 `24800`；
4. 点 Apply / Start。

### 2.4 授予 macOS 权限（需要用户亲手点击）

打开「系统设置 → 隐私与安全性」，在以下两个列表里打开 **Deskflow** 的开关（列表里没有就点 `+` 从「应用程序」中添加）：

1. **辅助功能（Accessibility）**——没有这项，鼠标能移过来，但点击和键盘都不起作用。
2. **输入监控（Input Monitoring）**。

可以用下面的命令直接打开对应页面：

```bash
open "x-apple.systempreferences:com.apple.preference.security?Privacy_Accessibility"
open "x-apple.systempreferences:com.apple.preference.security?Privacy_ListenEvent"
```

改完权限后，**重新启动 Deskflow**（权限只对新启动的进程生效）。若升级过版本，旧的授权可能失效：先在列表里把 Deskflow 删掉（`-`），再重新添加。

### 2.5 启动并连接

```bash
open -a Deskflow
```

界面里确认是客户端模式、服务器地址为 `192.168.110.174`，点 **Start**。

**验证**：Deskflow 底部状态显示 *Connected*（已连接）。

### 2.6 第一次连接：在 Windows 上把 Mac 加进布局（需要用户在 Windows 上操作）

主机的新版布局里**目前只有 WINBOX 自己**，Mac 第一次连上来时：
- Windows 上的 Deskflow 会弹窗询问是否添加新电脑 `MACBOOK`，点同意；
- 如果没有弹窗，或 Mac 日志显示被拒绝（如 `unknown client` / `unrecognised client name`）：在 Windows 上打开托盘里的 Deskflow → **配置服务器(C)** → 把右上角的电脑图标拖到 `WINBOX` **正上方**的格子 → 双击它，名字改为 `MACBOOK` → 确定 → 点**应用/重启**。

然后在 Mac 上再点一次 Start（或等它自动重连）。

**最终验证**：在 Windows 上把鼠标移出屏幕**顶边**，指针出现在 Mac 上；在 Mac 上能点击、能打字；在一边复制文字，另一边能粘贴。

---

## 3. 推荐的 Mac 端调整（可选）

- **修饰键互换**：Windows 键盘的 Ctrl 默认对应 Mac 的 Control。想在 Mac 上也按 Ctrl+C 复制，需要在 **Windows 主机**的「配置服务器」里双击 `MACBOOK`，把修饰键映射改为 `Ctrl → Meta(Command)`、`Meta(Win) → Ctrl`。这一项在主机端设置。
- **开机自动连接**：「系统设置 → 通用 → 登录项」里添加 Deskflow。注意：Mac 锁屏/刚开机时 Deskflow 还没运行，需要用 Mac 自带的键盘或触控板输一次密码。
- **滚轮方向**：觉得 Mac 上滚轮方向反了，可在 `[client]` 里加 `invertYScroll=true`，或在 Mac 的鼠标设置里切换“自然滚动”。

---

## 3.5 已连上但鼠标在 Mac 上不流畅（卡顿 / 一顿一顿）

**主机端已排查（2026-11-10）**：Windows → Mac（192.168.110.193）ping 稳定 2–5 ms，0 丢包；屏幕切换日志正常。所以网络不是主因，按下面顺序处理 Mac 端：

1. **关闭 Deskflow 的 App Nap（最常见原因）**。Deskflow 窗口不在前台时，macOS 会给它节流，鼠标就会一顿一顿：

   ```bash
   BID="$(osascript -e 'id of app "Deskflow"')"; echo "$BID"
   defaults write "$BID" NSAppSleepDisabled -bool YES
   pkill -x Deskflow; pkill -f deskflow-core; sleep 1; open -a Deskflow
   ```

   验证：`defaults read "$BID" NSAppSleepDisabled` 输出 `1`。

2. **日志级别不要用 DEBUG**。DEBUG 会把每次鼠标移动都写进日志，非常拖慢速度。确认 `~/Library/Deskflow/Deskflow.conf` 里没有 `level=DEBUG`（用 `level=INFO` 或直接删掉这一行），改完后重启 Deskflow。

3. **反向测一下网络抖动**（Mac 的 Wi‑Fi 省电偶尔会造成卡顿尖峰）：

   ```bash
   ping -c 60 -i 0.2 192.168.110.174 | tail -3
   ```

   如果 max 远大于 avg（比如超过 50 ms），让 Mac 连 5 GHz Wi‑Fi 或改用有线网。

4. **屏幕缩放**：Mac 外接 21:9 显示器时，如果在「显示器」设置里选了非默认的缩放分辨率，指针移动的计算量会变大。可以试试切回“默认”分辨率，对比一下流畅度。

5. **鼠标回报率（在 Windows 那边处理，需要用户操作）**：如果鼠标是 4000/8000 Hz 的高回报率游戏鼠标，Deskflow 每秒要转发几千个移动事件，Mac 端会跟不上。请用户在鼠标驱动（如 Logi Options+ / G HUB）里把回报率降到 **1000 Hz 或 500 Hz** 再试。

每处理一项，都请用户实际移动鼠标对比一下，记录是哪一项起了作用。

---

## 4. 排错

| 现象 | 处理 |
|---|---|
| 连不上 / `Timed out` / `Connection refused` | `nc -vz 192.168.110.174 24800`；不通就检查网络（第 2.2 节）。请用户在 Windows PowerShell 里运行 `Get-NetIPAddress -AddressFamily IPv4` 确认 IP 有没有变 |
| `unknown client name` / 连上后立刻断开 | Mac 的计算机名不是 `MACBOOK`，或主机布局里还没加 Mac（第 2.6 节） |
| TLS / SSL 握手错误 | 两边 TLS 开关不一致，或版本不一致；Mac 上确认 `tlsEnabled=true`，版本 1.27.0 |
| 鼠标能过去但不能点击 / 不能打字 | 辅助功能、输入监控权限没开，或开了之后没重启 Deskflow（第 2.4 节） |
| 连上了但不流畅 | 见第 3.5 节（App Nap、日志级别、Wi‑Fi、回报率） |

Mac 端日志：Deskflow 界面下方的日志面板。也可以在 Preferences 里打开“记录日志到文件”，把文件内容反馈给用户。

---

## 附：主机端是怎么配置的（供参考，Mac 无需执行）

1. `winget install -e --id Deskflow.Deskflow` 安装 1.27.0（Windows 服务 `Deskflow` 开机自动运行）。
2. 设置文件为 `C:\ProgramData\Deskflow\Deskflow.conf`。1.27 不再读取旧版外部布局文件里的 `section: screens`，所以改用图形界面内置的布局（`externalConfig=false`）。
3. 修正了系统迁移后 `C:\ProgramData\Deskflow` 目录的写权限。
4. 在“启动”文件夹添加了 Deskflow 快捷方式，登录后自动在托盘运行。
5. Windows 防火墙已有 `Deskflow Server` 入站放行规则（`deskflow-core.exe`，所有网络类型）。
