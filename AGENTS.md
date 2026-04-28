# AGENTS.md — AI 知识库助手项目指南

## 1. 项目概述

本项目是一个 AI 驱动的技术知识库助手，自动从 GitHub Trending 和 Hacker News 采集 AI/LLM/Agent 领域的技术动态，经 AI 分析与结构化处理后以 JSON 格式持久化存储，并支持通过 Telegram 和飞书等渠道进行分发，帮助团队高效追踪前沿技术资讯。

## 2. 技术栈

| 类别 | 技术 |
|------|------|
| 语言 | Python 3.12 |
| Agent 框架 | LangGraph |
| 采集工具 | OpenClaw |
| 开发环境 | OpenCode + 国产大模型 |
| 数据格式 | JSON |
| 分发渠道 | Telegram Bot、飞书 Bot |

## 3. 编码规范

- **代码风格**：遵循 PEP 8
- **命名**：变量与函数使用 `snake_case`，类使用 `PascalCase`
- **文档字符串**：使用 Google 风格 docstring

```python
def fetch_trending(topic: str, limit: int = 10) -> list[dict]:
    """从 GitHub Trending 获取指定主题的热门项目。

    Args:
        topic: 主题关键词，如 'llm' 或 'agent'。
        limit: 返回条目的最大数量。

    Returns:
        包含项目信息的字典列表。

    Raises:
        RequestError: 当网络请求失败时抛出。
    """
    ...
```

- **日志**：禁止裸 `print()`，统一使用 `logging` 模块
- **类型注解**：所有公开函数必须提供完整的类型注解
- **依赖管理**：使用 `requirements.txt` 或 `pyproject.toml`

## 4. 项目结构

```
ai-knowledge-base/
├── AGENTS.md                  # 本文件：Agent 行为与项目指南
├── .opencode/
│   ├── agents/                # Agent 定义与提示词
│   └── skills/                # 可复用的 Skill 模块
├── knowledge/
│   ├── raw/                   # 采集的原始数据（HTML/JSON）
│   └── artic/                 # 结构化后的知识条目
└── requirements.txt           # Python 依赖
```

## 5. 知识条目 JSON 格式

存储路径：`knowledge/artic/<id>.json`

```json
{
  "id": "gh-20260428-001",
  "title": "LangGraph v0.3 发布：支持多Agent编排",
  "source_url": "https://github.com/langchain-ai/langgraph/releases/tag/v0.3.0",
  "source_type": "github_trending",
  "summary": "LangGraph v0.3 引入了原生多Agent编排能力，支持动态子图调度和共享状态管理。",
  "tags": ["langgraph", "multi-agent", "orchestration"],
  "category": "framework",
  "status": "published",
  "collected_at": "2026-04-28T10:00:00Z",
  "analyzed_at": "2026-04-28T10:05:00Z",
  "published_at": "2026-04-28T10:10:00Z"
}
```

**字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 唯一标识，格式：`{来源缩写}-{日期}-{序号}` |
| `title` | string | 是 | 条目标题 |
| `source_url` | string | 是 | 原始链接 |
| `source_type` | string | 是 | 来源类型：`github_trending` / `hacker_news` |
| `summary` | string | 是 | AI 生成的中文摘要（100-300字） |
| `tags` | string[] | 是 | 标签列表，至少1个 |
| `category` | string | 是 | 分类：`framework` / `model` / `paper` / `tool` / `tutorial` / `news` |
| `status` | string | 是 | 状态：`raw` → `analyzed` → `published` |
| `collected_at` | string | 是 | ISO 8601 采集时间 |
| `analyzed_at` | string | 否 | ISO 8601 分析时间 |
| `published_at` | string | 否 | ISO 8601 分发时间 |

## 6. Agent 角色概览

| 角色 | 职责 | 输入 | 输出 | 定义位置 |
|------|------|------|------|----------|
| **采集 Agent** (Collector) | 从 GitHub Trending / HN 爬取 AI 相关条目，去重后存入 `knowledge/raw/` | 定时触发 / 手动触发 | 原始 JSON 文件 | `.opencode/agents/` |
| **分析 Agent** (Analyzer) | 读取原始数据，调用大模型生成中文摘要、提取标签和分类，写入结构化 JSON | `knowledge/raw/` 中的文件 | `knowledge/artic/<id>.json` | `.opencode/agents/` |
| **分发 Agent** (Publisher) | 将 status=published 的条目推送到 Telegram / 飞书 | `knowledge/artic/` 中 status=analyzed 的条目 | 更新 status 为 published | `.opencode/agents/` |

**工作流状态机**：

```
raw → analyzed → published
          │
          └──→ failed（分析失败，保留原始数据供重试）
```

## 7. 红线（绝对禁止）

- **禁止**在代码中硬编码任何 API Key、Token 或密钥，统一使用环境变量
- **禁止**向已发布的知识条目（`status=published`）中回写修改摘要内容
- **禁止**删除 `knowledge/raw/` 中的原始数据，原始数据是不可变的事实来源
- **禁止**在没有类型注解的情况下合并公开函数
- **禁止**使用裸 `print()`，统一使用 `logging`
- **禁止**跳过去重逻辑直接写入已存在的 `id` 条目
- **禁止**在 Agent 流程中执行任何不可逆的 destructive 操作
