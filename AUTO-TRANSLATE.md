# 自动翻译（自建构建说明）

这个仓库是 [permissionlesstech/bitchat](https://github.com/permissionlesstech/bitchat) 的副本，
**唯一改动是加了「机内自动翻译」**：别人发来的外语消息会在原文下面显示译文，默认翻成 **繁體中文**，
目标语言可在设置里换（14 种）。

## 改了什么

| 文件 | 说明 |
| --- | --- |
| `bitchat/Features/translation/AutoTranslate.swift` | 新增。语言枚举、翻译管线、状态诊断、设置界面、消息行下方的译文视图 |
| `bitchat/Views/Components/TextMessageView.swift` | +10 行：在非本人发送的消息下方插入 `AutoTranslatedMessageView` |
| `bitchat/Views/AppInfoView.swift` | +6 行：设置页（点左上角 `bitchat/` → settings）插入 `AutoTranslateSettingsSection` |
| `.github/workflows/build-ipa.yml` | 新增。CI 打未签名 IPA |

工程使用的是 Xcode 的 file-system synchronized group（`PBXFileSystemSynchronizedRootGroup`），
新增 Swift 文件会自动进入 target，**不需要改 `project.pbxproj`**。

## 工作原理

1. 消息行出现时调用 `AutoTranslateStore.request(messageID:text:)`；
2. 用 `NLLanguageRecognizer` 判断语言，已经是目标语言 / 命令（`/`开头）/ 无字母 → 跳过；
3. 用 `LanguageAvailability` 查这一对的可用性：
   - `已安装` → 用无界面 `TranslationSession(installedSource:target:)`（iOS 26+）直接翻译；
   - `需下载` → 消息下方显示 `⚠︎ 需下载语言包（fr→zh-Hant）`，并记下待下载的语言；
   - `不支持` → 显示原因，繁體中文不支持时自动退回简体中文重试；
4. 译文缓存按消息 ID 保存，切目标语言或关闭开关会清空缓存；
5. 全部在本机执行，不上传任何文本（Apple `Translation.framework`）。

## 设置入口

`bitchat/` → **settings** → **AUTO-TRANSLATE**：
- `translate incoming messages` 开关
- `target language` 目标语言（默认 繁體中文 / zh-Hant）
- 状态区：已翻译/排队/失败条数、最近错误、每种语言对的可用性
- 按钮：**下载语言包**（走系统授权下载模型）、**测试翻译**、**刷新**

第一次遇到某语言，需要下载对应语言模型（约几十 MB，系统弹窗确认），之后离线可用。

## 构建

### GitHub Actions（推荐）
仓库已带 `.github/workflows/build-ipa.yml`：
Actions → ios-ipa → Run workflow，选 runner（`xcode-27` 或 `macos-26`），
产物在 run 的 Artifacts 里：`ios_outputs_<label>`（未签名 IPA）和 `buildlog_<label>`（编译日志）。

注意：仓库 push 到 `main` 时也会自动构建（`paths-ignore: **.md`）。

### 本地
```bash
xcodebuild -project bitchat.xcodeproj -scheme "bitchat (iOS)" \
  -configuration Release -sdk iphoneos \
  -destination 'generic/platform=iOS' \
  CODE_SIGNING_ALLOWED=NO build
```
然后用 `Payload/` 方式打包成 IPA，用 iloader 等工具签名安装。

## 已知限制
- 只翻译**别人发来的**消息；自己发的不翻译；已是目标语言的消息跳过。
- Apple 机内翻译的语言对有限，冷门语种不会出译文（会显示原因）。
- 侧载缺 App Group 权限：分享扩展可能不可用，主体聊天 / 蓝牙 mesh / Nostr 正常。
- 推送（APNs）在侧载环境下不可用。
