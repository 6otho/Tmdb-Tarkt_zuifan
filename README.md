
# 🎬 Cloudflare Worker 追番/观影列表 (KV 极速版)

本项目是一个基于 Cloudflare Worker 的个人影视追踪页面。
它集成了 **Reflix 伪装头**（获取高质量数据）和 **KV 智能缓存**（极速加载 + 防 API 限制）。

> **✨ 主要特性**
> *   **数据源**：Trakt (历史/进度) + TMDB (图片/中文元数据)
> *   **Reflix 伪装**：保留 Trakt VIP 数据效果。
> *   **智能缓存**：个人数据 1分钟刷新，公共数据 1小时刷新。
> *   **零成本**：完全基于 Cloudflare 免费额度。

---

## 🛠 前置准备

在开始之前，请确保你拥有以下账号，并打开一个记事本用于临时存放密钥：
1.  [Cloudflare](https://dash.cloudflare.com/)
2.  [TheMovieDB (TMDB)](https://www.themoviedb.org/)
3.  [Trakt.tv](https://trakt.tv/)

---

## 🟢 第一步：获取 TMDB Token (封面数据源)

我们需要 TMDB 的 API 令牌来获取海报和中文简介。

### 1. 注册与申请
1. 访问 [TMDB 官网](https://www.themoviedb.org/) 并登录。
2. 点击右上角头像 → **Settings (设置)**。
3. 左侧菜单点击 **API**。
4. 点击 **Create (创建)** 或 **Request an API Key**，选择 **Developer (开发者)**。
5. **填写申请表**（内容随意）：
   - *Type of use*: `Personal`
   - *URL*: `http://localhost`
   - *Summary*: `Personal watchlist project`
   - 补全地址信息后提交。

### 2. 复制关键 Token
在 API 页面下方找到 **API Read Access Token (v4 auth)**。

> ⚠️ **注意**：请复制那个 **以 `eyJ` 开头、特别长** 的字符串（不是短的 API Key）。

👉 **保存为：`TMDB_TOKEN`**

---

## 🔴 第二步：创建 Trakt 应用 (历史记录源)

我们需要创建一个“应用”来允许 Worker 读取你的观看历史。

### 1. 创建应用
1. 访问 [Trakt 开发者后台](https://trakt.tv/oauth/applications)。
2. 点击右上角绿色按钮 **NEW APPLICATION**。

### 2. 填写配置（⚡️ 极其重要）
请严格按照以下内容填写，否则后续无法验证：

- **Name**: `MyWorkerList` (随意)
- **Description**: `Personal list`
- **Redirect URI**: **必须填入下方代码，一个字都不能错** 👇
  ```text
  urn:ietf:wg:oauth:2.0:oob
  ```
- **Javascript (CORS) origins**: 留空。
- **Permissions**: 保持默认。

点击底部 **SAVE APP** 保存。

### 3. 保存 ID 和 Secret
保存后，页面上方会显示：
- **Client ID** 👉 **保存为：`TRAKT_CLIENT_ID`**
- **Client Secret** 👉 **保存为：`TRAKT_CLIENT_SECRET`** (下一步换 Token 用)

---

## 🔵 第三步：手动获取 Access Token (最关键一步)

由于 Trakt 需要 OAuth2 验证，我们需要用 **ReqBin** 在线工具手动生成一个永久令牌。

### 3.1 获取授权验证码 (Code)
1. 复制下方链接到浏览器地址栏（**先不要回车**）：
   ```text
   https://trakt.tv/oauth/authorize?response_type=code&client_id=你的Client_ID&redirect_uri=urn:ietf:wg:oauth:2.0:oob
   ```
2. 将链接中的 `你的Client_ID` 替换为 **第二步获取的 Client ID**。
3. 回车访问链接，点击绿色按钮 **Yes** 授权。
4. 页面显示一串 **8位数的验证码**，复制它。

### 3.2 使用 ReqBin 换取 Token
1. 打开在线工具：[https://reqbin.com/](https://reqbin.com/)
2. **配置请求面板**（请严格按照说明操作）：
   - **请求方式**: 下拉选择 **`POST`**。
   - **API 地址**: 填入 `https://api.trakt.tv/oauth/token`
   - **Headers**: 点击 Headers 标签，填入 `Content-Type: application/json`
3. **填写 JSON 内容**:
   - 点击 **Content** 标签，格式选择 **JSON**。
   - 复制下方代码，**替换为你自己的数据**：

   ```json
   {
       "code": "这里填刚才获取的8位验证码",
       "client_id": "这里填第二步的 Client ID",
       "client_secret": "这里填第二步的 Client Secret",
       "redirect_uri": "urn:ietf:wg:oauth:2.0:oob",
       "grant_type": "authorization_code"
   }
   ```
4. 点击 **Send**，在右侧返回结果中找到 `"access_token": "..."`。引号里的乱码就是我们要的。

👉 **保存为：`TRAKT_ACCESS_TOKEN`**

---

## 🗄️ 第四步：配置 Cloudflare KV 数据库

这一步是为了开启**智能缓存**，让网站秒开并减少 API 请求。

### 1. 创建命名空间
1. 登录 Cloudflare 控制台，进入 **Workers & Pages**。
2. 左侧菜单点击 **KV**。
3. 点击 **Create a Namespace**。
4. 输入名称：`ANIME_KV` (建议使用此名称以对应教程)。
5. 点击 **Add**。

### 2. 绑定到 Worker (⚠️ 核心步骤)
1. 回到 Workers 列表，点击你的 Worker 项目（如果还没创建，请先创建一个 Hello World Worker）。
2. 点击顶部的 **Settings (设置)** 标签。
3. 找到 **Variables (变量)** 或 **Bindings** 区域。
4. 在 **KV Namespace Bindings** 下点击 **Add Binding**。
5. **填写绑定信息**：
   - **Variable name (变量名)**: 必须填 **`KV`** (必须大写，不能改)。
   - **KV Namespace**: 选择刚才创建的 `ANIME_KV`。
6. 点击 **Save and deploy**。

---

## 🚀 第五步：部署代码

1. 在 Worker 详情页点击 **Edit code (编辑代码)**。
2. **全选并删除** 编辑器内原有的所有代码。
3. **复制粘贴** 本项目提供的 `worker.js` 完整代码。
4. 在代码最顶部的 **配置区**，填入前几步获取的密钥：

```javascript
// ============================================
// 🔴 必填：配置区
// ============================================
const TMDB_TOKEN = "eyJhbGciOiJIUzI1NiJ9..."; // 填入第一步的长 Token
const TRAKT_CLIENT_ID = "你的Client_ID";       // 填入第二步的 ID
const TRAKT_ACCESS_TOKEN = "你的Access_Token"; // 填入第三步的 Token
```

5. 点击右上角 **Save and Deploy (保存并部署)**。
6. 访问你的 Worker 网址，大功告成！🎉

---

## ❓ 常见问题 (FAQ)

**Q1: 为什么刚看完一集，网页上没更新？**
> **A**: 因为开启了 KV 缓存。为了兼顾速度，个人数据（历史/收藏）设置了 **1分钟** 的缓存时间。请等待 1 分钟后刷新即可。

**Q2: ReqBin 报错 401/400？**
> **A**: 验证码（Code）是一次性的且时效很短。请重新执行 **3.1 步骤** 获取新的验证码，然后立即去 ReqBin 发送请求。

**Q3: 网页一直加载或显示空白？**
> **A**: 
> 1. 检查 Cloudflare 后台 KV 绑定变量名是否为 `KV`。
> 2. 检查 `TMDB_TOKEN` 是否完整。
> 3. 确保你的 Trakt 账号里确实有观看记录。

**Q4: 如何强制刷新缓存？**
> **A**: 虽然不建议，但你可以去 Cloudflare 后台 -> KV -> 查看 `ANIME_KV` -> 手动删除里面的 Key，或者等待它自动过期。
