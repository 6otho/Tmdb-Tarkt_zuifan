# 📺 Cloudflare Worker - TMDB & Trakt 追番面板 (最终定制版 V8)

这是一个基于 Cloudflare Workers 的 Serverless 追番管理面板。它结合了 **TMDB** (The Movie Database) 的详细数据和 **Trakt.tv** 的观看进度同步功能，提供了一个美观、丝滑的移动端 Web App。

## ✨ 主要功能

- **无需服务器**：完全运行在 Cloudflare Edge 网络上，免费且速度极快。
- **双向同步**：
  - 查看 Trakt 待看列表 (Watchlist)。
  - 查看 Trakt 播放历史 (History)。
  - 查看 Trakt 个人日历 (Calendar - 剧集更新提醒)。
  - 支持 TMDB 收藏夹同步。
- **交互操作**：直接在页面上“加入/取消追番”或“标记为已看”，数据实时同步回 Trakt。
- **智能缓存**：使用 Cloudflare KV 缓存请求，加快加载速度并防止 API 超限。
- **精美 UI**：类似 iOS 原生 App 的设计，支持 PWA（添加到主屏幕）。

---

## 🛠️ 准备工作

在开始之前，您需要准备好以下三个平台的账号：

1. **Cloudflare**: [https://dash.cloudflare.com/](https://dash.cloudflare.com/) (用于部署代码)
2. **TMDB**: [https://www.themoviedb.org/](https://www.themoviedb.org/) (用于获取海报和元数据)
3. **Trakt**: [https://trakt.tv/](https://trakt.tv/) (用于同步观看进度)

---

## 🚀 详细部署教程

### 第一步：获取 TMDB API Token

1. 登录 [TMDB 官网](https://www.themoviedb.org/)。
2. 点击头像 -> **Settings (设置)** -> 左侧菜单 **API**。
3. 点击 **Create (创建)** -> 选择 **Developer (开发者)**。
4. 填写必要信息（URL 可以随便填，例如 `http://localhost`），提交申请。
5. 申请成功后，在 API 页面找到 **API Read Access Token (API 读访问令牌)**。
   > ⚠️ **注意**：我们需要那个很长的 Token，不是短的 API Key。请记下这个 Token，稍后环境变量名为 `TMDB_TOKEN`。

---

### 第二步：创建 Trakt API 应用

1. 登录 [Trakt 官网](https://trakt.tv/)。
2. 进入 [Trakt API App 创建页面](https://trakt.tv/oauth/apps)。
3. 点击 **NEW APPLICATION**。
4. 填写信息：
   - **Name**: 随意（例如 `MyCloudflareTracker`）。
   - **Redirect URI**: ⚠️ **必须填写** `urn:ietf:wg:oauth:2.0:oob`
   - **Javascript (CORS) origins**: 留空或填写 `*`。
5. 保存应用 (Save App)。
6. 保存成功后，你会看到 `Client ID` 和 `Client Secret`。
   > ⚠️ **注意**：请记下这两个值，稍后环境变量名为 `TRAKT_ID` 和 `TRAKT_SECRET`。

---

### 第三步：获取 Trakt 初始 Refresh Token (最关键一步)

由于 Worker 是无服务器环境，我们需要手动进行第一次授权来获取一个能够“自我刷新”的 Token。我们使用在线工具来完成，非常简单。

#### 3.1 获取授权码 (PIN Code)
将下方链接中的 `您的_CLIENT_ID` 替换为你上一步获取的 `Client ID`，然后在浏览器访问：

```text
https://trakt.tv/oauth/authorize?response_type=code&client_id=您的_CLIENT_ID&redirect_uri=urn:ietf:wg:oauth:2.0:oob
点击 Yes 授权，你会在网页上看到一串 8位数的 PIN 码。
3.2 在线换取 Refresh Token
打开在线工具网站：https://reqbin.com/curl
复制下面这段代码，粘贴到网站左边的框里：
code
Bash
curl -X POST https://api.trakt.tv/oauth/token \
-H "Content-Type: application/json" \
-d '{
  "code": "这里填_8位数_PIN_码",
  "client_id": "这里填_CLIENT_ID",
  "client_secret": "这里填_CLIENT_SECRET",
  "redirect_uri": "urn:ietf:wg:oauth:2.0:oob",
  "grant_type": "authorization_code"
}'
修改内容：把代码里中文标注的 PIN码、CLIENT_ID 和 CLIENT_SECRET 换成你自己的。
点击 Run 按钮。
右边窗口会出现一串 JSON 数据，找到 "refresh_token": "..." 这一行。
⚠️ 注意：复制那个很长的 refresh token（不要带引号），稍后环境变量名为 TRAKT_INIT_REFRESH。
第四步：Cloudflare Worker 部署
登录 Cloudflare Dashboard。
进入 Workers & Pages -> Create Application -> Create Worker。
命名你的 Worker（例如 my-trakt-app），点击 Deploy。
点击 Edit code。
清空 编辑器里原有的代码，将本项目提供的 worker.js (或者你文件里的代码) 全部粘贴 进去。
点击右上角 Save and Deploy。
第五步：绑定 KV 数据库 (用于缓存)
在 Cloudflare Dashboard 左侧菜单找到 Workers & Pages -> KV。
点击 Create a Namespace。
名称填写 TRAKT_CACHE (或者随意)，点击 Add。
回到你刚才创建的 Worker 页面 -> Settings (设置) -> Variables (变量)。
找到 KV Namespace Bindings 部分，点击 Add Binding。
Variable name: KV (⚠️必须必须是大写的 KV，这对应代码里的 env.KV)。
KV Namespace: 选择刚才创建的 TRAKT_CACHE。
点击 Save and Deploy。
第六步：配置环境变量
在 Worker 的 Settings (设置) -> Variables (变量) -> Environment Variables 部分，添加以下 4 个变量：
Variable Name (变量名)	Value (值)	说明
TRAKT_ID	您的_Client_ID	来自 Trakt 开发者后台
TRAKT_SECRET	您的_Client_Secret	来自 Trakt 开发者后台
TRAKT_INIT_REFRESH	您的_Refresh_Token	第三步中通过 ReqBin 获取的那串长字符
TMDB_TOKEN	您的_TMDB_Read_Token	来自 TMDB 设置页面的长 Token
点击 Save and Deploy。
🎉 大功告成！
现在，访问你的 Worker 网址（例如 https://my-trakt-app.您的用户名.workers.dev），你应该能看到完整的追番面板了！
首次加载可能会稍慢（因为没有缓存），之后浏览会非常快。
💡 使用指南
新番时刻表：显示当前正在播出的动画/剧集，按周几排列。
我的：
继续观看 (Calendar)：你正在追的剧，接下来要播出的集数。
播放记录 (History)：你已经在 Trakt 标记看过的记录。
我的追番 (Watchlist)：你添加了但在等待观看的列表。
TMDB 收藏：你在 TMDB 网站点的喜欢。
发现：查看热门趋势。
操作：点击任意海报进入详情，或点击海报右上角的 ... 呼出快捷菜单，可以进行“加入追番”或“标记已看”操作，数据会实时同步到 Trakt。
❓ 常见问题 (FAQ)
Q: 页面显示 "Config Error"？
A: 请检查环境变量（Step 6）是否填写正确，且变量名是否全大写，不能有空格。
Q: 页面显示 "Auth Failed"？
A: 你的 TRAKT_INIT_REFRESH 可能过期或错误。Trakt 的 Refresh Token 有效期较长，但如果失效，你需要重复 第三步 获取一个新的 Refresh Token 并更新到环境变量中。
Q: 图片加载不出来？
A: TMDB 的图片域名 image.tmdb.org 在某些网络环境下可能被阻断。Cloudflare Worker 是在服务器端获取数据的，但图片是客户端（你的浏览器）直接加载的。如果图片挂了，你需要自备科学上网环境。
Q: 如何强制刷新数据？
A: 在你的网址后面加上 /flush (例如 https://...workers.dev/flush) 访问一次，会清空所有缓存。
code
Code
