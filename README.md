# n8n 工作流合集

基于 [n8n](https://n8n.io) 的自动化工作流仓库，收录日常用到的可复用工作流。

## 目录

- [技术内容自动发布 WordPress](#技术内容自动发布-wordpress) — 自动选题、AI 生成、自动分类打标签、查重、发布到 WordPress

---

## 技术内容自动发布 WordPress

面向技术类网站的自动内容发布工作流。每 6 小时自动从 6 大主题中随机选题，由 DeepSeek 生成一篇 SEO/GEO 优化的原创技术文章，自动创建/复用分类与标签，查重后发布到 WordPress。

### 覆盖主题

| 主题 | 说明 |
| --- | --- |
| 开源程序教程 | 知名开源程序的安装、配置、使用教程 |
| 主机测评 | 主机/VPS 选购指南与测评（强调真实、不虚构数据） |
| 服务器运维 | Linux 命令、巡检、安全加固、备份、日志等实战 |
| WordPress 优化 | 性能、缓存、SEO、安全、插件优化教程 |
| 在线工具教程 | 实用在线工具的功能与用法 |
| Python/PHP 入门 | 环境搭建、基础语法、常用库/框架、小实战 |

### 工作流链路

```
Schedule Trigger (每6小时)
  → Code - 选题（6主题随机）
  → AI Agent（DeepSeek 生成 JSON）
  → Code - 解析AI输出
  → HTTP Request - 查重（按 slug 检测重复）
  → Code - 判断重复
  → IF - 是否重复
      ├─ 重复 → 结束（不发布）
      └─ 非重复 → Code - 准备术语（补齐默认标签）
            → HTTP - 创建分类 → Code - 取得分类ID
            → Code - 展开标签 → HTTP - 创建标签 → Code - 重组标签ID
            → Code - 组装发布参数
            → HTTP - 创建文章（发布到 WordPress REST API）
```

### 功能特性

- **自动选题**：6 大主题随机轮换，保持内容多样。
- **AI 生成**：DeepSeek 撰写原创技术文章，做 SEO/GEO 优化，输出结构化 JSON。
- **内容真实性约束**：提示词强制“不得编造版本号/价格/跑分/配置、以官方文档/官网为准”。
- **自动分类与标签**：分类/标签不存在时自动创建，存在时复用；标签不足时自动补齐默认标签（技术教程、实用技巧、网站运维），保证标签始终 ≥ 3 个。
- **重复文章检测**：按 slug 与已发布文章比对，重复则跳过，避免重复发布。
- **断流防护**：HTTP 空数组响应、空标签等场景均不会导致链路中断。

### 文件位置

| 文件 | 说明 |
| --- | --- |
| [`workflows/tech-content-wordpress-publish.json`](./workflows/tech-content-wordpress-publish.json) | 可导入的工作流文件 |

---

## 文档位置

本仓库在 `docs/` 目录下提供一套完整文档：

| 文档 | 路径 | 说明 |
| --- | --- | --- |
| 项目文档 | [`docs/PROJECT.md`](./docs/PROJECT.md) | 项目介绍、价值、特性、技术栈 |
| PRD 文档 | [`docs/PRD.md`](./docs/PRD.md) | 产品需求、功能需求、验收标准 |
| 技术方案文档 | [`docs/TECHNICAL.md`](./docs/TECHNICAL.md) | 架构、节点设计、数据流、边界处理 |
| 项目开发文档 | [`docs/DEVELOPMENT.md`](./docs/DEVELOPMENT.md) | 开发环境、流程、规范、常见坑 |
| AI 智能体交接文档 | [`docs/AI_HANDOVER.md`](./docs/AI_HANDOVER.md) | 交接给 AI 智能体/后续维护者 |
| 运维文档 | [`docs/OPS.md`](./docs/OPS.md) | 部署、监控、备份、故障排查 |
| 更新文档 | [`docs/UPDATE.md`](./docs/UPDATE.md) | 更新流程、版本历史、变更指引 |
| 环境文档 | [`docs/ENVIRONMENT.md`](./docs/ENVIRONMENT.md) | 开发/生产环境介绍 + 教程 |

---

## 安装教程

### 前置要求

- 一个可访问的 n8n 实例（云端或自托管）。
- 以下凭据（Credential）：
  - **DeepSeek API**：用于 AI 生成，需在 n8n 中创建 `DeepSeek` 类型凭据。
  - **WordPress**：用于发布文章，需具备 WordPress 用户名 + 应用密码（Application Password），在 n8n 中创建 `WordPress` 类型凭据。

### 第一步：准备 WordPress 应用密码

1. 登录你的 WordPress 后台。
2. 进入 **用户 → 个人资料（Profile）**。
3. 滚动到 **应用程序密码（Application Passwords）** 区域。
4. 输入名称（如 `n8n`），点击 **新增应用程序密码**。
5. 复制生成的应用密码（格式如 `xxxx xxxx xxxx xxxx xxxx xxxx`）。

> 应用密码仅显示一次，请妥善保存。

### 第二步：创建 n8n 凭据

**DeepSeek 凭据：**
1. 在 n8n 左侧导航进入 **凭据（Credentials）**。
2. 点击 **新增凭据**，搜索并选择 **DeepSeek**。
3. 填入你的 DeepSeek API Key。
4. 保存。

**WordPress 凭据：**
1. 在 n8n 左侧导航进入 **凭据（Credentials）**。
2. 点击 **新增凭据**，搜索并选择 **WordPress**。
3. 填写 WordPress 站点地址（如 `https://word.apexck.com`）、用户名、步骤一中生成的应用密码。
4. 保存。

### 第三步：导入工作流

**方式一：通过 n8n 编辑器导入**

1. 下载本仓库的 [`workflows/tech-content-wordpress-publish.json`](./workflows/tech-content-wordpress-publish.json)。
2. 打开 n8n，点击右上角 **Workflows**，再点击 **Import from File**。
3. 选择刚才下载的 JSON 文件。
4. 导入后，打开工作流，为以下节点绑定凭据：
   - **DeepSeek Chat Model** → 选择你创建的 DeepSeek 凭据。
   - **HTTP - 创建分类 / HTTP - 创建标签 / HTTP - 创建文章** → 选择你创建的 WordPress 凭据。

**方式二：通过 n8n CLI 导入（自托管）**

```bash
n8n import:workflow --input=workflows/tech-content-wordpress-publish.json
```

### 第四步：修改站点地址（可选）

工作流中的 WordPress 地址默认指向 `https://word.apexck.com`。若你的站点不同，请修改以下节点的 URL：

- `HTTP Request - 查重`
- `HTTP - 创建分类`
- `HTTP - 创建标签`
- `HTTP - 创建文章`

将域名替换成你的站点域名即可。

### 第五步：测试与激活

1. 点击工作流右上角 **Execute workflow** 手动运行一次。
2. 确认文章成功发布到 WordPress、分类和标签正确创建。
3. 确认无误后，点击右上角 **Active** 开关激活工作流，进入每 6 小时自动发布。

### 发布频率调整

工作流的触发节点为 **Schedule Trigger**，当前为每 6 小时一次。如需调整：

1. 双击 **Schedule Trigger** 节点。
2. 修改 **Hours** 间隔的数值（如改成 `12` 表示每 12 小时一篇）。
3. 保存并重新激活工作流。

---

## 维护与故障排查

- 工作流未执行：确认工作流已 **Active**，且依赖的凭据已正确绑定。
- 发布失败：查看 n8n 最近的执行详情（Executions），定位失败节点与错误信息。
- 更多细节见 [`docs/AI_HANDOVER.md`](./docs/AI_HANDOVER.md)。

## 许可

本仓库内容仅供学习与个人使用，请遵守 n8n 及第三方服务的相关条款。