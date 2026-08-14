# 技术方案文档 — 技术内容自动发布 WordPress

| 项目 | 内容 |
|------|------|
| 文档版本 | v1.0 |
| 关联文档 | PRD（docs/PRD.md）、开发文档（docs/DEVELOPMENT.md）、环境文档（docs/ENVIRONMENT.md） |
| 技术栈 | n8n + DeepSeek + WordPress REST API |

---

## 1. 系统架构

```
┌────────────────────────────────────────────────────────────┐
│                    n8n 工作流引擎                            │
│                                                            │
│  Schedule Trigger → Code → AI Agent → Code → HTTP → ...   │
│                    (执行顺序 v1, 串行)                       │
└──────────┬─────────────────────────────────────────────────┘
           │
     ┌─────┴─────┐      ┌──────────────────┐
     │ DeepSeek  │      │  WordPress REST  │
     │ Chat API  │      │  API (wp-json)   │
     └───────────┘      └──────────────────┘
```

### 核心设计原则
- **串行执行**：`executionOrder: v1`，节点按连线顺序依次执行。
- **单一路径**：避免 Merge 节点双输入等待，以 IF 分流 + 单路输出替代。
- **断流防护**：HTTP 空数组响应、空标签等场景均不会导致链路中断。

---

## 2. 节点设计

### 2.1 触发层
| 节点 | 类型 | 参数 |
|------|------|------|
| Schedule Trigger | `scheduleTrigger` | 每 6 小时 |

### 2.2 选题层
| 节点 | 类型 | 说明 |
|------|------|------|
| Code - 选题 | `code` | 定义 6 主题数组，随机取 1，输出 `{category, instruction}` |

### 2.3 AI 生成层
| 节点 | 类型 | 说明 |
|------|------|------|
| AI Agent | `@n8n/n8n-nodes-langchain.agent` | 接收 instruction，调用 DeepSeek |
| DeepSeek Chat Model | `lmChatDeepSeek` | LangChain 模型节点，绑定 deepSeekApi 凭据 |

**提示词要点**：
- 真实性约束（禁止编造版本号、价格、跑分）。
- 输出格式为严格 JSON。
- 目标：SEO/GEO 优化原创技术文章。

### 2.4 解析层
| 节点 | 类型 | 说明 |
|------|------|------|
| Code - 解析AI输出 | `code` | 解析 JSON，兼容纯 JSON / markdown json 代码块 / 非标准空格 |

### 2.5 查重层
| 节点 | 类型 | 说明 |
|------|------|------|
| HTTP Request - 查重 | `httpRequest` | GET 已发布文章 slug，`alwaysOutputData: true` |
| Code - 判断重复 | `code` | 跨节点引用解析，输出含 `isDuplicate` 布尔值 |
| IF - 是否重复 | `if` | 分流：重复→结束，非重复→发布链路 |

### 2.6 分类/标签层
| 节点 | 类型 | 说明 |
|------|------|------|
| Code - 准备术语 | `code` | 补齐默认标签，保证 tagNames ≥ 3 |
| HTTP - 创建分类 | `httpRequest` | POST 分类，`continueOnFail: true` |
| Code - 取得分类ID | `code` | 从响应/错误响应中提取 `term_id` |
| Code - 展开标签 | `code` | 每个标签拆成独立 item |
| HTTP - 创建标签 | `httpRequest` | POST 标签，`continueOnFail: true` |
| Code - 重组标签ID | `code` | 汇总所有标签 ID，校验数量 |

### 2.7 发布层
| 节点 | 类型 | 说明 |
|------|------|------|
| Code - 组装发布参数 | `code` | 组装 REST 请求体，`categories:[ID]`、`tags:[ID]` |
| HTTP - 创建文章 | `httpRequest` | POST 发布文章 |

---

## 3. 数据流设计

### 3.1 核心数据结构

**选题输出**：
```json
{ "category": "开源程序教程", "instruction": "选择一款知名开源程序..." }
```

**AI 输出**（解析后）：
```json
{
  "title": "标题",
  "slug": "url-alias",
  "excerpt": "摘要",
  "content": "<p>HTML正文</p>",
  "category": "分类名",
  "tags": ["标签1", "标签2", "标签3"]
}
```

**查重后**：
```json
{ "isDuplicate": false, "title": "...", ... }
```

**准备术语后**：
```json
{ "category": "分类名", "tagNames": ["标签1", "标签2", "标签3"], ... }
```

**发布参数**：
```json
{
  "title": "...",
  "content": "...",
  "slug": "...",
  "excerpt": "...",
  "status": "publish",
  "comment_status": "open",
  "categories": [1],
  "tags": [1, 2, 3]
}
```

### 3.2 跨节点引用
- `Code - 判断重复` 通过 `$('Code - 解析AI输出')` 引用文章数据。
- `Code - 取得分类ID` 通过 `$('Code - 准备术语')` 引用上下文。
- `Code - 重组标签ID` 通过 `$('Code - 取得分类ID')` 引用分类 ID 及上下文。

---

## 4. 凭据管理

| 凭据名称 | 类型 | 用途 | 存储方式 |
|----------|------|------|----------|
| deepSeekApi | DeepSeek | AI 模型调用 | n8n 凭据系统 |
| wordpressApi | WordPress | REST API 调用 | n8n 凭据系统 |

凭据 ID 在 JSON 中记录，导入新实例后需重新绑定。

---

## 5. 边界场景处理

| 场景 | 处理方式 |
|------|----------|
| 站点无文章（空 slug 列表） | HTTP Request 输出 0 项，`alwaysOutputData: true` 继续流转 |
| 目标 slug 为空 | 视为不重复，继续发布 |
| 分类已存在 | POST 返回 400/term_exists，`continueOnFail: true`，从错误响应解析已有 ID |
| 标签已存在 | 同上 |
| AI 输出非 JSON | Code 节点 throw Error，执行失败并可在 n8n 查看错误 |
| 标签不足 3 个 | 自动补齐默认标签 |
| 标签为空数组 | 补齐默认标签后继续 |