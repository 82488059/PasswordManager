# 04 Windows 桌面端设计

> 本文覆盖 Windows 桌面端：**Qt 6 / QML 界面 + C++ 适配层（pm-qt）+ Rust 原生层（pm-core / pm-win）**的分工、前台识别与匹配、自动输入（Auto-Type）、密码框检测、**手动输入后保存/更新凭据**、托盘与锁定，以及与浏览器扩展的协作（ExtensionBridge / NativeHost）。浏览器网页内的填充与保存由扩展完成，见 05b。总体技术路线与许可证见 02 §5–§7。

## 0. 分层与工程结构
```
app/                                  # Qt 6 应用（Windows 与 Android 共用 QML + C++）
├─ CMakeLists.txt                     # qt_add_executable / qt_add_qml_module；Corrosion 引入 Rust
├─ qml/                               # Main.qml、pages/{Unlock,Entries,Otp,Settings,Pairing,Conflicts}.qml、
│                                     # windows/{PickerWindow,SaveBubble,OtpMiniWindow,ScreenSnip}.qml、components/
├─ src/                               # pm-qt：C++ 服务与模型
│  ├─ VaultService / OtpService / SettingsService（QML_SINGLETON）
│  ├─ EntryListModel / OtpListModel（QAbstractListModel）
│  ├─ platform/PlatformServices.h     # 平台抽象接口
│  ├─ platform/win/                   # 托盘、热键事件过滤器、子窗口样式、截屏
│  └─ platform/android/               # JNI 封装（见 05）
└─ android/                           # Qt Android 包源码目录（见 05）
native/
├─ pm-core/                           # Rust：保险库、加密、匹配、TOTP、合并、通道
├─ pm-win/                            # Rust：Win32/WinRT 原生能力（cfg(windows)）
├─ pm-cxx/                            # Rust：cxx 桥接（生成 C++ 头/源），Windows 版链接 pm-win
└─ pm-native-host/                    # Rust：Native Messaging 宿主 exe
```
| 能力 | 实现位置 | 说明 |
|---|---|---|
| 所有界面（主窗口、动态码卡片页、设置、配对确认、保存/更新对话框） | QML（Qt Quick Controls，Material 风格 + 自定义 Token） | 主窗口 `ApplicationWindow`，左侧导航（自定义 `NavigationRail` 组件），宽度 < 600px 时折叠 |
| 辅助窗口（Picker、保存气泡、迷你动态码浮窗、屏幕框选覆盖层） | 独立 QML `Window` | `flags: Qt.Tool \| Qt.FramelessWindowHint \| Qt.WindowStaysOnTopHint \| Qt.WindowDoesNotAcceptFocus`；需要键盘输入时（用户点击后）去掉 `WindowDoesNotAcceptFocus` 并 `requestActivate()`；pm-qt 再对 `winId()` 补设 `WS_EX_NOACTIVATE` 确保不抢焦点 |
| 托盘图标与菜单 | `QSystemTrayIcon` + `QMenu`（C++） | 托盘鼠标按下时先调用 `pm_win::snapshot_foreground()` 缓存前台窗口，再弹出菜单 |
| 全局热键 | pm-qt：`RegisterHotKey` + `QAbstractNativeEventFilter` 处理 `WM_HOTKEY` | Qt 无内置全局热键；回调中**先**在原生侧记录前台 HWND，再发信号给 QML |
| 单实例 | `QLocalServer`/`QLocalSocket` | 第二个实例转发参数后退出 |
| 前台识别、UIA、WinEvent、SendInput、IME、UIPI 检测 | pm-win（Rust `windows` crate），后台线程（UIA 专用 MTA 线程） | 事件经 pm-cxx 回调 → `QMetaObject::invokeMethod(Qt::QueuedConnection)` → 信号 |
| 自动输入取值 | pm-win 直接从 pm-core 读取 `entryId` 对应字段 | **密码不经过 C++/QML** |
| 剪贴板（排除历史/云同步、定时清除） | pm-win（不用 `QClipboard`，因为它不能设置排除格式） | 设置 `ExcludeClipboardContentFromMonitorProcessing`、`CanIncludeInClipboardHistory=0`、`CanUploadToCloudClipboard=0` |
| 防截屏 | pm-win 对 `QQuickWindow::winId()` 调用 `SetWindowDisplayAffinity` | |
| Windows Hello 快速解锁、DPAPI | pm-win（WinRT `KeyCredentialManager`、`CryptProtectData`） | |
| ExtensionBridge（命名管道服务端） | pm-win（tokio `named_pipe`） | 配对请求经信号弹出 QML 确认对话框 |
| 摄像头扫码 | Qt Multimedia（`CaptureSession` + `Camera` + `VideoOutput`，C++ 侧 `QVideoSink` 取帧）→ pm-core `rxing` 解码 | |
| 屏幕框选扫码 | QML 全屏半透明覆盖窗口框选 → `QScreen::grabWindow(0, rect)` → `rxing` | 本应用自身窗口已排除截屏 |
| Native Messaging 宿主 | `pm-native-host.exe`（Rust） | §11 |
| 部署 | `windeployqt` 收集 Qt DLL 与 QML 模块；Rust 静态库链接进 `PasswordManager.exe` | Qt 动态链接（LGPL，02 §7） |

## 1. 模块总览
| 模块 | 实现 | 说明 |
|---|---|---|
| Shell/UI | QML + pm-qt | 解锁窗口、主列表、条目编辑、密码生成器、设置、“浏览器”页（扩展安装与配对管理）、动态码页 |
| Tray | `QSystemTrayIcon` + pm-win | 托盘图标与菜单（§9） |
| LockManager | pm-core（策略）+ pm-win（会话通知） | 自动锁定策略（§10） |
| ForegroundWatcher | pm-win | `SetWinEventHook(EVENT_SYSTEM_FOREGROUND, EVENT_OBJECT_NAMECHANGE)` 跟踪前台窗口、进程、标题 |
| AppIdentityResolver | pm-win | 由 HWND 得到进程名、exe 完整路径、签名主体、窗口类名、标题（§2） |
| EntryMatcher | pm-core | 应用规则与 URL 规则匹配、打分排序（§3） |
| BrowserUrlReader | pm-win | 回退方案：UIA 读取 Chrome/Edge/Firefox 地址栏（仅当无扩展连接） |
| AutoTypeEngine | pm-win（解析器在 pm-core） | 序列解析、焦点校验、`SendInput` 输出（§4–§5） |
| PasswordFieldDetector | pm-win | UIA 焦点事件 + `IsPassword` 检测密码框（§6） |
| **CredentialCaptureService** | pm-win（检测）+ QML（对话框窗口） | 手动输入后保存/更新：登录提交检测、保存/更新对话框（§7） |
| Picker | QML `PickerWindow` | 多条匹配时的选择窗口（§8） |
| Clipboard | pm-win | 排除剪贴板历史/云同步，30 秒自动清除 |
| ExtensionBridge / pm-native-host.exe | pm-win / Rust exe | 与浏览器扩展通信（§11，协议见 05b §7） |

## 2. 应用身份（App Identity）
```
HWND ──GetWindowThreadProcessId──► PID
PID ──QueryFullProcessImageNameW──► exe 完整路径（如 C:\Program Files\Tencent\QQNT\QQ.exe）
     └─ 进程名 = 文件名（QQ.exe，比较时大小写不敏感）
     └─ 签名主体（WinVerifyTrust + CertGetNameString，缓存）——可选强校验
HWND ──GetClassNameW / GetWindowTextW──► 窗口类名、标题
UWP / 打包应用：ApplicationFrameHost 托管时取子窗口真实进程；记录 AUMID（GetApplicationUserModelId）
```
条目中的 `windowsApps` 规则：
```json
{ "process": "QQ.exe",
  "exePath": "*\\Tencent\\QQNT\\QQ.exe",      // 可选，通配符；%ProgramFiles% 等环境变量展开
  "titlePattern": "^QQ$|登录",               // 可选，正则（超时 50 ms）
  "windowClass": null,                       // 可选
  "publisher": "Tencent Technology(Shenzhen) Company Limited", // 可选，签名主体
  "aumid": null }                            // 可选，UWP
```

## 3. 匹配算法与优先级
### 3.1 候选生成
1. 浏览器且有扩展会话 → 交给扩展（§11），桌面不匹配。
2. 浏览器无扩展 → `BrowserUrlReader` 取 URL → 按 05b §6 的 URL 规则匹配（结果标注“地址未经扩展校验”，不允许自动回车提交）。
3. 其它应用 → 遍历所有条目的 `windowsApps` 规则。

### 3.2 规则命中与打分
每条规则内所有**已填写**字段都必须命中才算命中；分数为各字段权重之和：

| 字段 | 权重 |
|---|---|
| `aumid` 一致 | 50 |
| `exePath` 命中 | 40 |
| `publisher` 一致 | 30（若规则填写了 publisher 而签名不符 → 规则不命中，并在日志中警告“可能是仿冒程序”） |
| `process` 一致 | 20 |
| `titlePattern` 命中 | 15 |
| `windowClass` 一致 | 10 |

排序：分数降序 → 条目“最近使用”降序 → 标题。分数最高者唯一且 ≥ 35（如 进程名+标题）时可直接输入；否则弹出 Picker。只命中 `process`（20 分）时总是弹出 Picker，避免同一进程（如 `javaw.exe`、`electron` 应用）多个程序混淆。

### 3.3 排除
全局 `neverFillApps`（进程名/路径）与锁屏、UAC 安全桌面、本应用自身窗口一律不匹配。

## 4. 自动输入序列语法
默认序列：`{USERNAME}{TAB}{PASSWORD}{ENTER}`；条目可覆盖，也可为同一条目的不同窗口规则（如“登录窗口”“二次验证窗口”）设置不同序列。

| 占位符 / 指令 | 含义 |
|---|---|
| `{USERNAME}` `{PASSWORD}` `{TOTP}` `{TITLE}` `{URL}` `{NOTES}` | 条目字段；`{TOTP}` 在输出时才计算，剩余 < 3 秒自动等待下一周期（08 §5） |
| `{S:字段名}` | 自定义字段 |
| `{TAB}` `{ENTER}` `{SPACE}` `{BACKSPACE}` `{DELETE}` `{ESC}` `{UP}` `{DOWN}` `{LEFT}` `{RIGHT}` `{HOME}` `{END}` | 功能键 |
| `{TAB 3}` | 重复 N 次 |
| `^a` `+{TAB}` `%{F4}` | 修饰键：`^`=Ctrl，`+`=Shift，`%`=Alt（与 KeePass 语法兼容） |
| `{CLEARFIELD}` | 相当于 `^a{BACKSPACE}`，清空当前框 |
| `{DELAY 500}` | 暂停 500 ms |
| `{DELAY=30}` | 将之后每个按键间隔设为 30 ms |
| `{VKEY 0x0D}` | 发送指定虚拟键 |
| `{WAIT:PASSWORDFIELD 3000}` | 等待焦点进入密码框（UIA 判断，最长 3 秒），用于两步登录 |
| `{MODE:UNICODE}` / `{MODE:VK}` | 字符输出方式（§5.2） |
| `{{}` `{}}` | 字面量花括号 |

解析器生成指令列表 `[Text("alice"), Key(VK_TAB), Text(<secret>), Key(VK_RETURN)]`；秘密值以 `char[]` 持有，执行后清零。语法错误在条目编辑界面实时提示，并提供“测试输入到记事本”按钮。

## 5. SendInput 细节
### 5.1 执行流程
```
触发（热键 / Picker 选择 / 托盘）
 → 记录目标 HWND、PID（触发瞬间的前台窗口）
 → 若由 Picker 触发：关闭 Picker，SetForegroundWindow(目标) 并等待最多 500 ms 确认
 → 等待所有修饰键释放（GetAsyncKeyState 检查 Ctrl/Alt/Shift/Win，最长 1 秒；
    否则用户按着 Ctrl+Alt 热键时输入会变成快捷键）
 → 逐条指令输出；**每条指令前**做焦点校验（§5.4）
 → 结束后清零秘密缓冲
```
### 5.2 字符输出
- 默认 **Unicode 模式**：每个字符发送 `INPUT{type=KEYBOARD, wVk=0, wScan=ch, dwFlags=KEYEVENTF_UNICODE}` 按下 + `KEYEVENTF_UNICODE|KEYEVENTF_KEYUP` 抬起；UTF-16 代理对拆为两个连续事件。优点：与键盘布局无关，支持中文/符号。
- **VK 模式**（兼容不处理 `VK_PACKET` 的程序，如部分游戏、DirectInput、远程桌面客户端、Java 旧版 Swing）：`VkKeyScanExW(ch, 目标线程的 HKL)` 得到虚拟键+修饰键，按需包裹 Shift/Ctrl/Alt；无法映射的字符回退到 Unicode 并记录警告。条目可强制某模式，全局可设默认。
- 功能键（Tab/Enter 等）始终用虚拟键 + 扫描码（`MapVirtualKeyExW`），方向键等设置 `KEYEVENTF_EXTENDEDKEY`。
- 一次 `SendInput` 调用只发送一个字符的按下/抬起对，以配合按键间隔并避免被 UIPI 部分拦截时难以察觉；检查返回值，小于期望个数即中止并提示。

### 5.3 输入法（IME）干扰
- 问题：中文输入法处于中文模式时，VK 模式的字母会进入候选窗，导致输入错误；Unicode 模式一般可绕过，但部分输入法仍会拦截。
- 对策：
  1. 输入前对目标窗口 `ImmGetContext` → 记录 `ImmGetOpenStatus`，若开启则临时 `ImmSetOpenStatus(false)`（或切到英文模式 `ImmSetConversionStatus`），输入完成后恢复；跨进程不可用时改为发送 IME 切换由用户配置的按键（如 `{VKEY 0x10}` Shift），默认不发送。
  2. 若检测到 TSF 输入法且无法关闭，在首次失败后提示“请先切换到英文输入状态”，并允许条目设置 `{MODE:UNICODE}`。
  3. 密码框通常由系统禁用 IME（`ES_PASSWORD`），风险主要在用户名字段。

### 5.4 焦点校验
每条指令执行前检查：`GetForegroundWindow()` 的根窗口（`GetAncestor(GA_ROOT)`）与 PID 仍是触发时的目标；若变化则**立即中止**，清零缓冲，托盘提示“目标窗口已改变，自动输入已取消”。可选强校验：输出 `{PASSWORD}` 前用 UIA `FocusedElement` 确认焦点在 `IsPassword=true` 的元素上（目标不支持 UIA 时跳过并记录）。防止密码被打进聊天窗口等错误位置。

### 5.5 UIPI 与提升权限窗口
- 普通权限的本应用无法向**以管理员运行**（高完整性级别）的窗口发送输入（UIPI，SendInput 静默失败），也无法挂接其 UIA。
- 检测：`OpenProcessToken` + `GetTokenInformation(TokenIntegrityLevel)` 比较目标与自身；若目标更高 → 不尝试输入，提示“目标程序以管理员身份运行，请改用复制粘贴，或以管理员身份运行 PasswordManager（不推荐）”。
- UAC 安全桌面、锁屏、`Ctrl+Alt+Del` 界面不可输入，属系统设计。
- 不使用 `uiAccess=true` 清单（需签名并安装到受保护目录，且扩大攻击面）——列为后续评估项。

### 5.6 延迟设置
| 设置 | 默认 | 说明 |
|---|---|---|
| 启动延迟 | 150 ms | 热键释放到开始输入 |
| 按键间隔 | 10 ms（0–200） | 慢速程序/远程桌面可调大 |
| 功能键后额外延迟 | 50 ms | 如 `{TAB}` 后等待焦点切换 |
| `{ENTER}` 前延迟 | 100 ms | |
| 每条目覆盖 | 可 | 存于条目 `autoType.delays` |

## 6. 密码框检测（PasswordFieldDetector）
- 订阅 UIA `AddFocusChangedEventHandler`（在专用 MTA 线程，避免阻塞 UI）；读取焦点元素 `IsPassword`、`ControlType`、`AutomationId`、`Name`、`ClassName`、`ProcessId`。
- Win32 编辑框回退：`GetGUIThreadInfo` 取焦点控件 → `GetWindowLong(GWL_STYLE) & ES_PASSWORD`。
- **永不读取密码框的值**：UIA 对 `IsPassword` 元素的 `ValuePattern` 不返回明文（设计如此），本应用也不尝试其它方式读取。
- 聚焦密码框时：若有匹配条目 → 托盘气泡/小浮窗“按 Ctrl+Alt+A 填充 QQ（alice）”；若无匹配 → 记录“该窗口出现过密码框”，供 §7 的保存提示使用。
- 自绘界面（游戏、部分 Electron/Qt/DirectUI 程序）可能不暴露 UIA：此时检测失败，用户仍可用热键或托盘手动操作。

## 7. 手动输入后保存/更新凭据（CredentialCaptureService）

### 7.1 设计约束
- **不能读取密码**：UIA 不暴露密码框内容；读取其它进程内存属恶意行为，不做。
- **不默认使用全局键盘钩子**：`SetWindowsHookEx(WH_KEYBOARD_LL)` 等同键盘记录器，会被杀毒软件/EDR 告警，且一旦本应用被利用即成为键盘记录器。**结论：默认不实现**；仅作为“实验性”高级选项列入后续评估（§7.6），v1 不提供。
- 因此 Windows 原生应用的保存流程为“**检测到登录 → 弹出预填好应用身份和用户名的对话框 → 用户自行输入/粘贴密码**”。

### 7.2 登录提交检测（基于 UIA，只看事件不看密码）
当某前台窗口出现过密码框焦点（§6），且密码框“非空”可判定时（UIA `ValuePattern` 对密码框通常不可读；用 `TextPattern`/`LegacyIAccessible.Value` 长度掩码“●●●”可得非空，若都不可得则视为“未知”）进入**候选登录状态**，随后满足任一信号即判定“可能已提交登录”：

| 信号 | 检测方式 |
|---|---|
| 在密码框中按 Enter | UIA 无键盘事件；改用 `EVENT_OBJECT_FOCUS`/`EVENT_OBJECT_STATECHANGE` 观察焦点离开 + 窗口变化组合判断（不挂键盘钩子） |
| 点击“登录”类按钮 | UIA `InvokePattern.InvokedEvent`（按钮 Name/AutomationId 含 登录/登陆/Sign in/Log in/OK/确定） |
| 登录窗口关闭或隐藏 | `EVENT_OBJECT_DESTROY` / `EVENT_OBJECT_HIDE`（同 PID 的顶层窗口）在密码框聚焦后 30 秒内发生 |
| 同进程窗口切换为主界面 | 同 PID 出现新顶层窗口且标题/类名变化（如 “QQ 登录” → “QQ”） |
| 密码框从 UIA 树中消失 | `StructureChangedEvent` |

判定后去抖 1.5 秒（避免登录失败重试时多次提示），再执行 §7.3。用户在设置中可关闭“检测到登录时提示保存”（默认开，但对每个应用首次出现时询问一次“以后在此应用登录时提示保存吗？”）。

### 7.3 新建 vs 已有判断
- 用 §3 的规则匹配当前应用身份 → 得到候选条目集合 C。
- 尝试读取用户名：在同一窗口的 UIA 树中找到距离密码框最近的、`IsPassword=false` 的 Edit/ComboBox 元素，读取 `ValuePattern.Value`（明文可读的普通字段）；读不到则留空。
- 判断：
  - C 为空 → **新建**模式。
  - C 中存在用户名相同（大小写不敏感、去空白）的条目 → **更新候选**模式（由于读不到密码，无法自动判断“密码是否改变”，由用户决定是否更新）。
  - C 非空但用户名不同/未知 → **新建或选择已有**模式（同一应用多账号）。
- 若该应用身份在 `neverSaveApps` 中 → 不提示。

### 7.4 保存/更新对话框
```
┌ 保存到 PasswordManager ───────────────────────────────┐
│ 检测到在「QQ」登录                                         │
│ 应用：QQ.exe   路径：C:\Program Files\Tencent\QQNT\QQ.exe  │
│ 匹配规则：[✓] 进程名 [✓] 路径 [ ] 窗口标题 ^QQ$ [ ] 签名主体 │
│ 标题：[ QQ                    ]                          │
│ 用户名：[ 123456789           ]（已从窗口读取，可修改）      │
│ 密码：  [ ●●●●●●●●  ] 👁  （请输入或粘贴；本应用无法读取原窗口密码）│
│ ( ) 新建条目   (•) 更新已有：[ QQ - 123456789 ▼ ]           │
│ [保存]  [本应用从不]  [忽略]                               │
└────────────────────────────────────────────────────────┘
```
- 非模态、置顶、不抢焦点（QML `SaveBubble` 窗口，`Qt.WindowDoesNotAcceptFocus` + pm-qt 补设 `WS_EX_NOACTIVATE`；气泡先出现，点击后才展开为对话框并激活），避免打断登录流程或让用户误把密码打进对话框之外。
- **保存**：新建条目，写入 `windowsApps` 规则（按勾选字段生成；默认勾选进程名+路径）。
- **更新已有**：选择目标条目 → 若输入的密码与原密码不同，旧密码进入 `passwordHistory`（见 01 FR-C6），更新 `updated`；若相同则仅补充应用规则（如原条目只有网址，现在关联此应用）。
- **本应用从不**：加入 `neverSaveApps`（按 exe 路径），可在设置中撤销。
- **忽略**：本次不处理；同一窗口会话内不再提示。
- 保险库锁定时：先显示“解锁以保存”，解锁后继续；暂存的身份/用户名在内存中最长 5 分钟。

### 7.5 手动“保存当前应用凭据”
- 全局热键 **Ctrl+Alt+S**（可改）与托盘菜单项“保存当前应用凭据…”。
- 触发时记录**触发前**的前台窗口（托盘菜单打开会改变前台，故在菜单弹出前缓存 `GetForegroundWindow`），打开 §7.4 对话框（同样预填应用身份与可读用户名）。
- 适用：UIA 检测不到的自绘程序、游戏启动器、关闭了自动提示的应用。

### 7.6 键盘钩子（不在 v1 范围，若将来提供须满足）
- 默认关闭、需在“高级 → 实验性”中手动开启并阅读风险说明（会被安全软件视为键盘记录器、可能导致误报或被拦截）。
- 只在 UIA 确认焦点处于 `IsPassword=true` 元素且属于用户勾选的应用时缓冲按键，焦点离开即清零；不记录到磁盘；不处理粘贴、IME 组合输入（准确性有限）。
- 本设计文档的推荐结论：**不实现**，以 §7.4/§7.5 的人工输入方式替代。

## 8. Picker（多条匹配选择器）
- QML `PickerWindow`（无边框、置顶、`Qt.WindowDoesNotAcceptFocus` 直到用户点击），出现在目标窗口附近（Per-Monitor V2 DPI 感知，位置由 pm-win 按目标窗口矩形计算）；显示：图标（exe 图标）、标题、用户名、匹配原因（“路径+标题”）。
- 键盘：数字 1–9 直接选择、↑↓、Enter、Esc；输入文字即时过滤全库（可选“仅匹配项”）。
- 次级操作：`Ctrl+C` 复制密码、`Ctrl+B` 复制用户名、`Ctrl+E` 编辑条目、“把此窗口关联到所选条目”。
- 选择后关闭 Picker → 恢复目标前台 → §5.1 流程。

## 9. 托盘
菜单：打开主窗口｜搜索并填充…｜保存当前应用凭据…（§7.5）｜动态码…（迷你浮窗）｜复制当前应用动态码｜从屏幕扫描 2FA 二维码…｜生成密码并复制｜立即锁定｜浏览器扩展状态（已连接的浏览器列表）｜设置｜退出。
图标状态：已锁定（灰）、已解锁（彩色）、有待处理保存提示（带点）。双击打开主窗口；解锁状态下左键单击打开快速搜索。

## 10. 锁定
| 触发 | 默认 |
|---|---|
| 空闲（`GetLastInputInfo`） | 10 分钟 |
| 系统锁屏（`WTSRegisterSessionNotification` → `WTS_SESSION_LOCK`） | 立即 |
| 睡眠/休眠（`PBT_APMSUSPEND`） | 立即 |
| 用户切换、远程会话断开 | 立即 |
| 手动 / 托盘 / 热键 Ctrl+Alt+L | — |
锁定时：清空内存中解密数据与秘密缓冲、清空本应用写入的剪贴板、关闭编辑窗口（未保存修改提示后丢弃）、通知 ExtensionBridge 推送 `vault.locked`、丢弃暂存的待保存凭据。快速解锁：Windows Hello（`KeyCredentialManager` 签名派生密钥包装 Vault Key，仅本机）。

## 11. ExtensionBridge 与 NativeHost
- 管道 `\\.\pipe\PasswordManager.<UserSID>.ext`，由 pm-win 以 tokio `ServerOptions` 创建：安全描述符仅授予当前用户 SID、`first_pipe_instance(true)`、`reject_remote_clients(true)`。
- 客户端校验：`GetNamedPipeClientProcessId` → 映像路径为安装目录下 `pm-native-host.exe` 且签名主体一致（开发构建可关闭）。
- NativeHost 告知 `{browser, extId, browserPid}`；仅白名单扩展 ID；未配对只接受配对消息，配对请求弹出置前对话框显示 SAS。
- 前台为已连接扩展的浏览器时，热键转为 `fill.trigger`，Ctrl+Alt+S 转为 `save.trigger`（扩展对当前页打开保存提示）。
- 锁定时返回 `LOCKED`；可选“每次填充前在桌面确认”；审计日志（不含密码）。

## 12. 安装器与设置
- 用户级安装（MSIX 或 Inno Setup；包含 `windeployqt` 收集的 Qt DLL/QML 模块、`PasswordManager.exe`（静态链接 Rust 核心）、`pm-native-host.exe`、Qt LGPL 许可证文本）；为 Chrome、Edge、Firefox 写宿主清单与 HKCU 注册表（05b §8）；“修复宿主注册”按钮；卸载清理。

| 设置 | 默认 |
|---|---|
| 自动输入热键 / 保存热键 / 锁定热键 / 动态码热键 | Ctrl+Alt+A / Ctrl+Alt+S / Ctrl+Alt+L / Ctrl+Alt+T |
| 默认序列 | `{USERNAME}{TAB}{PASSWORD}{ENTER}` |
| 字符模式 | Unicode |
| 输入前校验焦点在密码框 | 关（目标不支持 UIA 时无效） |
| 检测到登录时提示保存 | 开（每个应用首次询问） |
| 启用浏览器扩展连接 / 页面加载自动填充 / 填充前确认 | 开 / 关 / 关 |
| 扩展请求时若应用未运行则自动启动 | 询问 |

## 13. 测试
- 单元：序列解析器、打分器、应用规则匹配（Rust `cargo test`）；IME 状态恢复逻辑（模拟）；Qt Quick Test（QML `TestCase`）+ 截图对比（`QQuickWindow::grabWindow`）覆盖 Picker、保存对话框、动态码卡片；Qt Test 覆盖 C++ 模型（`QAbstractItemModelTester`）。
- 集成：自带测试目标程序（Win32 / WinForms / WPF / Electron 对话框，含密码框、登录按钮、登录后关闭窗口）验证提交检测与对话框预填；记事本验证 Unicode/VK 输出；管理员权限目标验证 UIPI 提示；输入过程中切换窗口验证中止。
- 手工矩阵：QQ、微信、Steam、企业 VPN 客户端、远程桌面（mstsc）、Java/Electron/Qt 应用 × 微软拼音/搜狗输入法。

## 14. 动态码（TOTP）
> 通用设计见 08。
- **动态码库**：默认独立的 `otp.pmv`（独立密码、默认空闲 5 分钟锁定、可单独启用 Windows Hello），与密码保险库分别解锁；托盘图标菜单分别显示两库的锁定状态。
- **动态码页**：主窗口左侧导航「动态码」页，QML 实现（08 §11.11），采用 08 §11 的类身份验证器卡片界面（深色/绿色强调、卡片“发行方: 账号”、大号分组码、⧉ 复制、✦ 输入到上一个窗口、“N秒”+圆形进度、右下角 ＋ FAB、搜索、置顶、拖动排序）；卡片网格随窗口宽度 1–3 列；点击码即复制；键盘快捷键见 08 §11.8。托盘左键弹出只含置顶项的迷你动态码浮窗。窗口启用 `SetWindowDisplayAffinity(hwnd, WDA_EXCLUDEFROMCAPTURE)`（Win10 2004+，更早用 `WDA_MONITOR`），设置中可关闭。
- **Auto-Type**：
  - 两步登录的应用常在登录后弹出“输入验证码”窗口：条目可添加第二条窗口规则（如 `titlePattern: "安全验证|二次验证|Two-factor"`）并配置序列 `{TOTP}{ENTER}`。
  - 热键 **Ctrl+Alt+T**：按动态码项自带的 `match.windowsApps` 规则匹配当前前台窗口并只输入动态码（多条匹配时 Picker 仅列出动态码项）；动态码库锁定时先弹出动态码库解锁窗口；前台为已连接扩展的浏览器时转为 `totp.fill.trigger` 事件交给扩展。
  - `{TOTP}` 输出前执行边界等待；等待期间若焦点改变则按 §5.4 中止。
- **导入**：
  - “从屏幕扫描二维码”：QML 全屏半透明覆盖窗口（`ScreenSnip.qml`）框选区域 → `QScreen::grabWindow(0, rect)` 抓取像素到内存 → pm-core `rxing` 解码；本应用窗口自身已被排除截屏，不影响抓取网页二维码。
  - 图片文件（QML `FileDialog`）/ 剪贴板图片（`QClipboard::image()`，只读）/ 摄像头（Qt Multimedia 取帧 → `rxing` 解码，`QMediaDevices::videoInputs()` 为空时隐藏入口）。
  - 粘贴 `otpauth://` / `otpauth-migration://` 文本；手工输入 Base32。
  - 导入预览与比对、提示删除源图片（08 §3）。
- **时钟检测**：启动和每小时检查 W32Time 服务运行状态与“自动设置时间”；未同步时动态码页面顶部告警并提供“打开日期和时间设置”（`ms-settings:dateandtime`）。
- **剪贴板**：动态码复制后在过期或 30 秒（较早者）清除；设置“禁止复制动态码”。
- **托盘**：“复制当前应用动态码”（按托盘菜单打开前缓存的前台窗口匹配）。
