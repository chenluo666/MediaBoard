# MediaBoard

一个部署在 Cloudflare Workers 上的影视周更管理面板，支持按周一至周日维护动漫条目，自动匹配 TMDB，导入/导出 JSON，以及一键同步周更榜单到 GitHub 仓库。

## 功能

- 按“周一”到“周日”七个分组管理动漫
- 添加作品时可选择更新日，并做重复检测
- TMDB 搜索匹配，自动补充 `tmdb_id`
- 导入/导出 JSON，不强制要求 `poster` 字段
- 一键同步榜单到 GitHub 仓库，生成 `周一.json` 等文件
- 暗色/浅色主题切换，多种主题色
- 清除数据前三次确认，防止误操作
- 半透明弹幕背景，适配暗色模式

## 项目结构

```text
.
├── worker.js                # Cloudflare Worker 单文件
├── example-data/
│   └── anime_by_weeks.json   # 按周分组的示例榜单
└── README.md
```

## 部署到 Cloudflare Workers

1. 在 Cloudflare Dashboard 创建 Worker。
2. 创建 D1 数据库。
3. 将 D1 数据库绑定到 Worker，绑定名称可使用 `DB`、`D1` 或 `DATABASE`。
4. 把 `worker.js` 内容粘贴到 Worker 编辑器，或者用 Wrangler 部署。
5. 添加需要的环境变量。

### 环境变量

| 变量 | 必填 | 说明 |
| --- | --- | --- |
| `GITHUB_TOKEN` | 否 | GitHub Personal Access Token，需要 `repo` 权限，用于同步榜单 |
| `GITHUB_REPO` | 否 | 如 `username/anime-schedule-data`，默认 `YOUR_GITHUB_USERNAME/YOUR_GITHUB_REPO` |
| `GITHUB_BRANCH` | 否 | 同步仓库分支，默认 `main` |

> 所有敏感信息都通过环境变量注入，代码仓库中不保存任何真实 Token。

### Wrangler 示例

在项目根目录创建 `wrangler.toml`，按实际情况修改：

```toml
name = "anime-schedule-board"
main = "worker.js"
compatibility_date = "2026-09-07"

[[d1_databases]]
binding = "DB"
database_name = "anime-schedule"
database_id = "your-d1-database-id"

[vars]
GITHUB_REPO = "YOUR_GITHUB_USERNAME/YOUR_GITHUB_REPO"
GITHUB_BRANCH = "main"
```

`GITHUB_TOKEN` 建议通过 Cloudflare 的 Secret 功能配置，不要写在 `wrangler.toml` 或公开仓库中。

## API

### `GET /api/schedule`

返回当前激活榜单按周分组的 JSON：

```json
{
  "周一": [
    { "title": "吞噬星空", "tmdb_id": 101172 }
  ],
  "周二": []
}
```

### `GET /api/state`

返回完整的应用状态，用于前端保存和恢复。

### `POST /api/state`

保存完整应用状态。

### `POST /api/sync-github`

把当前榜单按周同步到 GitHub 仓库。需要配置 `GITHUB_TOKEN`。

## 本地验证

项目没有编译依赖，可直接使用 Node 做语法检查：

```bash
node --check worker.js
```

启动后的面板可以在浏览器中访问，页面自带弹幕背景和主题切换。

## 隐私说明

- 代码文件中不包含真实 GitHub Token。
- 默认同步仓库使用占位符，部署时通过环境变量覆盖。
- 示例数据仅用于展示导入格式，不包含任何账号或个人信息。
