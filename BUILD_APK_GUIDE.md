# 足迹 Footprint —— 新手发布 APK 完整指南

> 本文档专门写给“**刚 fork 了这个项目、电脑上没装任何开发环境**”的新手。
> 跟着做，**不需要在自己电脑安装 Android Studio / Flutter / Rust**，全程用 GitHub 的免费云端（GitHub Actions）打包。
> 配套说明：本指南由 Codex 助手根据项目源码与作者文档整理，同时已帮你把需要的文件改好（见第 2 节）。

---

## 0. 先花 2 分钟了解：这个项目为什么“不能直接打包”

作者 README 里只写了“装 JDK、Android SDK、申请高德 Key”，但**实际编译还需要另外两样东西**，新手照 README 做会在半路卡住：

| 隐藏依赖 | 作用 | 说明 |
| --- | --- | --- |
| **Flutter SDK** | App 的主界面其实是个 Flutter 模块（flutter_ui） | 仓库把 flutter_ui/.android/ 加了 gitignore，需要先用 Flutter 工具生成它，Gradle 才能通过 |
| **Rust + cargo-ndk** | 编译一个原生渲染库 libfootprint_renderer.so（app/rust） | 每次构建前会自动执行 cargo ndk ...，没有 Rust 环境会直接报错 |
| **高德地图 Key** | 地图功能必须 | Key 和“包名 + 签名证书 SHA1”绑定，所以**必须先有稳定的签名文件**，Key 才有效 |

另外还有两个“坑”我已经帮你处理/提示：
1. gradle.properties 里写死了作者电脑的 JDK 路径（E:/DevTools/...），其它电脑和云端会因此构建失败 → **已注释掉**。
2. Release 版本默认**没有签名配置**，直接打包出来的是“未签名包”，装不到手机上 → **已加上可选签名支持**。

> 手机要求：Android 7.0 及以上，且是 **64 位（arm64）** 手机（2017 年后的主流手机基本都是）。

---

## 1. 你只需要记住 5 件事

1. 把改好的文件放进你的 GitHub fork（第 3.1 节）。
2. 在 GitHub 上运行一次“生成 Release 签名文件”，下载签名文件、记下 SHA1（第 3.3 节）。
3. 把签名文件等信息填进仓库 Secrets（第 3.4 节）。
4. 去高德开放平台申请一个 Key（第 3.5 节）。
5. 运行“Android Build (打包 APK)”，下载 APK 安装到手机（第 3.6、3.7 节）。

---

## 2. 我已经帮你改好 / 新增的文件

以下文件都在本目录：C:\Users\Johnyu\Documents\Code_Projects\Footprint-master

| 文件 | 状态 | 作用 |
| --- | --- | --- |
| .github/workflows/android_build.yml | **重写** | 云端打包主流程：自动装 JDK/Flutter/Rust/NDK → 生成 .android → 构建 Debug+Release → 上传 APK |
| .github/workflows/generate_keystore.yml | **新增** | 一键生成 Release 签名文件并显示 SHA1（不用本地装 Java） |
| gradle.properties | **修改** | 注释掉写死的作者 JDK 路径，避免其它电脑/云端构建失败 |
| app/build.gradle.kts | **修改** | 增加“可选 Release 签名”：根目录存在 keystore.properties 时自动签名 |
| keystore.properties.example | **新增** | 签名配置模板（复制改名后用） |
| BUILD_APK_GUIDE.md | **新增** | 就是本文件 |

---

## 3. 路线 A：GitHub 云端打包（推荐，全程不用装环境）

### 3.1 把改动放进你的 fork

你的 fork 在 GitHub 上，但本地这个文件夹是从 GitHub 下载的压缩包（没有 .git），所以推荐用 **GitHub Desktop** 重新克隆一次再覆盖文件：

**方式 1：GitHub Desktop（推荐，图形界面，不敲命令）**
1. 到 https://desktop.github.com 下载安装 GitHub Desktop，登录你的 GitHub 账号。
2. 菜单 File → Clone repository… → 选你的仓库（例如 你的用户名/Footprint）→ 选一个存放位置，例如 C:\Users\Johnyu\Documents\Code_Projects\Footprint-fork → Clone。
3. 把下面 6 个文件，从 Footprint-master 复制过去**覆盖**到克隆目录的相同位置：
   - .github/workflows/android_build.yml
   - .github/workflows/generate_keystore.yml
   - gradle.properties
   - app/build.gradle.kts
   - keystore.properties.example
   - BUILD_APK_GUIDE.md
4. 回到 GitHub Desktop，左下角填“提交信息”（例如 配置云端打包），点 Commit to master，再点 Push origin。

**方式 2：直接在 GitHub 网页上改（不装软件，但粘贴较多）**
- 到你的 fork，逐个打开上述文件，点右上角 ✏️ 编辑，把本地文件内容全部粘贴进去后 Commit。新增文件用 Add file → Create new file（注意 .github/workflows/ 目录要建对）。

> 小提示：无论哪种方式，改完后**去仓库页面确认这几个文件内容已经是新版**再继续。

### 3.2 开启 Actions

1. 打开你的 fork 仓库 → Settings → Actions → General。
2. 在 Actions permissions 里选 Allow all actions and reusable workflows → Save。
   （公开仓库一般默认已允许；如果看不到设置，说明已允许，跳过即可。）

### 3.3 第一步：生成“签名文件”（只做一次，相当于应用的身份证）

1. 打开仓库 Actions 页 → 左侧点“生成 Release 签名文件 (keystore)” → 右边点 Run workflow。
2. 填写参数（可保持默认，也可以自己改）：
   - **密钥库密码**：请改成你自己记得住的密码（建议只用字母和数字），这个密码**以后每次升级都要用**；
   - Key 别名、姓名：保持默认即可。
3. 点绿色 Run workflow，等 1 分钟左右跑完。
4. 打开这次运行的详情页：
   - 在 Artifacts 区下载 release-keystore，解压得到 **release-keystore.jks** —— ⚠️ **务必保存好，这是你应用的“身份证”**；
   - 在日志（显示 SHA1 那一步）里复制 SHA1: XX:XX:XX:… 那一行，先粘贴到记事本备用（申请高德 Key 用）。

### 3.4 第二步：把签名信息告诉 GitHub（配置 Secrets，只做一次）

1. **把 .jks 转成一长串文本**：在 Windows 打开 PowerShell，先进入你下载解压 release-keystore.jks 的文件夹，然后执行：

   ```powershell
   [IO.File]::WriteAllText("$env:USERPROFILE\Desktop\keystore_base64.txt", [Convert]::ToBase64String([IO.File]::ReadAllBytes("$PWD\release-keystore.jks")))
   ```

   执行后，**桌面上会出现 keystore_base64.txt**，打开它，里面是**一整行很长的字母数字**，全部复制。
2. 打开你的 fork 仓库 → Settings → Secrets and variables → Actions → New repository secret，逐个添加下面 5 个（名字必须完全一致）：

| Secret 名字 | 填什么 |
| --- | --- |
| KEYSTORE_BASE64 | 刚才那整行超长文本 |
| KEYSTORE_PASSWORD | 你在 3.3 填的“密钥库密码” |
| KEY_ALIAS | Key 别名（默认 footprint） |
| KEY_PASSWORD | Key 密码（与密钥库密码相同即可） |
| AMAP_KEY | （可选，建议后面加）高德 Key，见 3.5 |

### 3.5 第三步：申请高德地图 Key（地图要显示必须做）

1. 打开高德开放平台：https://console.amap.com （注册/登录）。
2. 应用管理 → 我的应用 → 创建新应用（名称随意）。
3. 在应用下点 添加 Key：
   - 服务平台：选 Android 平台；
   - **发布版安全码 SHA1**：粘贴 3.3 记下的那行 SHA1；
   - PackageName：填 com.footprint（不能错）。
4. 提交后会得到一个 Key（一串字符）。
5. 把它加到仓库 Secret：名字 AMAP_KEY，值填这串 Key。
   > 不填也可以：装好后打开 App 地图页右上角 ⚙️ 设置，手动输入 Key 并重启。但手动输入同样要求这个 Key 绑定的 SHA1/包名和你这个签名一致。

### 3.6 第四步：云端打包

1. 打开仓库 Actions 页 → 左侧点“Android Build (打包 APK)” → Run workflow（以后每次 push 代码也会自动触发）。
2. 等待构建完成。**第一次较久（10~25 分钟）**，因为要在云端现装 Flutter、Rust 并下载依赖；之后每次会快一些。
3. 跑完后，在本次运行页面最下方的 Artifacts 里下载：
   - app-release → 里面的 app-release.apk 是正式包（已用你的签名，可直接安装、可分享）；
   - app-debug → app-debug.apk 是调试包，用来先试试能不能装。

### 3.7 第五步：安装到手机

1. 把 app-release.apk 传到手机（微信/QQ 文件传输助手、网盘、数据线都行）。
2. 在手机上点击 APK 安装。若提示“禁止安装未知来源应用”，按提示允许（或去设置里打开“安装未知应用”）。
3. 若提示“与已安装应用签名不一致 / 应用未安装”：说明手机上装过别的签名版本，先卸载旧版本再装。
4. 打开 App 若地图空白：确认高德后台 SHA1/包名与你的签名一致；或在 App 地图页右上角 ⚙️ 设置里输入高德 Key 后重启。

**以后每次更新**：改完代码 push 到 master → Actions 自动打包 → 下载新 APK。**升级安装时手机上的旧版必须是你用同一个 keystore 签的**，否则装不上。

---

## 4. 路线 B：以后想在自己电脑上开发 / 打包（可选）

等你以后想自己改代码、在本地跑，再装环境。**本地打包不是发布所必需**，这里只给清单，不做展开：

1. 安装 Android Studio（自带 JDK 17）；SDK Manager 里装 Android SDK Platform 35 和 NDK。
2. 安装 Flutter SDK（stable），并把 flutter 加入 PATH。
3. 安装 Rust（https://rustup.rs），然后执行：

   ```bash
   rustup target add aarch64-linux-android
   cargo install cargo-ndk
   ```

4. 打开项目前，先到 flutter_ui 目录执行 flutter pub get（生成被 gitignore 的 .android 目录）。
5. 用 Android Studio 打开项目，会自动生成 local.properties（指向你的 SDK）；确保系统 JAVA_HOME 指向 JDK 17（gradle.properties 已不再写死路径）。
6. 想本地出正式包：把 release-keystore.jks 和 keystore.properties 放到项目根目录，然后运行：

   ```bash
   ./gradlew assembleRelease
   ```

   APK 输出在 build\app_build_fresh\outputs\apk\release\。

---

## 5. 常见问题（FAQ）

**Q：Actions 页面看不到“Android Build”工作流？**
先确认你已经把新版 workflow 文件 push 到了**默认分支**（master），并刷新页面；再检查 3.2 的 Actions 开关。也可以直接点 Run workflow 手动触发。

**Q：构建失败，日志里有 rust / cargo-ndk 相关报错？**
一般是云端安装 Rust 工具时网络抖动，直接**重新 Run workflow** 一次；如果反复失败，把日志里红色报错段落发给我看。

**Q：构建失败，报 Flutter/Dart 编译错误？**
作者用的 Flutter 版本较老，云端默认装的是最新 stable。若新版 Flutter 不兼容，可以把 .github/workflows/android_build.yml 里 subosito/flutter-action 的 channel: stable 改成作者当时用的具体版本号（先告诉我，我帮你查），或贴报错给我。

**Q：下载的 APK 装不上 / 提示未签名？**
- app-release-unsigned.apk 是没配 Secrets 时产生的**未签名包，装不了**，请用 app-debug.apk，或按 3.3/3.4 配置好签名后重新打包用 app-release.apk。
- 手机上装过旧版：先卸载再装。

**Q：App 能装，但地图空白/提示 Key 错误？**
高德 Key 与“包名 + 签名 SHA1”绑定。确认：① 高德后台填的 SHA1 必须来自**给你这个 APK 签名的那个 keystore**（release 包用 release keystore 的 SHA1，debug 包用 debug keystore 的 SHA1，两者不一样）；② 包名是 com.footprint；③ 也可以在 App 设置里手动填 Key。

**Q：想改应用名字 / 图标 / 版本号？**
- 应用名：app/src/main/res/values/strings.xml 里的 app_name。
- 版本号：app/build.gradle.kts 里 versionCode（数字，每次升级 +1）和 versionName（给人看的版本号）。
- 图标：替换 app/src/main/res/mipmap-* 里的图标文件（项目根目录有 update_icons.py 和 Footprint.png 可参考）。

**Q：这个 APK 能上架应用商店吗？**
技术上是“已签名的正式包”，可以侧载分发（发朋友、发网盘、发群）。国内应用商店上架还需要软著/备案/隐私政策等资质材料，那是另一回事。

---

## 6. 安全提醒（重要）

1. **release-keystore.jks 和密码务必备份多处**。丢了 = 以后无法给已安装用户升级（只能换包名重新上架，用户数据不通用）。
2. **不要把 .jks 和 keystore.properties 提交到公开仓库**。本项目的 .gitignore 已忽略 *.jks/*.keystore/*.p12，正常 push 不会带上去。
3. 高德 Key 有免费调用额度，个人使用足够；请勿把 Key 明文写进会被公开的代码里（本项目已用 Secrets / local.properties 方式处理）。

---

*祝你打包顺利！卡在哪一步，把 GitHub Actions 日志里红色的报错复制给我，我帮你继续排查。*
