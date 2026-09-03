# tg-image-bed-worker · Telegram 图床反代 Worker

部署在 Cloudflare Workers 上的 Telegram 频道图床**访客访问层**：把图片存进 Telegram 频道（Bot API），访客经本 Worker 匿名读图——**Bot Token 只存在 Worker 的 secret 里，浏览器永远拿不到**。

```
访客 <img src="https://你的Worker域名/f/{file_id}">
  │
  ▼
本 Worker（Cloudflare 边缘，持 Token）
  │  ① getFile?file_id=xxx → 解析临时 file_path
  │  ② fetch api.telegram.org/file/bot<TOKEN>/{file_path}
  ▼
Telegram 服务器 ──图片字节──▶ CF Cache 缓存 ──▶ 访客
```

## 特性

- **零成本**：Cloudflare Workers 免费额度（每天 10 万次请求）足够个人博客图床使用
- **零依赖**：纯单文件 JavaScript，无需 R2/D1/KV 绑定，无需数据库
- **Token 安全**：Bot Token 经 `wrangler secret` 注入，只存在于 Cloudflare 服务端；访客可见的唯一标识是 `file_id`（Telegram 公开文件标识，泄露无风险）
- **边缘缓存**：Cache API 缓存图片字节（命中时连 `getFile` 都省），浏览器侧一天缓存
- **绕开网络限制**：`*.workers.dev` 或自定义域名走 Cloudflare 全球网络；配合自定义域名可规避部分网络环境对 `workers.dev` 的污染

## 部署方式一：wrangler CLI（推荐）

```bash
# 0) 准备：一个 Cloudflare 账号 + 本仓库三个文件（index.js / wrangler.toml / README 可不带走）
# 1) 克隆或下载本仓库
git clone https://github.com/roberts9012062/tg-image-bed-worker.git
cd tg-image-bed-worker

# 2) 登录 Cloudflare（浏览器授权）
npx wrangler login

# 3) 注入 Bot Token 为 secret（从 @BotFather 获取；不要写进任何文件）
npx wrangler secret put TG_BOT_TOKEN

# 4) 部署
npx wrangler deploy
```

部署成功会输出 `https://tg-image-bed-worker.<你的账号>.workers.dev`。

## 部署方式二：Dashboard 在线编辑器（零命令行）

1. 打开 [Cloudflare Dashboard](https://dash.cloudflare.com/) → **Workers & Pages** → **Create** → **Create Worker**；
2. 部署默认 Worker 后点 **Edit code**，把本仓库 [`index.js`](index.js) 的全部内容粘贴进去覆盖，**Deploy**；
3. 进入该 Worker 的 **Settings → Variables and Secrets** → **Add**：
   - Type 选 **Secret**，Name 填 `TG_BOT_TOKEN`，Value 粘贴你的 Bot Token；
4. 完成。Worker 地址在 Overview 页的 `xxx.workers.dev`。

## 配置

| 环境变量 | 类型 | 说明 |
|---|---|---|
| `TG_BOT_TOKEN` | Secret（必填） | [@BotFather](https://t.me/BotFather) 创建的 Bot Token；Bot 需已加入存图的频道并为管理员 |

无其他配置。`wrangler.toml` 中的 `name` 决定默认域名，可自行修改。

## 路由契约

| 路由 | 说明 |
|---|---|
| `GET /health` | 存活探测 → `{"ok":true}`（未配置 Token 返回 500 + 提示） |
| `GET /f/{file_id}` | 读图：`getFile` 解析临时 `file_path` → 持 Token 回源 → 缓存 → 返回访客（`Content-Type` 按 file_path 扩展名推断） |

## 可选：绑定自定义域名

`*.workers.dev` 在部分网络环境可能被污染，**推荐绑定自有域名**（域名需托管在同一 Cloudflare 账号）：

```toml
# wrangler.toml 末尾追加，再次 wrangler deploy 即自动创建子域 DNS 与证书
routes = [
  { pattern = "tg.你的域名.com", custom_domain = true }
]
```

或 Dashboard：Worker → Settings → Domains & Routes → Add Custom Domain。注意一个域名只能路由到一个 Worker。

## 安全说明

- Token 只存 Cloudflare secret：仓库与本仓库任何文件都不含 Token，Worker 默认日志也不打印它；
- 泄露处置：立即在 @BotFather 执行 `/revoke` 吊销旧 Token → 重新生成 → `npx wrangler secret put TG_BOT_TOKEN` 更新 → 同步更新使用该 Token 的其他配置；
- 限制：单文件 ≤ 20MB（Telegram Bot API `getFile` 下载上限）；`file_path` 为临时链接，本 Worker 每次访问实时解析（对齐 [telegraph-Image](https://github.com/x-dr/telegraph-Image) 的 cfile 机制，致谢）。

## 与月言博客「TG图床」插件配合

本 Worker 是 [月言博客](https://github.com/roberts9012062/boke)（boke）**TG图床插件**的配套访客访问层：

1. 博客后台「插件商城」安装 **TG图床**；
2. 插件设置中填写：Bot Token（同一个）、频道 Chat ID、**本 Worker 的地址**（`https://tg-image-bed-worker.<账号>.workers.dev` 或自定义域名）；
3. 完成——发帖插图直达 Telegram 频道，图片 URL 即本 Worker 地址。

也可独立使用：任何「图片存在 TG 频道、需要公开 URL」的场景（上传侧自行调用 `sendDocument`/`sendPhoto` 获取 `file_id`）。
