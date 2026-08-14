# AI 智能体交接文档

> 本文档用于向 AI 智能体（或后续维护者）交接“技术内容自动发布 WordPress”工作流。请先阅读本文件以了解工作流的背景、结构、配置、已知问题与维护要点。

## 1. 项目概述

- **仓库**：`laobi465/n8n`
- **分支**：`workflow/tech-content-wordpress-publish`
- **工作流名称**：技术内容自动发布 WordPress
- **工作流文件**：`workflows/tech-content-wordpress-publish.json`
- **目标 WordPress 站点**：`https://word.apexck.com`
- **发布频率**：每 6 小时 1 篇

该工作流自动抓取 6 大技术主题中的随机一个，用 DeepSeek 生成原创 SEO/GEO 优化文章，自动创建/复用分类和标签，检测重复后发布到 WordPress。

## 2. 工作流结构与数据流

节点按顺序（`executionOrder: v1`）：

```
Schedule Trigger (每6小时)
  → Code - 选题
  → AI Agent (DeepSeek Chat Model)
  → Code - 解析AI输出
  → HTTP Request - 查重
  → Code - 判断重复
  → IF - 是否重复
      ├─ 重复 → 结束（main[0] 空）
      └─ 非重复 → Code - 准备术语
            → HTTP - 创建分类
            → Code - 取得分类ID
            → Code - 展开标签
            → HTTP - 创建标签
            → Code - 重组标签ID
            → Code - 组装发布参数
            → HTTP - 创建文章
```

### 各节点职责

| 节点 | 职责 | 关键点 |
| --- | --- | --- |
| Schedule Trigger | 每 6 小时触发 | 调整频率改这里 |
| Code - 选题 | 6 主题随机选 1 | 输出 `{category, instruction}` |
| AI Agent | DeepSeek 生成文章 | 提示词含真实性约束，输出严格 JSON |
| DeepSeek Chat Model | LLM 模型 | 绑定 deepSeekApi 凭据 |
| Code - 解析AI输出 | 解析 AI 的 JSON 输出 | 兼容纯 JSON / markdown json 代码块；slug 兜底生成 |
| HTTP Request - 查重 | 拉取已发布文章 slug | `alwaysOutputData: true`（防空响应断流） |
| Code - 判断重复 | 判定 slug 是否重复 | 无有效 slug 时视为不重复 |
| IF - 是否重复 | 分流 | 重复→结束；非重复→发布链路 |
| Code - 准备术语 | 整理分类/标签 | **标签不足自动补齐默认 3 个**（技术教程/实用技巧/网站运维） |
| HTTP - 创建分类 | 创建分类 | `continueOnFail: true`（已存在时返回冲突，需解析 term_id） |
| Code - 取得分类ID | 从响应提取分类 ID | 兼容多种响应结构 |
| Code - 展开标签 | 每个标签拆成独立 item | 依赖 `tagNames` 非空 |
| HTTP - 创建标签 | 创建标签 | `continueOnFail: true` |
| Code - 重组标签ID | 汇总标签 ID | 校验数量与 `tagNames` 一致 |
| Code - 组装发布参数 | 组装 REST 请求体 | `categories:[ID]`、`tags:[ID]` |
| HTTP - 创建文章 | POST 发布 | 最终发布动作 |

## 3. 关键配置与凭据

### 凭据（Credentials）
- **DeepSeek**：`deepSeekApi`，ID `SgrYhV32MeVZERum`。
- **WordPress**：`wordpressApi`，ID `qsJwYYosAjqF2zc9`。

> 注意：工作流 JSON 中记录了凭据 ID 与名称。若在另一 n8n 实例导入，ID 可能不同，需重新绑定凭据。

### WordPress 站点地址
默认指向 `https://word.apexck.com`。以下节点含该域名，切换站点需同步修改：
- `HTTP Request - 查重`
- `HTTP - 创建分类`
- `HTTP - 创建标签`
- `HTTP - 创建文章`

## 4. 已知问题与修复历史（重要）

以下是开发过程中遇到并修复的关键问题，维护时应避免回归。

### 4.1 “IF 节点不执行”/“工作流不执行”
**根因**：n8n 的 HTTP Request 节点在响应为空数组 `[]` 时输出 **0 个 item**，下游节点拿不到输入便不执行。
**修复**：
- 查重节点设置 `alwaysOutputData: true`，空数组也继续流转。
- `Code - 判断重复` 在无有效 slug 时判定为不重复，保证发布链路可继续。

### 4.2 “WordPress 无效参数：categories, tags”
**根因**：n8n WordPress 节点的 `categories`/`tags` 只接受术语 **ID 数组**，不接受名称字符串。
**修复**：改用 HTTP Request 节点直接调用 WordPress REST API，先创建/取得术语 ID，再以数字 ID 数组发布。

### 4.3 分类/标签存在时重复创建
**根因**：直接 POST 创建已存在的分类/标签会返回 `term_exists` 冲突错误。
**修复**：创建节点开启 `continueOnFail: true`，`Code - 取得分类ID` / `Code - 重组标签ID` 会从成功响应或错误响应中解析已有的 `term_id`。

### 4.4 双输入 Merge 导致执行卡死（已移除）
**根因**：IF 分支只执行一条，但双输入 Merge 会等待另一条永不执行的输入，工作流停在 Merge 前。
**修复**：移除所有 Merge 节点，改为**单一路径**的条件处理，每个分支直接输出完整上下文。

### 4.5 标签为空导致断流
**根因**：AI 未输出标签或标签全为空时，`Code - 展开标签` 输出 0 个 item。
**修复**：`Code - 准备术语` 自动补齐默认标签（技术教程、实用技巧、网站运维），保证标签始终 ≥ 3 个。

## 5. 维护与故障排查指南

### 5.1 执行失败如何排查
1. 在 n8n 打开工作流，进入 **Executions** 查看最近执行。
2. 定位失败节点与错误信息。
3. 常见失败点：
   - AI Agent 输出非 JSON → 检查 `Code - 解析AI输出` 报错。
   - 分类/标签 term_id 解析失败 → 检查 `Code - 取得分类ID` / `Code - 重组标签ID`。
   - 发布失败 → 检查 WordPress 凭据权限、站点地址、REST API 是否可用。

### 5.2 常见调整需求
- **改发布频率**：改 `Schedule Trigger` 的 hoursInterval。
- **改站点**：见上文“WordPress 站点地址”。
- **加主题**：在 `Code - 选题` 的 `themes` 数组中追加对象 `{category, instruction}`。
- **改默认标签**：在 `Code - 准备术语` 的 `defaults` 数组修改。

### 5.3 已验证的边界场景
- 站点已有发布文章：查重正常返回非空，IF 正常分流。
- 站点无标签：`Code - 准备术语` 补齐默认标签，标签创建链路正常。
- 分类已存在：创建节点返回冲突，ID 解析节点仍能取到已有 ID。

## 6. 部署与激活

- 部署：通过 n8n 编辑器 **Import from File** 导入，或 `n8n import:workflow --input=...`。
- 激活：导入后需在编辑器手动绑定凭据并点击 **Execute workflow** 测试，确认后打开 **Active** 开关。
- **重要**：n8n Public API 无法远程激活工作流（`active` 字段只读），必须人工在编辑器激活。

## 7. 待办 / 已知限制

- 当前工作流在 n8n 实例中为未激活状态，需要人工激活后才会自动运行。
- 内容真实性主要依赖提示词约束，未做发布前人工审核或二次校验；如需更强保障，可考虑增加“发布前人工确认”或“事实核查”节点。
- 查重仅按 slug 比对，未覆盖标题/正文相似度语义查重。

## 8. 文件清单

| 路径 | 说明 |
| --- | --- |
| `workflows/tech-content-wordpress-publish.json` | 工作流定义（可导入） |
| `README.md` | 项目介绍与安装教程 |
| `docs/AI_HANDOVER.md` | 本文档 |