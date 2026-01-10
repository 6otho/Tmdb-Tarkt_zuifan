# 🎬 Cloudflare Worker 追番/观影列表配置指南

本项目是一个基于 Cloudflare Worker 的个人影视追踪页面。你需要获取 **TMDB** 和 **Trakt** 的授权信息才能正常运行。

> **⚠️ 准备工作**
> 1. 一个 **TMDB** 账号
> 2. 一个 **Trakt** 账号
> 3. 打开文本编辑器（记事本），用于临时存放获取到的密钥

---

## 🟢 第一步：获取 TMDB Token (封面数据源)

我们需要 TMDB 的 API 令牌来获取海报和中文简介。

### 1. 注册与申请
1. 访问官网：[https://www.themoviedb.org/](https://www.themoviedb.org/) 并登录。
2. 点击右上角头像 → 选择 **Settings (设置)**。
3. 在左侧菜单栏点击 **API**。
4. 点击 **Create (创建)** 或 **Request an API Key**，选择 **Developer (开发者)**。
5. **填写申请表**（内容可以随意填，如下）：
   - **Type of use**: `Personal`
   - **URL**: `http://localhost`
   - **Summary**: `For personal watchlist display`
   - 补全地址信息后提交。

### 2. 复制关键 Token
申请成功后，在 API 页面下方找到 **API Read Access Token (v4 auth)**。

> ❌ **不要复制** 那个很短的 API Key。
> ✅ **要复制** 那个 **以 `eyJ` 开头、特别长** 的字符串。

👉 **请将它暂时保存，记作：`TMDB_TOKEN`**

---

## 🔴 第二步：创建 Trakt 应用 (历史记录源)

我们需要创建一个“应用”来允许 Worker 读取你的观看历史。

### 1. 创建应用
1. 访问开发者后台：[https://trakt.tv/oauth/applications](https://trakt.tv/oauth/applications)
2. 点击右上角的绿色按钮 **NEW APPLICATION**。

### 2. 填写配置（⚡️ 极其重要）
请严格按照以下内容填写，否则下一步无法获取 Token：

- **Name**: `MyWorkerList` (或者你喜欢的名字)
- **Description**: `Personal list`
- **Redirect URI**: 必须填入下方这行代码 👇
  ```text
  urn:ietf:wg:oauth:2.0:oob
  ```
- **Javascript (CORS) origins**: 留空即可。
- **Permissions**: 保持默认勾选。

点击底部的 **SAVE APP** 保存。

### 3. 保存 ID 和 Secret
保存成功后，页面上方会显示应用信息：
- 复制 **Client ID** 👉 **记作：`TRAKT_CLIENT_ID`**
- 复制 **Client Secret** 👉 **记作：`TRAKT_CLIENT_SECRET`** (下一步要用)

---

## 🔵 第三步：手动获取 Access Token (最关键一步)

由于 Trakt 需要 OAuth2 验证，我们需要手动生成一个永久访问令牌。我们将使用 **ReqBin** 这个在线工具来完成。

### 3.1 获取授权验证码 (Code)
1. 复制下方链接到记事本：
   ```text
   https://trakt.tv/oauth/authorize?response_type=code&client_id=你的Client_ID&redirect_uri=urn:ietf:wg:oauth:2.0:oob
   ```
2. 将链接中的 `你的Client_ID` 替换为 **第二步获取的 Client ID**。
3. 将替换好的完整链接粘贴到浏览器访问。
4. 点击绿色按钮 **Yes** 允许授权。
5. 页面会显示一串 **8位数的验证码**，复制它。

### 3.2 使用 ReqBin 换取 Token
1. 打开在线工具：[https://reqbin.com/](https://reqbin.com/)
2. **配置请求面板**（请严格按照图文操作）：
   - **请求方式**: 点击下拉菜单，将 `GET` 改为 **`POST`**。
   - **API 地址**: 填入 `https://api.trakt.tv/oauth/token`
   - **Headers**: 点击 `Headers` 标签页，填入：
     ```text
     Content-Type: application/json
     ```
3. **填写 JSON 内容**:
   - 点击 `Content` 标签页。
   - 确保格式选择的是 **JSON**。
   - 复制下方代码框的内容，并**修改为你自己的数据**：

   ```json
   {
       "code": "这里填刚才获取的8位验证码",
       "client_id": "这里填第二步的 Client ID",
       "client_secret": "这里填第二步的 Client Secret",
       "redirect_uri": "urn:ietf:wg:oauth:2.0:oob",
       "grant_type": "authorization_code"
   }
   ```

4. **发送与获取**:
   - 点击黄色的 **Send** 按钮。
   - 查看右侧的返回结果。
   - 找到 `"access_token": "..."` 这一行。
   - 引号里那一串乱码就是我们要的。

👉 **请保存它，记作：`TRAKT_ACCESS_TOKEN`**

---

## 📝 第四步：填入代码配置

回到你的 `worker.js` 代码文件，找到最顶部的配置区域，将刚才获取的三个参数填入双引号中。

```javascript
// ============================================
// 🔴 必填：配置区
// ============================================

// 1. 填入第一步获取的 TMDB 长 Token (eyJ开头...)
const TMDB_TOKEN = "eyJhbGciOiJIUzI1NiJ9.eyJu..."; 

// 2. Trakt 配置
// 填入第二步获取的 Client ID
const TRAKT_CLIENT_ID = "609ffd95de13..."; 

// 3. 填入第三步 ReqBin 返回的 Access Token
const TRAKT_ACCESS_TOKEN = "95e773e1792a..."; 
```

---

## 🛠 常见错误排查

1. **ReqBin 报错 401 或 400？**
   - **原因**：验证码（Code）过期了。
   - **解决**：Code 是一次性的且时效很短。请重新执行 **3.1 步骤** 获取新的验证码，然后立刻去 ReqBin 发送请求。

2. **ReqBin 报错 "redirect_uri mismatch"？**
   - **原因**：Trakt 后台的 Redirect URI 填错了。
   - **解决**：回到 Trakt 开发者后台，确保 Redirect URI 填的是 `urn:ietf:wg:oauth:2.0:oob`，一个字都不能差。

3. **页面显示空白？**
   - **解决**：去 Trakt.tv 网站上随便标记一部电影为“看过” (History) 或者加入“待看” (Watchlist)，列表才会有内容。
