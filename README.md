# PasswordManager —— 跨平台密码管理器（设计文档）

> 当前阶段：**仅设计文档，暂无代码**。本仓库描述一个支持 **Windows 桌面、Android、浏览器（Chrome / Edge / Firefox）** 的本地优先密码管理器：
> 统一管理账号密码，自动识别当前正在使用的 **应用 / 网站**，并自动填充用户名与密码。

## 核心能力一览

| 平台 | 识别方式 | 填充方式 |
|---|---|---|
| Windows 桌面应用 | 前台窗口进程名 / exe 路径 / 签名 / 窗口标题（WinEvent + UI Automation） | 全局热键或密码框聚焦提示 → SendInput 自动输入（可自定义序列）；登录后弹出保存/更新对话框；`{TOTP}` 输入动态码 |
| 浏览器（Windows 上的 Chrome / Edge / Firefox） | 浏览器扩展读取标签页/框架的真实 Origin | 内容脚本识别登录表单并直接填值；弹出窗口搜索/复制；手动输入后保存/更新提示（含改密表单）；动态码输入框填充 |
| Android 应用 | AutofillService：请求方包名 + 签名证书 | 系统自动填充数据集（需生物识别/主密码解锁）；SaveInfo 保存/更新；OTP 字段填充动态码 |
| Android 浏览器 | AutofillService：`ViewNode.webDomain`（原生支持的浏览器）+ 兼容模式（地址栏解析） | 同上；仅信任白名单浏览器上报的域名 |

**动态码（2FA / TOTP）**：内置身份验证器，界面为类似常见身份验证器 App 的卡片列表（深色主题、绿色强调、大号分组动态码、“N秒”圆形倒计时、点击即复制、右下角 ＋ 添加、搜索与置顶）。动态码密钥**默认保存在独立的动态码库 `otp.pmv`，使用与主密码不同的独立密码**，密码保险库泄露也不会让两步验证失效。可扫描网站两步验证二维码或导入 Google 身份验证器（`otpauth-migration`），并在 Windows 应用（`{TOTP}` / Ctrl+Alt+T）、浏览器 OTP 输入框、Android 自动填充中直接填入；支持 SHA1/SHA256/SHA512、6/8 位、自定义周期，兼容 HOTP 与 Steam Guard。浏览器扩展只拿到动态码，从不接触密钥。

**手动输入后保存/更新**：在网页、Windows 应用、Android 中手动输入账号密码登录或修改密码后，自动判断是新账号还是已有账号，提示“保存 / 更新 / 此站点（应用）从不 / 忽略”；更新时保留密码历史；支持同一站点多个账号。Windows 原生应用无法读取密码框内容，因此保存对话框会预填应用信息与用户名，由用户输入/粘贴密码（不使用键盘钩子）。

关键安全原则：
- **保险库只存在于桌面端 / 手机端应用中**。浏览器扩展**从不存储保险库**，也不持有主密钥，只在填充瞬间于内存中接收单条凭据。
- 扩展 ↔ 桌面应用通过 **Native Messaging** 通信，并在其上叠加 **配对（密钥交换 + 短码确认）后的端到端加密通道**。
- **动态码与密码分库存放**：`otp.pmv` 独立密码、独立密钥、独立锁定与同步位置；“同库存储”仅为不推荐的高级选项。
- 网址匹配基于 **公共后缀列表（PSL）的可注册域名**，含子域名规则与反钓鱼保护（同形异义域名、相似域名、跨域 iframe、非 HTTPS 等）。

## 文档索引

| 编号 | 文档 | 内容 |
|---|---|---|
| 01 | [需求规格说明](docs/01-需求规格说明.md) | 功能/非功能需求，含浏览器扩展、Android 浏览器填充、**手动输入后保存/更新（§3.6）**、**动态码管理（§3.7）** |
| 02 | [系统架构设计](docs/02-系统架构设计.md) | **Qt 6/QML + C++ 适配层 + Rust 核心 + Kotlin 原生**的架构图、组件职责、数据流、进程模型、技术选型、**选型取舍（为何 Qt、Rust 还是 C++ 核心、cxx 桥接）**、**Qt 许可证说明** |
| 03 | [密码学设计与威胁模型](docs/03-密码学设计与威胁模型.md) | 保险库加密、文件格式（含密码历史、动态码字段）、扩展通道加密与配对、威胁模型 T1–T30（含保存流程、键盘钩子、动态码与分库风险） |
| 04 | [Windows 桌面端设计](docs/04-Windows桌面端设计.md) | Qt/QML 界面、pm-qt 与 pm-win 分工（§0）、应用身份与匹配算法、Auto-Type 序列语法与 SendInput/IME/UIPI、Picker、密码框检测、**手动输入后保存/更新**、托盘、锁定、Native Messaging 宿主、**动态码页（卡片界面）/独立动态码库/扫码/{TOTP}（§14）** |
| 05 | [Android 端设计](docs/05-Android端设计.md) | Qt for Android + Kotlin 原生模块（AutofillService 与原生 Activity）+ JNI/UniFFI 工程结构（§0）、AutofillService、浏览器 webDomain、兼容模式、生物识别、**SaveInfo/onSaveRequest 保存与更新**、**动态码页、OTP 自动填充与扫码（§8）** |
| 05b | [浏览器扩展设计](docs/05b-浏览器扩展设计.md) | MV3 扩展、表单识别、填充、弹出窗口、**保存/更新捕获（submit/SPA/XHR/改密）**、URL 匹配、反钓鱼、Native Messaging 协议、**动态码输入框填充（§13）** |
| 06 | [同步设计](docs/06-同步设计.md) | 文件格式与同步、三方字段级合并、墓碑、冲突中心、网盘/Syncthing、Android SAF、示例 |
| 07 | [开发计划](docs/07-开发计划.md) | 里程碑（约 37 周，Qt/QML 单界面）、任务拆分、测试与发布计划、风险 |
| 08 | [动态码（2FA / TOTP）设计](docs/08-动态码TOTP设计.md) | **独立动态码库 `otp.pmv`（默认）**与数据模型、TOTP/HOTP/Steam 算法、otpauth / otpauth-migration / 二维码导入、显示与防截屏、复制与填充、时钟偏差、安全与导出、同步、测试向量、**类身份验证器卡片界面规格（线框、交互、FAB、搜索、置顶、各端布局、商标说明、QML 组件结构 §11.11）** |

> 编号说明：浏览器扩展与 Android 同属“客户端/填充端”设计，故编为 05b，紧随 05；不打乱其余编号；动态码作为跨端专题编为 08。

## 技术路线
- **界面**：一个 **Qt 6 应用（QML / Qt Quick）** 同时覆盖 Windows 桌面与 Android；C++ 适配层 `pm-qt` 提供 QML 使用的服务与列表模型，托盘、多窗口（Picker、保存气泡、迷你动态码浮窗）、摄像头、截屏由 Qt 提供。
- **核心**：Rust 库 `pm-core`（保险库、Argon2id + AES-256-GCM、URL 匹配、保存判定、TOTP、同步合并、扩展通道加密），经 `cxx` 桥接供 C++/Qt 调用、经 UniFFI 供 Android Kotlin 调用；明文秘密默认只存在于 Rust 内存。保留 Rust 而非改用 C++ 核心的理由见 [02 §6](docs/02-系统架构设计.md)。
- **平台原生能力**：Windows 自动输入 / UI Automation / 受保护剪贴板 / Native Messaging 管道在 Rust `pm-win`；Android 自动填充服务、生物识别、FLAG_SECURE、SAF 为 Kotlin 原生代码（Qt 无法接管 AutofillService 生命周期，自动填充流程中的小界面也用原生 Activity）。
- **浏览器扩展**：独立的 TypeScript（MV3）工程，经 Native Messaging 与桌面应用通信。
- **许可证**：Qt 以 LGPLv3（动态链接、只用 LGPL 模块）或商业许可使用，义务与取舍见 [02 §7](docs/02-系统架构设计.md)（非法律意见）。

## 计划中的目录结构

```
PasswordManager/
├─ README.md
├─ docs/                         # 本设计文档
├─ app/                          # Qt 6 应用（Windows + Android 共用）
│  ├─ CMakeLists.txt             # qt_add_executable / qt_add_qml_module；Corrosion 引入 Rust
│  ├─ qml/                       # Main.qml、pages/、otp/（动态码卡片）、windows/（Picker、保存气泡、迷你浮窗）、components/
│  ├─ src/                       # pm-qt：C++ 服务（VaultService、OtpService…）、列表模型、平台抽象
│  │  └─ platform/{win,android}/ # 托盘/热键/窗口样式；JNI 封装
│  └─ android/                   # Qt Android 包源码目录：AndroidManifest、Gradle、Kotlin
│     └─ src/com/passwordmanager/{platform,autofill}/   # 生物识别、FLAG_SECURE、SAF；AutofillService 与原生 Activity
├─ native/                       # Rust workspace
│  ├─ pm-core/                   # 保险库 / 加密 / 匹配 / TOTP / 合并 / 通道（唯一核心实现）
│  ├─ pm-win/                    # Windows 原生能力（SendInput、UIA、剪贴板、DPAPI、Hello、命名管道）
│  ├─ pm-cxx/                    # cxx 桥接（Rust ↔ C++）；UniFFI 导出供 Kotlin
│  └─ pm-native-host/            # Native Messaging 宿主 exe（stdio ↔ 命名管道，极简）
├─ browser-extension/            # MV3 扩展（TypeScript），Chrome/Edge/Firefox 共用代码
│  ├─ src/{background,content,popup,inline-menu,notification-bar}/
│  ├─ manifests/{chromium.json,firefox.json}
│  └─ native-host-manifests/     # 宿主清单模板（安装器写注册表）
└─ testvectors/                  # 共享测试向量：crypto / channel / url-match / save-compare / merge / totp
```

## 安全提示（摘要）
- 主密码不可找回；忘记即无法解密。请离线保存恢复信息。
- 浏览器扩展必须先与桌面应用 **配对** 才能使用；配对时请核对两端显示的 6 位短码一致。
- 默认 **不在页面加载时自动填充**，需用户点击/快捷键触发；域名不匹配时绝不静默填充。
- 详见 [03-密码学设计与威胁模型](docs/03-密码学设计与威胁模型.md)。
