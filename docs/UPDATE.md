# 更新文档 — 技术内容自动发布 WordPress

| 项目 | 内容 |
|------|------|
| 文档版本 | v1.0 |
| 关联文档 | 开发文档（docs/DEVELOPMENT.md）、运维文档（docs/OPS.md） |

---

## 1. 更新流程概述

本工作流的“更新”分为两类：
1. **工作流变更**：修改节点、提示词、频率、站点等，需导出 JSON 并更新仓库。
2. **文档更新**：新增/修改 `/docs` 下的文档。

---

## 2. 更新操作流程

### 2.1 修改工作流并提交
1. 在 n8n 编辑器中修改工作流。
2. 手动测试通过。
3. 导出 JSON，覆盖 `workflows/tech-content-wordpress-publish.json`。
4. 提交到 GitHub（Conventional Commits）。

### 2.2 版本标记
- 采用 Git 标签管理版本，如 `v1.0.0`。
- 大改动（如新增功能）升主版本号。

---

## 3. 版本历史

| 版本 | 日期 | 变更内容 | 提交信息 |
|------|------|----------|----------|
| v1.0.0 | 2026-08 | 创建工作流：选题 → AI 生成 → 查重 → 分类/标签 → 发布 | `feat: 技术类内容自动发布 WordPress 工作流` |
| v1.1.0 | 2026-08-14 | 内容语言从中文改为英文，面向海外站点；选题主题、AI 提示词、默认标签、默认分类、错误消息全部英文化；新增完整文档集（PRD/技术方案/开发/运维/更新/环境/项目/AI 交接） | `feat: 工作流内容语言改为英语` / `docs: 补充完整项目文档集` |

---

## 4. 常见配置变更指引

### 4.1 更改发布频率
- 修改 **Schedule Trigger** 的 `hoursInterval`。
- 例如改为 `12` 表示每 12 小时 1 篇。

### 4.2 更改站点地址
以下 4 个节点包含域名 `word.apexck.com`，需一并修改：
- `HTTP Request - 查重`
- `HTTP - 创建分类`
- `HTTP - 创建标签`
- `HTTP - 创建文章`

并将 WordPress 凭据站点地址改为新站点。

### 4.3 新增主题
在 **Code - 选题** 的 `themes` 数组追加对象：
```js
{ category: '新主题名', instruction: '写作指令' }
```

### 4.4 修改默认标签
在 **Code - 准备术语** 的 `defaults` 数组修改：
```js
const defaults = ['Tech Tutorials', 'Practical Tips', 'Web Hosting'];
```

### 4.5 修改 AI 提示词
在 **AI Agent** 节点的 systemMessage 中修改，保持“严格 JSON 输出”与“内容真实性”约束。

---

## 5. 变更记录提交规范

- 使用 Conventional Commits：
  - `feat:` 新功能
  - `fix:` 修复
  - `docs:` 文档
  - `chore:` 维护
- 示例：
  ```
  feat: 新增主题“Kubernetes 入门”
  fix: 修复标签为空时断流问题
  docs: 补充运维文档
  ```

---

## 6. 回滚

- 若某次更新导致工作流异常：
  1. 从仓库 `git log` 找到上一个可用版本。
  2. 恢复 `workflows/tech-content-wordpress-publish.json`。
  3. 重新导入 n8n 并测试激活。