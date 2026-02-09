# Bypass Paywalls Clean

## BPC 介绍

该扩展可让你阅读已实现付费墙的（受支持的）网站文章。

你也可以将域名添加为自定义站点并尝试绕过付费墙。  

> 注意：项目目录下每个 crx 文件，都可以手动更改后缀名为 .zip，然后可以打开里面有针对该插件更详细的说明文档

## BPC（Chrome 版）

### 安装

- 该扩展不在 Google Chrome 应用商店提供。
- 在基于 [Chromium](https://en.wikipedia.org/wiki/Chromium_(web_browser)) 的桌面浏览器中安装第三方扩展，需要按照以下说明操作。
- 在扩展开发者模式下，你可以通过 *Load unpacked* 安装 BPC（最新 master，但无自动更新），或通过 crx 文件安装（最新发布版且自动更新，但可能需要把扩展加入允许列表）。
- 你可以通过工具栏扩展菜单（拼图形状图标）将扩展图标添加/固定到工具栏。

#### Load unpacked：Chrome、MS Edge 或 Brave（桌面版）

- 或将扩展加入允许列表并安装可自动更新的 crx 文件（见下一节）

1. 从 [GitFlic](https://gitflic.ru/project/magnolia1234/bpc_uploads/blob/raw?file=bypass-paywalls-chrome-clean-master.zip) 下载该仓库的 zip 文件。
2. 解压后应得到名为 *bypass-paywalls-chrome-clean-master* 的文件夹。
3. 将该文件夹移动到电脑上的永久位置（安装后不要删除该文件夹）。
4. 打开扩展页面：*chrome://extensions*
5. 启用开发者模式。
6. 点击 *Load unpacked* 并选择/打开扩展文件夹（包含 manifest.json）。

- 默认情况下 BPC 只有有限的主机权限，但你可以选择启用自定义站点（并可为未列出的站点清理 Cookie/阻止通用付费墙脚本）。你也可以只为自己添加的自定义站点申请主机权限（或点击 *clear cookies*（BPC 图标）请求当前站点的主机权限）。

#### CRX 文件：其他 Chromium 浏览器（Opera/Vivaldi/Yandex）

- 或为 Chrome、MS Edge 或 Brave 将扩展加入允许列表（见说明 [本地版](https://github.com/bpc-clone/bypass-paywalls-chrome-clean/blob/master/allowlist/README.html) 或 [在线版](https://github.com/bpc-clone/bypass-paywalls-chrome-clean/blob/master/allowlist/README.md)）

1. 从 [GitFlic](https://gitflic.ru/project/magnolia1234/bpc_uploads) 下载扩展的 crx 文件（右键 > 另存为）。
2. 在浏览器中打开扩展页面：*chrome://extensions*
3. 启用开发者模式。
4. 将 crx 文件拖到页面任意位置以导入（若已有正在使用的 “load unpacked” 安装，请先移除并备份自定义站点；以便自动更新）。

- 默认情况下 BPC 只有有限权限，但你可以选择启用自定义站点（并可为未列出的站点清理 Cookie/阻止通用付费墙脚本）。你也可以只为自己添加的自定义站点申请主机权限（或点击 *clear cookies*（BPC 图标）请求当前站点的主机权限）。

### 更新

对于 crx 安装：扩展会自动更新，或可在 chrome://extensions 中手动检查更新。
当扩展为新增域名需要新的主机权限时（Chrome/Edge），扩展可能会被禁用：对自定义站点进行一次启用/禁用操作即可消除该“错误”（浏览器会记住所授予的主机权限）。

对于 zip 安装（load unpacked/开发者模式）：将文件解压到安装文件夹。

你也可以在启动时检查（发布后）站点规则更新（需选择启用）；仅在修复版发布后约 10 天内可用。
对于新增（更新）的站点，你也需要启用自定义站点/为新域名请求主机权限（或等待新的发布版本）。

### 安卓

1. 从 Google Play 商店安装 [Kiwi Browser](https://play.google.com/store/apps/details?id=com.kiwibrowser.browser&hl=nl)。
2. 你有两个选择：

- 从 [GitFlic](https://gitflic.ru/project/magnolia1234/bpc_uploads) 加载 CRX 文件（自动更新，不需要允许列表；自定义站点的选择启用不可用（使用 kiwi-custom crx；会更新到最新正式版））
- 安装最新 master zip 文件（无自动更新；自定义站点使用 custom 文件夹中的 manifest.json）。
  当安装超时（未安装；Android 问题）时，你可能还需要重新打包 zip 文件，确保其中不包含顶层文件夹（如 GitHub 默认打包方式）。

### 解决新版本 Chrome 无法使用问题

Chrome 添加插件时报错通常是因为浏览器版本较新，而插件仍使用 Manifest V2。Chrome 正在逐步停止支持 V2 并强制要求 V3，因此会出现安装或启用失败。

####  **问题说明**

| 版本               | V2 支持状态    | 解决方案                           |
| :----------------- | :------------- | :--------------------------------- |
| **< Chrome 117**   | 完全支持       | 使用实验性标志                     |
| **Chrome 117-118** | 有限支持       | `Temporarily Unavailable` 标志     |
| **Chrome 119-126** | 企业策略支持   | 配置组策略/注册表                  |
| **Chrome 127+**    | **完全不支持** | **必须升级到 V3** 或使用替代浏览器 |

####  **验证方法**

如果你想确认当前 Chrome 是否支持 V2：

1. 访问 `chrome://version/`
2. 查看你的版本号
3. 对照上表

------

####  **当前唯一可行方案**

1. **升级插件到 Manifest V3**（需要开发者操作）

2. **使用其他浏览器**（Edge、Firefox、Brave 等）

   - Edge 浏览器目前仍可通过标志支持 V2

3. **降级到 Chrome 126 或更早版本**（不推荐，安全风险）

   

详细方案（优先推荐修改并加载本地文件夹）：

**方法一：解包并修改 manifest 后以“Load unpacked”加载（推荐）**

1. 准备扩展文件：
- 如果是 `.crx` 文件：将后缀改为 `.zip` 并解压。
  - ​	实际是，将文件 `bypass-paywalls-chrome-clean-3.8.7.0.crx` 使用压缩工具解压成 `bypass-paywalls-chrome-clean-3.8.7.0.zip`
2. 找到解压目录中的 `manifest.json`，此时。
3. 找到解压目录中的 `manifest/mv3/manifest.json`，这个就是 `V3` 版本，复制并覆盖掉根目录下的 `manifest.json` 文件（它是 `Manifest V2` 版本），即可。
4. 打开 Chrome 扩展管理页：`chrome://extensions`
5. 打开“开发者模式”。
6. 点击 “Load unpacked”，选择解压后的扩展文件夹（包含 `manifest.json`）。

如果加载后仍报错，请查看扩展详情页的错误信息，按提示进一步调整。

**方法二：改用其他浏览器**

- 可尝试 Firefox 或 Edge（Edge 有时对旧扩展兼容性更好）。

**方法三：降级 Chrome（不推荐）**

- 可下载旧版本 Chrome 并关闭自动更新，但存在安全风险，不建议长期使用。

## 添加自定义站点

以下步骤用于添加自定义站点，尝试绕过付费墙：

1. 点击浏览器工具栏的 BPC 图标，进入弹窗后点 `Options`。
2. 在 `Options` 页面进入 `Custom sites`。
3. 添加自定义站点或站点组（站点组用逗号分隔；组名建议用 `group_...` 形式）。
4. 启用自定义站点权限：在 `Opt-in` 页面启用自定义站点，或只为你添加的域名单独授予主机权限。
说明：也可以在目标站点打开后，点 BPC 图标里的 `clear cookies` 来触发当前站点的权限请求。
5. 保存后刷新目标站点测试。
说明：自定义站点默认会阻止/清理 cookies 和本地存储，适合绕过“计量式付费墙”；若站点需要登录，可在自定义站点里开启 `allow_cookies`。

常用可选项（按需启用）：

1. 设置 `useragent`（Googlebot/Bingbot/Facebookbot/自定义）。
2. 设置 `referer`（Facebook/Google/Twitter/自定义）。
3. 禁用 JavaScript（站点/外部脚本/内联）。
4. AMP 相关（跳转 AMP 或显示 AMP 内容）。
5. JSON 抽取、archive/webcache 辅助、正则阻断脚本等。

## 资源地址

https://gitflic.ru/project/magnolia1234/bpc_uploads
