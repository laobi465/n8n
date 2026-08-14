# 项目开发文档 — 技术内容自动发布 WordPress

| 项目 | 内容 |
|------|------|
| 文档版本 | v1.0 |
| 关联文档 | 技术方案（docs/TECHNICAL.md）、环境文档（docs/ENVIRONMENT.md） |

---

## 1. 开发环境要求

- 一个可访问的 n8n 实例（本地自托管或云端）。
- 可访问 `https://word.apexck.com` 的 WordPress 站点。
- 可用的 DeepSeek API Key。

详见 [环境文档](ENVIRONMENT.md)。

---

## 2. 开发流程

### 2.1 编辑工作流
1. 在 n8n 编辑器中打开工作流。
2. 修改节点参数（选题、提示词、URL、频率等）。
3. 点击 **Execute workflow** 手动测试。

### 2.2 导出工作流
测试通过后，在工作流右上角 **... → Download** 导出 JSON 文件，覆盖 `workflows/tech-content-wordpress-publish.json`。

### 2.3 代码规范
- Code 节点使用 JavaScript（n8n Code Node 环境）。
- 关键逻辑必须兼容 n8n 沙箱限制（**禁止**在 Code 节点内发起 HTTP 请求，需用 HTTP Request 节点）。

---

## 3. 目录结构

```
laobi465/n8n/
├── workflows/
│   └── tech-content-wordpress-publish.json   # 工作流定义（唯一可执行文件）
├── docs/
│   ├── AI_HANDOVER.md      # AI 智能体交接文档
│   ├── PRD.md              # 产品需求文档
│   ├── TECHNICAL.md        # 技术方案文档
│   ├── DEVELOPMENT.md      # 项目开发文档（本文档）
│   ├── PROJECT.md          # 项目文档（项目介绍）
│   ├── OPS.md              # 运维文档
│   ├── UPDATE.md           # 更新文档
│   └── ENVIRONMENT.md      # 开发/生产环境文档
└── README.md               # 项目总览 + 安装教程 + 文档索引
```

---

## 4. 各节点开发要点

### 4.1 Code - 选题
- 主题数组 `themes`，每项含 `{category, instruction}`。
- 新增主题：在数组末尾追加对象即可。

### 4.2 AI Agent 提示词
- 位于节点 options 的 systemMessage。
- 修改需保持“输出严格 JSON”与“内容真实性”两个约束。

### 4.3 Code - 解析AI输出
- 兼容三种输入：纯 JSON、markdown ```json 代码块、带首尾花括号的字符串。
- 解析失败会 throw Error，n8n 标记节点失败。

### 4.4 Code - 判断重复
- 使用 `$('Code - 解析AI输出').all()[0].json` 跨节点引用。
- 输出 `isDuplicate` 布尔值。

### 4.5 Code - 准备术语
- `defaults` 数组定义默认标签（技术教程、实用技巧、网站运维）。
- 逻辑：去重 AI 标签，不足 3 个时依次补齐。

### 4.6 ID 解析（Code - 取得分类ID / 重组标签ID）
- 兼容多种响应结构：`id`、`term_id`、`data.term_id`、`error.data.term_id`。
- 无法取得 ID 时 throw Error。

---

## 5. 开发中常见坑（重要）

1. **Code 节点不能发 HTTP**：所有 API 调用必须用 HTTP Request 节点。
2. **HTTP 空数组断流**：设置 `alwaysOutputData: true`。
3. **避免双输入 Merge**：IF 分支只走一条，双输入 Merge 会等待导致卡死。用单路径 + 跨节点引用。
4. **WordPress 只接受术语 ID**：`categories`/`tags` 必须是数字数组，不能是名称字符串。
5. **分类/标签已存在的冲突**：创建接口 `continueOnFail: true`，从错误响应解析已有 term_id。

---

## 6. 版本与分支规范

- 主干分支：`main`。
- 功能分支：`feature/*`（如 `feature/tech-content-wordpress-autopublish`）。
- 提交信息遵循 Conventional Commits（`feat:` / `fix:` / `docs:` 等）。

---

## 7. 测试清单（发布到 GitHub 前）

- [ ] 工作流 JSON 是合法 JSON（可用 `python3 json.tool` 校验）。
- [ ] 手动执行一次成功发布文章。
- [ ] 分类/标签正确创建或复用。
- [ ] 重复 slug 不重复发布。
- [ ] 已知边界场景已覆盖（见 TECHNICAL.md 第 5 章）。