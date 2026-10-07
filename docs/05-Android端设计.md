# 05 Android 端设计

> 技术路线：界面为与 Windows 共用的 **Qt 6 / QML 应用**（Qt for Android，运行在 `QtActivity` 中）；核心逻辑在 Rust `pm-core`（`libpm_core.so`）；系统集成（自动填充、生物识别、FLAG_SECURE、SAF、敏感剪贴板）为 **Kotlin 原生代码**：AutofillService 经 UniFFI 直接调用 pm-core，QML/C++ 经 JNI（`QJniObject`）调用 Kotlin 帮助类。详见 02 §5–§7。
> **最低版本**：以所用 Qt 版本的官方最低支持为准；Qt 6.8 为 **Android 9（API 28）**，因此 minSdk 提升为 28（原设计为 26）。这也使兼容模式（需 API 28）在所有支持设备上可用，§5 中 API 26–27 的回退路径不再需要。

## 0. 工程结构
```
app/android/                    # Qt Android 包源码目录（QT_ANDROID_PACKAGE_SOURCE_DIR）
├─ AndroidManifest.xml          # QtActivity（主界面）+ 自动填充服务 + 原生 Activity 声明
├─ build.gradle                 # 在 androiddeployqt 生成的模板上增加 Kotlin 插件与 androidx 依赖
├─ res/xml/autofill_service.xml
└─ src/com/passwordmanager/
   ├─ platform/                 # 供 C++ 经 JNI 调用的 Kotlin 帮助类（静态方法 / @JvmStatic）
   │   ├─ BiometricBridge.kt    # BiometricPrompt + CryptoObject（Keystore 包装密钥），结果经 JNI 回调 C++
   │   ├─ SecureWindow.kt       # 在 UI 线程为 QtActivity 窗口设置/清除 FLAG_SECURE
   │   ├─ SafStorage.kt         # ACTION_OPEN/CREATE_DOCUMENT、持久授权、读写字节
   │   ├─ SensitiveClipboard.kt # EXTRA_IS_SENSITIVE + 定时清除
   │   └─ AutofillStatus.kt     # 是否为当前自动填充服务、跳转系统设置
   └─ autofill/                 # 纯 Kotlin：PmAutofillService、AuthActivity、SaveConfirmActivity、
                                # PickActivity、TrustedBrowsers、DAL 校验
(libpm_core.so 由 Corrosion/cargo-ndk 构建：arm64-v8a / armeabi-v7a / x86_64；
 同一 .so 被 Qt 的 C++ 代码链接，也被 Kotlin 经 UniFFI 加载)
```
- **一个进程、一个核心**：QtActivity（QML + pm-qt）与 `PmAutofillService` 运行在同一进程，共享 `libpm_core.so` 中的解锁状态（Rust 单例 `CoreRuntime`）。
- **自动填充不依赖 Qt**：系统单独拉起 `PmAutofillService` 时只加载 `libpm_core.so`，`onFillRequest` 全程在 Kotlin + Rust 中完成，响应快、内存小。
- **自动填充流程中的界面使用原生 Kotlin Activity**（AuthActivity、SaveConfirmActivity、PickActivity）。原因：Qt for Android 以单个 `QtActivity` 承载整个 QML 场景，Qt 运行时按进程初始化一次；让 QtActivity 作为由系统 `IntentSender` 拉起、需要 `setResult(EXTRA_AUTHENTICATION_RESULT)` 后立即结束的独立 Activity，会与主界面实例冲突、启动慢（加载 Qt 库 + QML 引擎），且生命周期难以控制。这些界面都很简单（生物识别提示、单个密码框、保存/更新选择、条目列表），以 Material Components 原生实现，**视觉 Token 与 QML 界面保持一致**（同一套颜色/圆角/字号，08 §11.1）。这是 Qt 方案下少量的重复界面代码。
- **主界面中的平台能力**：QML → pm-qt `PlatformServices`（Android 实现）→ `QJniObject::callStaticMethod` → Kotlin 帮助类；需要在 Android UI 线程执行的操作通过 `QNativeInterface::QAndroidApplication::runOnAndroidMainThread`；异步结果（如生物识别成功）经 JNI `native` 回调方法回到 C++，再以信号通知 QML。

## 1. 模块
| 模块 | 实现 | 说明 |
|---|---|---|
| UI | Qt 6 QML + pm-qt（C++） | 解锁、列表、编辑、生成器、设置（含“自动填充服务”引导、浏览器支持说明）、动态码页、冲突中心 |
| Core | Rust `pm-core`（cxx → C++；UniFFI → Kotlin） | vault.pmv / otp.pmv 读写、Argon2id、**URL 匹配 + PSL**、应用匹配、compare、TOTP、合并 |
| PmAutofillService | Kotlin | `AutofillService` 实现，经 UniFFI 调用 pm-core |
| AuthActivity | Kotlin（透明 Activity） | 数据集认证：BiometricPrompt（Keystore 包装的 Vault Key / OTP Vault Key）或主密码 |
| SaveConfirmActivity / PickActivity | Kotlin（Material Components） | 手动输入后保存/更新确认（§5）；“搜索保险库…”选择条目 |
| 动态码页 / 扫码页 | QML | 底部导航「动态码」页：类身份验证器卡片列表、FAB 速拨添加、搜索、置顶（08 §11）；Qt Multimedia 摄像头取帧 → pm-core `rxing` 解码（§8） |
| OtpVaultSession | pm-core + `BiometricBridge` | 独立动态码库 `otp.pmv` 的解锁/锁定（独立密码 + 独立 Keystore 生物识别密钥，默认 5 分钟锁定） |
| TrustedBrowsers | Kotlin | 可信浏览器白名单（包名 + 签名证书 SHA-256 + 地址栏资源 ID） |
| 平台帮助类 | Kotlin（经 JNI 供 C++ 调用） | 生物识别、FLAG_SECURE、SAF、敏感剪贴板、自动填充状态 |

## 2. 识别请求方
`onFillRequest` 中取最近的 `FillContext.structure`：
1. **App 场景**：`structure.activityComponent.packageName` → 通过 `PackageManager` 取签名证书 SHA-256 → 匹配条目 `androidApps`（包名 + 可选证书指纹）。
2. **网页场景**：遍历所有 `ViewNode`，收集 `getWebDomain()`（API 26）与 `getWebScheme()`（API 28+，缺省视为 https 未知并作“弱匹配”处理）。
3. 判定规则：
   - 请求方包名 ∈ 可信浏览器白名单且签名匹配 → 使用 webDomain，按 05b §6 的 URL 规则匹配网站条目。
   - 请求方不是浏览器但含 webDomain（App 内 WebView）→ 仅当该域名通过 **Digital Asset Links**（`/.well-known/assetlinks.json` 声明 `delegate_permission/common.get_login_creds` 指向该 App 包名+证书）校验时才按网站匹配；否则只按 App 包名匹配（防 T12 伪造）。校验结果缓存 24 小时；离线时不信任。
   - 同一结构中出现多个不同 webDomain：只采用包含被聚焦字段的那个节点所属的 webDomain。

## 3. 浏览器支持
### 3.1 原生支持自动填充框架的浏览器
浏览器把网页表单作为虚拟视图暴露，并在节点上设置 `webDomain`，字段带 `autofillHints`/`htmlInfo`（`HtmlInfo` 的 `type`、`autocomplete` 属性可用于识别用户名/密码，规则与扩展 §3.1 一致）。
| 浏览器 | 说明 |
|---|---|
| Chrome | 较新版本需用户在 Chrome「设置 → 自动填充服务」选择“使用其他服务自动填充”（否则使用 Google 密码管理器）；应用内提供图文引导 |
| Firefox（GeckoView） | 原生支持 |
| Samsung Internet、Brave、Edge、Vivaldi 等 | 多数基于 Chromium 已支持；以实测矩阵为准，不支持者走兼容模式 |

### 3.2 兼容模式（Android 9 / API 28+）
在 `res/xml/autofill_service.xml` 声明：
```xml
<autofill-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:settingsActivity="com.passwordmanager.SettingsActivity">
  <compatibility-package android:name="com.android.chrome" android:maxLongVersionCode="10000000000"/>
  <compatibility-package android:name="org.mozilla.firefox" android:maxLongVersionCode="10000000000"/>
  <compatibility-package android:name="com.sec.android.app.sbrowser" android:maxLongVersionCode="10000000000"/>
  <compatibility-package android:name="com.microsoft.emmx" android:maxLongVersionCode="10000000000"/>
  <!-- … 其它浏览器，见 TrustedBrowsers 列表 -->
</autofill-service>
```
- 兼容模式下系统框架通过无障碍机制为不支持自动填充的浏览器生成 AssistStructure，并从**地址栏**取得 URL：框架会把地址栏节点的文本解析为域名并设置到该节点的 `webDomain`，同时阻止对地址栏本身填充。是否对某包启用兼容模式还受系统白名单（平台配置）控制。
- 本应用的兜底：若未找到 webDomain，按 TrustedBrowsers 中记录的地址栏资源 ID（如 `com.android.chrome:id/url_bar`）定位节点文本并解析（只取主机名，要求能解析为合法 URL；否则不填充）。
- 兼容模式下字段信息有限（无 `htmlInfo`），用户名/密码识别依赖 `inputType`（`TYPE_TEXT_VARIATION_PASSWORD`/`WEB_PASSWORD`）与 hint 文本。
- 兼容模式存在的限制在应用内说明：可能延迟、个别页面不触发。

### 3.3 不支持的情况
Android 上的浏览器扩展（Firefox Android 虽支持扩展但不支持 Native Messaging），因此 Android 不提供扩展版本。

## 4. 返回数据集
- 未解锁：返回带 `authentication` IntentSender 的“PasswordManager — 点击解锁”数据集；解锁后在 `AuthActivity` 中构造真实数据集返回（`EXTRA_AUTHENTICATION_RESULT`）。
- 已解锁（短时间窗口内，默认 5 分钟）：每个匹配条目一个 Dataset（标题 + 用户名），仍对密码值使用数据集级认证以防“已解锁即任意填充”可选。
- 钓鱼告警（与扩展一致）：相似域名/同形字/http 弱匹配的数据集带“⚠”标签，并要求在 AuthActivity 中确认。
- Android 11+：支持 `InlinePresentation`（输入法内联建议）。
- 附加“搜索保险库…”数据集，打开选择界面手动选择条目（选择后可询问“将此网站/App 关联到该条目”）。

## 5. 手动输入后保存/更新（SaveInfo / onSaveRequest）

> 通用规则见 01 §3.6。Android 上“是否弹出保存界面”由系统框架决定，本应用通过 `SaveInfo` 告诉系统要跟踪哪些字段、何时触发；新建还是更新由本应用在 `onSaveRequest` 之后的界面中确定。

### 5.1 在 onFillRequest 中附加 SaveInfo
```kotlin
val saveInfo = SaveInfo.Builder(
        SaveInfo.SAVE_DATA_TYPE_USERNAME or SaveInfo.SAVE_DATA_TYPE_PASSWORD,
        arrayOf(passwordId))                         // 必填：密码框
    .setOptionalIds(arrayOfNotNull(usernameId, newPasswordId, confirmId))
    .setFlags(flags)
    .setDescription("保存到 PasswordManager？")
    .apply {
        if (Build.VERSION.SDK_INT >= 28 && loginButtonId != null) setTriggerId(loginButtonId)
        if (Build.VERSION.SDK_INT >= 30) setNegativeAction(SaveInfo.NEGATIVE_BUTTON_STYLE_NEVER, neverIntentSender)
        else setNegativeAction(SaveInfo.NEGATIVE_BUTTON_STYLE_REJECT, null)
        if (Build.VERSION.SDK_INT >= 27) setCustomDescription(customDesc)  // 显示站点/应用名、图标
    }.build()
```
即使没有匹配条目（无数据集）也要返回仅含 `SaveInfo` 的 `FillResponse`，否则手动输入后不会触发保存。

| 标志 / API | 版本 | 用途与本应用策略 |
|---|---|---|
| `SAVE_FLAG_TRIGGER_ON_ALL_VIEWS_INVISIBLE` | 26 | 被跟踪的字段全部不可见时触发保存。**对浏览器、WebView、单 Activity + Fragment 的 App 总是设置**——这些场景登录成功后 Activity 不会 finish，否则永远不触发 |
| `SAVE_FLAG_DONT_SAVE_ON_FINISH` | 28 | Activity finish 时不触发。仅在已设置 `setTriggerId`（登录按钮）时使用，避免用户返回/取消时误弹 |
| `setTriggerId(AutofillId)` | 28 | 点击该视图（识别出的“登录/确定”按钮）时触发保存 |
| `SAVE_FLAG_DELAY_SAVE` | 29 | 多屏登录（第一屏用户名、第二屏密码）：第一屏响应设置此标志，并用 `FillResponse.setClientState(Bundle)` 携带用户名 AutofillId/值，第二屏的 `FillRequest.clientState` 读取后合并为一次保存 |
| `NEGATIVE_BUTTON_STYLE_NEVER` | 30 | 系统保存界面上的“从不”按钮 → 触发 `neverIntentSender`，写入 `neverSaveAndroidPackages` / `neverSaveDomains` |
| `setCustomDescription` | 27 | 在系统保存界面显示“GitHub（github.com）· alice”等上下文 |
| `SaveCallback.onSuccess(IntentSender)` | 28 | `onSaveRequest` 返回后启动本应用的保存确认 Activity（§5.3） |

识别新密码字段：`autofillHints` 含 `newPassword`（`HintConstants.AUTOFILL_HINT_NEW_PASSWORD`）、`HtmlInfo` 中 `autocomplete=new-password`、或标签/提示含“新密码/确认密码”；同时出现当前密码与新密码 → 视为改密表单（规则同 05b §4.1）。

不附加 SaveInfo 的情况：请求方在 `neverSave*` 列表；网页 scheme 为 http 且设置为不提示；无法识别密码框。

### 5.2 系统何时不会弹出保存界面
- 用户提交的值与本应用刚填充的数据集完全相同（框架自动比较）——对应“已保存，不提示”。
- 目标 App 设置了 `importantForAutofill=no` 或 `autofill` 被关闭；或 App 在提交前清空了字段。
- 本应用不是当前选中的自动填充服务；或用户在系统界面多次选择“不保存”后系统可能限制。
这些限制在“设置 → 自动填充”页中说明，并提供“手动添加”入口（§5.5）。

### 5.3 onSaveRequest → 新建 / 更新判断
```
onSaveRequest(request, callback)
 ├─ 解析 request.fillContexts 中最后（及 DELAY_SAVE 之前的）结构：
 │    请求方身份（包名+证书 或 已验证的 webDomain，规则同 §2）、用户名、密码、旧/新密码、clientState
 ├─ 校验 neverSave*；仅可信浏览器 / 通过 DAL 的 WebView 才能以网页域名保存
 ├─ 将捕获内容放入进程内存暂存（PendingSave，5 分钟过期，不落盘）
 └─ callback.onSuccess(SaveConfirmActivity 的 IntentSender)   // minSdk 28，无需旧版通知回退
SaveConfirmActivity（原生 Kotlin，数据经 UniFFI 从 pm-core 取得，PendingSave 保存在 Rust 内存）
 ├─ 要求解锁（BiometricPrompt / 主密码）——比较必须用解密后的保险库
 ├─ pm-core compare(identity, username, password, oldPassword?) → new / new-account / changed / changed-pw-form / ambiguous / same
 └─ 显示：
     new            → [保存]（预填标题=应用名或站点名，规则=包名+证书 或 URI match=domain）
     new-account    → [保存为新账号] [更新已有 ▼]
     changed        → [更新 alice 的密码] [另存为新条目]
     changed-pw-form→ [更新]（以旧密码或用户名定位条目）
     ambiguous      → 下拉选择要更新的条目 / 新建
     same           → 自动关闭，Toast“已是最新”
     所有情况均有 [此应用/此站点从不] [忽略]
```
- 更新时旧密码写入 `passwordHistory`；若条目缺少当前包名/域名则追加。
- 写入本地缓存副本并通过后台任务按 06 §7 写回同步文件。

### 5.4 兼容模式下的限制
- 兼容模式由框架基于无障碍事件构建结构，**提交检测不可靠**：浏览器不会调用 `AutofillManager.commit()`，通常只能依赖 `SAVE_FLAG_TRIGGER_ON_ALL_VIEWS_INVISIBLE`（页面跳转导致字段消失）触发，SPA 中可能完全不触发。
- 无 `HtmlInfo`，字段分类只能依赖 `inputType` 与提示文字，改密/注册表单识别准确率低。
- 某些浏览器在兼容模式下密码框值可能不可得或已被掩码；若取得的值全为掩码字符（`•`、`*`），丢弃本次保存，提示用户使用“手动添加”。
- 保存时的域名以框架从地址栏解析的 `webDomain` 为准；地址栏正在编辑或显示搜索词时域名无效 → 不保存网页 URI，只允许用户手动填写。
- 结论：在兼容模式浏览器中，“保存”为尽力而为；UI 中标注“此浏览器为兼容模式，保存可能不会自动提示”。

### 5.5 手动添加
- 应用内“添加”及通知栏快捷方式“保存刚才的登录…”（若有 PendingSave 则预填）。
- 自动填充下拉中的“搜索保险库…”数据集选择条目后，可选“将此应用/网站关联到该条目”（解决只差匹配规则的场景）。

## 6. 可选 AccessibilityService 回退（默认关闭）
- 优点：可覆盖不支持自动填充的旧 App/浏览器。缺点：权限极大、易被滥用、厂商限制多、商店审核严格。
- 仅作为高级选项，开启时显示风险说明；不用于读取任何非当前焦点窗口内容。

## 7. 测试
- 单元：URL 匹配、save/update 判定等由 pm-core 的 `cargo test` 覆盖（共享向量）；Kotlin 单元测试覆盖 AssistStructure 解析、DAL 解析、TrustedBrowsers 签名校验；Qt Quick Test 覆盖 QML 动态码卡片；Espresso 覆盖 SaveConfirmActivity / PickActivity。
- 保存流程：测试 App 覆盖 finish 触发、`setTriggerId`、全部不可见触发、DELAY_SAVE 多屏、改密表单、“从不”按钮；浏览器覆盖原生模式与兼容模式。
- 仪器测试：使用测试 App + WebView 页面；手工矩阵：Chrome、Firefox、Samsung Internet、Edge、Brave × Android 9/11/14/15/16 × 原生/兼容模式。

## 8. 动态码（TOTP）
> 通用设计见 08。
### 8.1 自动填充 OTP 字段
- 识别信号：
  - `autofillHints` 含 androidx `HintConstants.AUTOFILL_HINT_2FA_APP_OTP`（`"2faAppOTPCode"`）、`AUTOFILL_HINT_SMS_OTP`（`"smsOTPCode"`）、`AUTOFILL_HINT_EMAIL_OTP`，或 App 自定义的 `"otp"`/`"oneTimeCode"` 等字符串；Android 框架 `View` 类本身没有名为 `AUTOFILL_HINT_ONE_TIME_CODE` 的常量，因此以 androidx 常量与浏览器 `HtmlInfo` 为准。
  - 浏览器网页：`HtmlInfo` 属性 `autocomplete="one-time-code"`、`inputmode="numeric"` + `maxlength` 6/8、name/id 含 otp/totp/2fa/mfa/code/动态码。
  - 兼容模式无 `HtmlInfo`，只能依赖 `inputType=number` 与提示文字，准确率低。
- 匹配：按与密码相同的请求方识别（包名+证书 / 可信浏览器 webDomain，§2）在**动态码库**中按动态码项自带的 `match` 规则（包名+证书 / URI）匹配动态码项。SMS/邮件验证码字段（`smsOTPCode`/`emailOTPCode`）若该站点没有匹配的动态码项，不提供数据集，避免干扰系统短信验证码填充。
- 数据集：动态码有时效，因此 OTP 数据集**总是**带 `authentication`，在 AuthActivity 中**解锁动态码库**（独立生物识别密钥或动态码库密码，与密码保险库分开）后才计算当前码（剩余 < 3 秒时用下一个码）并返回；数据集标签显示“GitHub 动态码”而非码值。
- 分格输入（6 个单字符框）：若识别到同一容器内 6/8 个 `maxLength=1` 的相邻字段，在一个 Dataset 中为每个字段分别设置对应位数字。
- 可选：密码填充完成后发一条可点按的通知“复制 GitHub 动态码”（默认关），用于不支持 OTP 字段识别的 App。

### 8.2 显示与扫码
- 动态码页（QML）：底部导航「密码 / 动态码 / 设置」中的「动态码」页，按 08 §11 的界面规格与 §11.11 的 QML 组件实现——顶栏（自有盾牌图标、“动态码”、🔍、≡）、卡片（图标、“发行方: 账号”、⋮、大号分组码、⧉、✦“复制并返回上一个应用”、“N秒”+圆形进度、≤5 秒预显示下一个码）、右下角绿色 ＋ FAB 速拨（扫描二维码 / 从相册导入 / 粘贴链接 / 手动输入 / 从 Google 身份验证器导入）、搜索（拼音首字母）、左滑置顶、拖动排序、空状态；动态码库锁定时只显示锁卡片。进入页面时经 pm-qt → JNI → `SecureWindow.kt` 为 QtActivity 窗口设置 `FLAG_SECURE`（全局设置可改为整个应用始终开启），应用进入后台（`Qt.application.state !== Qt.ApplicationActive`）时隐藏码值。
- 主屏小组件/快捷设置磁贴：后续版本（考虑锁屏可见性风险，默认不提供）。
- 扫码页（QML）：Qt Multimedia `CaptureSession` + `Camera` + `VideoOutput` 预览，C++ `QVideoSink` 取帧 → pm-core `rxing` 解码（不依赖 Google Play 服务；权限经 `QPermission`/`QCameraPermission` 申请）；运行时申请 `CAMERA` 权限，退出即释放；也支持 Photo Picker 选图（无需存储权限）。支持 `otpauth-migration` 多批次连续扫描。

### 8.3 剪贴板与时间
- Android 13+：复制时为 `ClipDescription` 设置 `EXTRA_IS_SENSITIVE=true`（系统预览中隐藏内容）；过期或 30 秒（较早者）后清除剪贴板（应用在前台时清除；后台时依赖系统剪贴板自动过期与下次回到前台清除）。
- 读取 `Settings.Global.AUTO_TIME`，未开启自动时间时在动态码页面告警。
