🎬 Cloudflare Worker 追番列表配置指南

本项目是一个基于 Cloudflare Worker 的个人追番/观影列表，数据源自 Trakt 和 TMDB。在使用本代码前，你需要获取以下三个核心参数：

TMDB_TOKEN (用于获取封面图片和中文元数据)

TRAKT_CLIENT_ID (用于连接 Trakt API)

TRAKT_ACCESS_TOKEN (用于获取你的个人观看记录)

🟢 第一步：获取 TMDB Token

TMDB (The Movie Database) 提供电影和剧集的详细信息。

注册/登录

访问官网：https://www.themoviedb.org/

登录你的账号（如果没有请先注册）。

申请 API

点击右上角头像 -> 选择 Settings (设置)。

在左侧菜单点击 API。

点击 Create (创建) 或 Request an API Key。

选择 Developer (开发者)。

填写申请表：

用途: Personal

网址: http://localhost (随意填写)

简介: Personal watchlist display (随意填写)

补全地址信息后提交。

复制 Token

申请成功后，在 API 页面找到 API Read Access Token (v4 auth)。

⚠️ 注意：复制那个以 eyJ 开头的超级长的字符串，不是短的 API Key。

✅ 保存为: TMDB_TOKEN

🔴 第二步：获取 Trakt Client ID & Secret

Trakt 用于同步你的观看历史和日历。

创建应用

访问开发者后台：https://trakt.tv/oauth/applications

点击右上角的 NEW APPLICATION。

填写配置 (⚠️ 关键步骤)

Name: MyWorkerList (随意)

Redirect URI: 必须严格填入以下地址 👇

code
Text
download
content_copy
expand_less
urn:ietf:wg:oauth:2.0:oob

Permissions: 保持默认即可。

点击底部的 SAVE APP。

保存密钥

保存页面上显示的 Client ID (这是 TRAKT_CLIENT_ID)。

保存页面上显示的 Client Secret (下一步换 Token 用)。

🔵 第三步：获取 Trakt Access Token (使用 ReqBin)

这是最重要的一步，我们需要用“验证码”换取“访问令牌”。

3.1 获取授权码 (Code)

复制下方链接到记事本：

code
Text
download
content_copy
expand_less
https://trakt.tv/oauth/authorize?response_type=code&client_id=你的Client_ID&redirect_uri=urn:ietf:wg:oauth:2.0:oob

将链接中的 你的Client_ID 替换为第二步获取的 ID。

在浏览器访问替换后的链接，点击绿色按钮 Yes 授权。

复制页面显示的 8位数字验证码。

3.2 使用 ReqBin 换取 Token

打开在线 API 工具：https://reqbin.com/

设置请求头：

方法下拉框选择：POST

地址栏填入：https://api.trakt.tv/oauth/token

点击 Headers 标签，填入：Content-Type: application/json

填写内容 (Content)：

点击 Content 标签，选择 JSON 格式。

复制下方代码填入（替换为你自己的数据）：

code
JSON
download
content_copy
expand_less
{
    "code": "这里填刚才获取的8位验证码",
    "client_id": "这里填第二步的 Client ID",
    "client_secret": "这里填第二步的 Client Secret",
    "redirect_uri": "urn:ietf:wg:oauth:2.0:oob",
    "grant_type": "authorization_code"
}

发送请求：

点击 Send。

在右侧返回结果中找到 "access_token": "..."。

✅ 引号里的字符串就是：TRAKT_ACCESS_TOKEN

⚠️ 提示：如果报错，通常是因为验证码过期了。验证码是一次性的，请重新执行 3.1 步骤获取新验证码，并立即使用。

📝 第四步：填入代码

回到你的 worker.js 文件，找到顶部的配置区进行修改：

code
JavaScript
download
content_copy
expand_less
// ============================================
// 🔴 必填：配置区
// ============================================

// 1. 填入第一步获取的 TMDB Token (eyJ开头...)
const TMDB_TOKEN = "eyJhbGciOiJIUzI1NiJ9.eyJu..."; 

// 2. Trakt 配置 (必须填入才能获取播放时间)
// 填入第二步获取的 Client ID
const TRAKT_CLIENT_ID = "609ffd95de..."; 

// 3. 填入第三步获取的 Access Token
const TRAKT_ACCESS_TOKEN = "95e773e179..."; 

// 下方代码无需修改...
🛠 常见问题

列表是空的？

请确保你在 Trakt 网站上已经标记了一些看过的内容或者添加了待看列表。

请确保 TMDB_TOKEN 是 v4 版本的长 Token。

ReqBin 返回 401/400 错误？

验证码（Code）只能用一次，且有时效性。重新访问授权链接获取新 Code 再试。

确保 JSON 格式没有少逗号或引号。

确保 Trakt 后台的 Redirect URI 必须是 urn:ietf:wg:oauth:2.0:oob。
