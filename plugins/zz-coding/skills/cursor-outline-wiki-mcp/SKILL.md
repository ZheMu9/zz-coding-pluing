---
name: cursor-outline-wiki-mcp
description: >-
  在 Cursor 中配置 Outline 官方 MCP（Streamable HTTP）、使用 API Key 认证、排查连接问题。
  适用于用户提到 Outline MCP、wiki.bamboogames.top、知识库 MCP、Cursor mcp.json、
  Authorization Bearer、自建 Outline /getoutline.com 子域名 等场景。
---

# Cursor × Outline Wiki（MCP）

## 何时查阅本 Skill

- 为新环境或新同事编写 **Cursor + Outline** 接入说明
- 用户已自建 Outline（如 `https://wiki.example.com`）需在 Cursor 里接 MCP
- MCP 连不上、401、或管理员关闭了工作区 AI 能力

## 配置文件位置

| 范围 | 路径 |
| ---- | ---- |
| 全局（推荐） | `~/.cursor/mcp.json`（Windows：`%USERPROFILE%\.cursor\mcp.json`） |
| 项目级 | 若团队约定在项目内维护，使用仓库 `.cursor/mcp.json`（勿把密钥提交进 Git） |

修改后 **重启 Cursor** 或在 **Settings → MCP** 中刷新后再试。

## `mcp.json` 模板（API Key）

Outline 官方 MCP 仅支持 **Streamable HTTP**，在 Cursor 里用 `url` + 请求头即可。

```json
{
  "mcpServers": {
    "outline": {
      "url": "https://YOUR_OUTLINE_HOST/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_OUTLINE_API_KEY"
      }
    }
  }
}
```

### URL 怎么填

- **Outline 云服务**：`https://<子域名>.getoutline.com/mcp`
- **自建**：`https://<你的域名>/mcp`（与 [Outline MCP 文档](https://docs.getoutline.com/s/guide/doc/mcp-6j9jtENNKL) 一致）

团队当前 Wiki 示例（仅作地址参考，密钥仍用占位符）：`https://wiki.bamboogames.top/mcp`

### API Key 从哪来

Outline → 个人头像 → **Settings** → **API Keys** → 新建并复制（格式通常为 `ol_api_...`）。

### 认证头格式

必须使用：

`Authorization: Bearer <API Key>`

（与官方文档中 API Key 认证说明一致。）

## Outline 工作区侧检查

1. 管理员在 **Settings → AI** 中已开启 MCP（关闭则所有成员客户端无法连接）。
2. 反向代理需把 **`/mcp`** 正确转发到 Outline（自建常见坑：路径或 WebSocket/长连接被截断）。

## 验证与排错

| 现象 | 处理 |
| ---- | ---- |
| Cursor 里 outline 一直断开 | 核对 `url` 是否含末尾 `/mcp`、证书是否有效、是否需公司代理 |
| 401 / 权限错误 | 轮换 API Key、确认 Bearer 前缀与空格、`Authorization` 键名拼写 |
| 工具不可用 | 重启 Cursor；确认全局 `mcp.json` JSON 无语法错误 |

可选：用 API 冒烟（不经过 MCP）验证 Key 与域名——`POST https://<host>/api/collections.list`，请求体 `{}`，请求头同上 `Authorization`。

## 安全清单（代写文档或 skill 时必须提醒）

- **不要把 API Key 写进聊天、截图、PR、仓库**。仅放在本机 `mcp.json` 或密钥管理方案中。
- Key 一旦泄露：在 Outline 里 **作废并重建**。
- `mcp.json` 加入 `.gitignore`；若必须团队共享配置，使用占位符 + 各开发者本地覆盖。

## 与「快速开始」Wiki 的对应关系

团队在 Wiki **快速开始** 集合中有「全局安装 OutlineMCP」等文档时，本 Skill 与之一致：**Cursor Settings → MCP → Add**，粘贴与上文相同的 JSON 结构即可；具体域名与 Key 仍以各环境为准。

## 参考链接

- [Outline：MCP 与客户端配置](https://docs.getoutline.com/s/guide/doc/mcp-6j9jtENNKL)
