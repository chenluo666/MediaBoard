# MediaBoard · 国漫更新榜

一个部署在 **Cloudflare Workers + D1** 上的单文件影视周更管理面板。按「周一 ~ 周日 + 完结」维护国漫条目，自动到 TMDB 补全元数据，每天自动记录人气快照生成热度趋势，并可一键把分类结果推送到 GitHub 供播放器 / 客户端读取。

整个后端只有一个文件：`worker.js`（无构建步骤、无第三方依赖）。

---

## 功能

### 榜单管理

- **多榜单**：可创建多个榜单并切换激活，同步 GitHub 时只推当前激活榜单
- **榜单锁定**：锁定的榜单不能直接删除（先解锁）；且始终至少保留一个榜单
- **分组**：`全部 / 周一 / 周二 / 周三 / 周四 / 周五 / 周六 / 周日 / 完结`
- **添加 / 删除条目**：添加时可选更新日，并做名称重复检测
- **拖拽排序**：列表视图内直接拖动调整条目顺序，顺序会随状态一起保存
- **撤回删除**：删除栈最多保留 10 条，可逐条撤回
- **删除保护**：连续删除 ≥ 3 次后要求输入管理员口令

### TMDB 集成

- 搜索匹配：条目级「匹配」和全榜批量「匹配 TMDB」
- 批量匹配自动补全 `tmdb_id`、海报、首播年份、季数、集数、评分、类型标签、播出平台 Logo
- 若干常见作品内置固定 `tmdb_id`，规避 TMDB 中文搜索不准
- 详情弹窗：评分、人气、简介、趋势大图、跳转 TMDB、重新匹配
- TMDB 凭据保存在服务端状态里（支持 Bearer Token 或 api_key），前端可隐藏显示、可单独清空

### 热度榜与趋势

- **热度榜 Top 8**：按 TMDB 人气值（popularity）排序
- **人气趋势**：服务端每天记录一次快照，按上海时区日期去重；同一天多次触发只更新当日值，不重复记点
- **需要 ≥ 2 天的数据**才会出现折线
- **自动同步**：Cloudflare Cron 每天 UTC 03:00（北京时间 11:00）自动跑一次
- **手动补记**：页面「更新热度」按钮，或调 `GET /api/auto-heat`
- **免费额度保护**：单次任务最多请求 45 部（`HEAT_LIMIT`），避开 Workers 免费版 50 子请求上限

### GitHub 同步

把当前激活榜单拆成 **8 个 JSON** 推送到目标仓库根目录：

| 文件 | 内容 |
| --- | --- |
| `周一.json` ~ `周日.json` | 该更新日的作品 |
| `完结.json` | 已完结的作品 |

每个文件的内容格式（保持与外部客户端的既有约定）：

```json
[
  { "title": "吞噬星空", "tmdb_id": 101172 }
]
```

已存在的文件会带 `sha` 更新，不存在则新建。置 `AUTO_SYNC_GITHUB=1` 后，每天自动任务跑完会自动推一次 GitHub。

### 视图与交互

- **列表 / 海报墙** 双视图切换
- **数据看板**：作品总数、连载中、已完结、平均评分、已匹配、总集数
- **搜索**：输入即显「一键清空」按钮（`Esc` 亦可清空）
- **行内删除键**：鼠标不在行上时隐藏，悬停时左滑出现（移动端无 hover，保持常显）
- **右下角回顶 / 回底**：顶部只显示「回底」，底部只显示「回顶」，不足一屏时整组隐藏
- **12 种主题色**（幻紫 / 星蓝 / 天青 / 青碧 / 翡翠 / 翠竹 / 琥珀 / 落日 / 珊瑚 / 玫瑰 / 紫罗兰 / 石墨）+ 明暗双模式
- **毛玻璃质感**：半透明面板 + 背景光晕
- **弹幕背景**：可开关，暗色 / 浅色两套观感，700+ 条弹幕词库随机飘过
- **图标体系**：全部使用 [Lucide](https://lucide.dev) 线性图标（ISC 许可），统一 24×24 描边、跟随主题色
- **PWA**：可安装为桌面 / 手机 App，自带 manifest、Service Worker 与四档图标（192 / 512 / maskable / SVG）
- **移动端优化**：批量渲染、图片懒加载、移动端关闭毛玻璃、禁用固定背景、搜索防抖 180ms

---

## 项目结构

```text
.
├── worker.js                        # Cloudflare Worker 单文件（前端 HTML/CSS/JS 全部内联）
├── example-data/
│   ├── anime_by_weeks.json          # 按周分组（周一 ~ 周日）的示例榜单
│   └── import-items.json            # 数组形态的导入示例，含可选字段
├── wrangler.toml.example            # Wrangler 配置模板（含 Cron Trigger）
└── README.md
```

---

## 部署

### 1. 创建 Worker

Cloudflare Dashboard → Workers & Pages → 创建 Worker。把 `worker.js` 全文粘贴进编辑器，或用 Wrangler 部署。

### 2. 创建并绑定 D1

```bash
npx wrangler d1 create mediaboard
```

绑定名称写 `DB`、`D1`、`DATABASE` 任意一个都可以，代码会自动识别（`findD1(env)`）。

首次部署后执行一次建表：

```sql
CREATE TABLE IF NOT EXISTS app_state (
  id INTEGER PRIMARY KEY CHECK (id = 1),
  data TEXT NOT NULL
);
```

### 3. 添加 Cron Trigger（重要）

自动记录人气快照**依赖** Cron Trigger，没有它趋势图永远不会出现。

Dashboard 路径：Worker → 设置 → 触发器 → Cron Triggers → 添加 `0 3 * * *`。

或在 `wrangler.toml` 里声明：

```toml
[triggers]
crons = ["0 3 * * *"]
```

`0 3 * * *` 是 UTC 时间，等于**北京时间每天 11:00**。要换时间自行换算。

### 4. 配置环境变量

| 变量 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `GITHUB_TOKEN` | 否 | — | GitHub PAT，需要目标仓库 `Contents: Read and write` 权限 |
| `GITHUB_REPO` | 否 | `chenluo666/EplayerX_Blocks` | 目标仓库，格式 `owner/repo` |
| `GITHUB_BRANCH` | 否 | `main` | 推送分支 |
| `AUTO_SYNC_GITHUB` | 否 | 未设置 | 设为 `1` 时，每天自动任务结束后自动推送 GitHub |
| `ADMIN_PASSWORD` | 否 | `mediaboard2026` | 管理员口令，用于清除数据、删除榜单、连续删除保护 |

> **`ADMIN_PASSWORD` 务必改成自己的。** 代码里留了默认值，任何人知道你的站点地址就能用默认口令清数据。
> Token 类变量请用 Cloudflare 的 **Secret**（加密变量），不要写进 `wrangler.toml`，也不要提交到公开仓库。

### 5. Wrangler 完整示例

```toml
name = "mediaboard"
main = "worker.js"
compatibility_date = "2026-09-07"

[[d1_databases]]
binding = "DB"
database_name = "mediaboard"
database_id = "your-d1-database-id"

[triggers]
crons = ["0 3 * * *"]

[vars]
GITHUB_REPO = "your-username/your-repo"
GITHUB_BRANCH = "main"

# 敏感值建议改用 Secret 注入：
# npx wrangler secret put GITHUB_TOKEN
# npx wrangler secret put ADMIN_PASSWORD
```

---

## API

所有接口返回 JSON，`/api/*` 带 CORS 头、不被 Service Worker 缓存。

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/api/state` | 返回完整应用状态，前端用它加载数据 |
| `POST` | `/api/state` | 保存完整应用状态 |
| `GET` | `/api/schedule` | 当前激活榜单按周分组的精简结果 |
| `POST` | `/api/sync-github` | 把当前榜单推成 8 个 JSON，需要 `GITHUB_TOKEN` |
| `GET` / `POST` | `/api/auto-heat` | 手动触发一次人气快照记录，返回 `{ok, updated, ...}` |
| `POST` | `/api/tmdb/search` | 代理搜索 TMDB，body：`{credential, type, query, tmdbId, language}` |
| `POST` | `/api/tmdb/detail` | 代理拉取 TMDB 详情 |
| `POST` | `/api/verify-password` | 校验管理员口令，body：`{password}` |
| `GET` | `/manifest.webmanifest` | PWA manifest |
| `GET` | `/sw.js` | Service Worker（导航走网络优先，更新不会被旧缓存卡住） |
| `GET` | `/icon.svg`、`/icon-192.png`、`/icon-512.png`、`/icon-512-maskable.png` | PWA 图标 |

`GET /api/schedule` 返回示例：

```json
{
  "周一": [{ "title": "吞噬星空", "tmdb_id": 101172 }],
  "周二": [],
  "周三": [],
  "完结": []
}
```

---

## 导入 / 导出

**导出**：右上角「导出 JSON」得到 `<榜单名>.json`，内容与 GitHub 同步产物一致（按周分组，不含 `全部`）。

**导入**：接受两种文件形态：

```json
[{ "name": "吞噬星空", "tmdbId": 101172, "day": "周一" }]
```

```json
{ "items": [{ "title": "吞噬星空", "tmdb_id": 101172 }] }
```

条目字段做兼容匹配：

| 用途 | 接受的字段名 |
| --- | --- |
| 名称 | `name` / `title` / `showName` |
| TMDB ID | `tmdbId` / `tmdb_id` / `id` |
| 更新日 | `day` / `weeklyDay` |
| 海报 | `poster` |

导入按 TMDB ID 或名称去重：已存在则补全缺失字段，不存在则新增。`poster` 非必填，缺失时自动占位。

---

## 本地验证

无构建步骤，直接用 Node 做语法检查：

```bash
node --check worker.js
```

---

## 常见问题

**装成 App 后 Windows 卸载列表里找不到？**
Chromium 系 PWA 的已知行为。Edge 可在 `edge://apps` 里卸载，或执行：

```powershell
& "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" --uninstall-app-id=<app-id>
```

**趋势图不出现？**
至少需要两天快照。部署当天只有一个点，第二天自动任务跑完才会出折线。

**更新 worker 代码后页面没变？**
Service Worker 导航走网络优先，正常刷新即可；仍有旧版就 `Ctrl + Shift + R`。

**想换 PWA 图标？**
图标以 base64 内联在 `worker.js` 顶部的 `PWA_ICON_*` 常量里，替换即可，不需要额外托管目录。

---

## 隐私与安全

- 仓库代码中不含任何真实 Token / 口令，敏感信息全部走环境变量或 Secret
- 默认同步仓库只是占位值，请自行用 `GITHUB_REPO` 覆盖
- 示例数据仅用于展示导入格式，不含个人信息
- 请务必修改默认的 `ADMIN_PASSWORD`
