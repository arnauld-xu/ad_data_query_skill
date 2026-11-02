# ad-organic-traffic

通用 Agent Skill。用来查自然流量、广告流量、内容主题、关联产品和赛道匹配。

仓库地址：

https://github.com/arnauld-xu/ad_data_query_skill

支持任何能读 Agent Skills 的工具，不限 Cursor。

## 安装

把仓库里的 `skills/ad-organic-traffic/` 整个文件夹放到该工具的 skills 目录。文件夹里要有 `SKILL.md` 和 `reference.md`。

常见目录：

| 工具 | 个人目录 | 项目目录 |
|---|---|---|
| Cursor | `~/.cursor/skills/` 或 `~/.agents/skills/` | `.cursor/skills/` 或 `.agents/skills/` |
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Codex | `~/.codex/skills/` | `.codex/skills/` |
| 其他兼容 Agent Skills 的工具 | 该工具文档里的 skills 目录 | `.agents/skills/` |

也可以用命令：

```sh
npx skills add arnauld-xu/ad_data_query_skill --skill ad-organic-traffic
```

```sh
gh skill install arnauld-xu/ad_data_query_skill ad-organic-traffic
```

Cursor 还可以：Customize → From GitHub Repository，粘贴仓库地址后安装。

## Metabase MCP

查数前必须安装 Metabase MCP，没有就先提示安装，不要查表。

地址：

https://bi.kalowave.com/api/metabase-mcp

各工具自己的 MCP 配置写成：

```json
"metabase": {
  "url": "https://bi.kalowave.com/api/metabase-mcp"
}
```

装好后各自登录一次 Metabase。
