# 📺 Cloudflare Worker - TMDB & Trakt 追番面板（最终定制版 V8）

这是一个基于 **Cloudflare Workers** 的 Serverless 追番管理面板，整合 **TMDB (The Movie Database)** 与 **Trakt.tv**，用于同步追番列表、观看进度与更新日历，并提供一个偏 iOS 风格、适合移动端使用的 Web App（支持 PWA）。

---

## ✨ 项目功能

* **无需服务器**：完全运行在 Cloudflare Edge 网络，免费、低延迟、免维护。
* **Trakt 双向同步**：

  * Watchlist（待看列表）
  * History（观看历史）
  * Calendar（剧集更新日历）
  * 页面内直接加入 / 取消追番、标记已看，实时同步回 Trakt。
* **TMDB 收藏同步**：同步 TMDB 网站中的收藏内容。
* **KV 智能缓存**：使用 Cloudflare KV 缓存请求，提升速度并避免 API 超限。
* **精美 UI**：类 iOS 原生风格，支持添加到主屏幕（PWA）。

---

## 🛠️ 准备工作

开始前，请准备以下账号：

1. **Cloudflare**：[https://dash.cloudflare.com/（部署](https://dash.cloudflare.com/（部署) Worker 与 KV）
2. **TMDB**：[https://www.themoviedb.org/（影视数据与海报）](https://www.themoviedb.org/（影视数据与海报）)
3. **Trakt**：[https://trakt.tv/（追番与观看进度同步）](https://trakt.tv/（追番与观看进度同步）)

---

## 🚀 部署教程（完整流程）

### 一、获取 TMDB API Token

1. 登录 TMDB：[https://www.themoviedb.org/](https://www.themoviedb.org/)
2. 点击头像 → **Settings** → **API**。
3. 点击 **Create** → 选择 **Developer**。
4. 填写申请信息（URL 可填写 `http://localhost`）。
5. 创建完成后，在 API 页面中找到：

   **API Read Access Token**

⚠️ 注意：

* 必须使用 **API Read Access Token（长 Token）**
* 不是短的 API Key
* 后续环境变量名为：`TMDB_TOKEN`

---

### 二、创建 Trakt API 应用

1. 登录 Trakt：[https://trakt.tv/](https://trakt.tv/)

2. 打开应用创建页面：[https://trakt.tv/oauth/apps](https://trakt.tv/oauth/apps)

3. 点击 **NEW APPLICATION**。

4. 填写以下信息：

   * **Name**：任意（如 `MyCloudflareTracker`）
   * **Redirect URI**：

     ```
     urn:ietf:wg:oauth:2.0:oob
     ```
   * **Javascript (CORS) origins**：留空或填写 `*`

5. 保存应用。

6. 记录以下信息：

   * `Client ID`
   * `Client Secret`

⚠️ 后续环境变量名：

* `TRAKT_ID`
* `TRAKT_SECRET`

---

### 三、获取 Trakt 初始 Refresh Token（关键步骤）

由于 Cloudflare Worker 无法进行交互式登录，需要手动完成一次授权，获取可自动刷新的 Refresh Token。

#### 1. 获取授权 PIN Code

将下方链接中的 `YOUR_CLIENT_ID` 替换为你的 Trakt Client ID，然后在浏览器中访问：

```
https://trakt.tv/oauth/authorize?response_type=code&client_id=YOUR_CLIENT_ID&redirect_uri=urn:ietf:wg:oauth:2.0:oob
```

* 点击 **Yes** 授权
* 页面会显示一个 **8 位 PIN 码**

---

#### 2. 使用在线工具换取 Refresh Token

1. 打开在线请求工具：[https://reqbin.com/curl](https://reqbin.com/curl)
2. 将以下内容粘贴到左侧编辑框：

```bash
curl -X POST https://api.trakt.tv/oauth/token \
-H "Content-Type: application/json" \
-d '{
  "code": "这里填写_8位_PIN",
  "client_id": "这里填写_CLIENT_ID",
  "client_secret": "这里填写_CLIENT_SECRET",
  "redirect_uri": "urn:ietf:wg:oauth:2.0:oob",
  "grant_type": "authorization_code"
}'
```

3. 替换 PIN、Client ID、Client Secret。
4. 点击 **Run**。
5. 在返回的 JSON 中找到：

```json
"refresh_token": "xxxxxxxxxxxxxxxx"
```

⚠️ 注意：

* 复制 refresh_token 的值（不带引号）
* 后续环境变量名为：`TRAKT_INIT_REFRESH`

---

### 四、创建并部署 Cloudflare Worker

1. 登录 Cloudflare Dashboard。
2. 进入 **Workers & Pages → Create Application → Create Worker**。
3. 命名 Worker（如 `my-trakt-app`）。
4. 点击 **Deploy**。
5. 点击 **Edit Code**。
6. 删除默认代码，粘贴项目中的 `worker.js`。
7. 点击 **Save and Deploy**。

---

### 五、绑定 KV 数据库（用于缓存）

1. 进入 **Workers & Pages → KV**。
2. 点击 **Create a Namespace**。
3. 命名为（示例）：

```
TRAKT_CACHE
```

4. 回到 Worker → **Settings → Variables**。
5. 在 **KV Namespace Bindings** 中添加：

| 项目            | 值             |
| ------------- | ------------- |
| Variable name | `KV`（必须大写）    |
| KV Namespace  | `TRAKT_CACHE` |

6. 保存并部署。

---

### 六、配置环境变量

在 **Worker → Settings → Variables → Environment Variables** 中添加以下变量：

| 变量名                  | 值               | 来源          |
| -------------------- | --------------- | ----------- |
| `TRAKT_ID`           | Client ID       | Trakt 应用后台  |
| `TRAKT_SECRET`       | Client Secret   | Trakt 应用后台  |
| `TRAKT_INIT_REFRESH` | Refresh Token   | 第三步获取       |
| `TMDB_TOKEN`         | TMDB Read Token | TMDB API 页面 |

点击 **Save and Deploy**。

---

## 🎉 完成

访问你的 Worker 地址，例如：

```
https://my-trakt-app.username.workers.dev
```

* 首次加载可能较慢（尚未缓存）
* 后续访问速度极快

---

## 💡 使用说明

* **新番时刻表**：显示当前播出的动画 / 剧集，按星期排列。
* **我的**：

  * 继续观看（Calendar）
  * 播放记录（History）
  * 我的追番（Watchlist）
  * TMDB 收藏
* **发现**：查看热门与趋势内容。
* **操作方式**：

  * 点击海报进入详情页
  * 点击右上角 `...` 呼出快捷菜单
  * 支持加入追番、标记已看，实时同步至 Trakt

---

## ❓ 常见问题（FAQ）

### 页面显示 `Config Error`

请检查：

* 环境变量是否全部配置
* 变量名是否全大写
* 是否存在多余空格

---

### 页面显示 `Auth Failed`

`TRAKT_INIT_REFRESH` 可能已失效，请重新执行 **第三步** 获取新的 Refresh Token 并更新环境变量。

---

### 图片无法加载

TMDB 图片域名 `image.tmdb.org` 在部分网络环境中可能被阻断。图片由客户端直接加载，需要自行解决网络访问问题。

---

### 如何强制刷新缓存

访问以下路径即可清空 KV 缓存：

```
https://你的Worker地址/flush
```
