# 环境文档 — 开发环境与生产环境介绍 + 教程

| 项目 | 内容 |
|------|------|
| 文档版本 | v1.0 |
| 关联文档 | 开发文档（docs/DEVELOPMENT.md）、运维文档（docs/OPS.md） |

---

## 1. 环境总览

本工作流依赖三部分环境：
1. **n8n 实例**：工作流引擎。
2. **DeepSeek API**：内容生成模型。
3. **WordPress 站点**：内容发布目标。

开发环境与生产环境可共用同一套 n8n，也可分离（推荐生产用独立实例）。

---

## 2. n8n 环境

### 2.1 开发环境
- **用途**：编辑、测试工作流。
- **要求**：可访问外网、具备局域网访问能力。
- **本地运行（Docker）**：
  ```bash
  docker run -it --rm \
    --name n8n-dev \
    -p 5678:5678 \
    -v n8n_data:/home/node/.n8n \
    -e GENERIC_TIMEZONE="Asia/Shanghai" \
    -e TZ="Asia/Shanghai" \
    n8nio/n8n
  ```
  访问 `http://localhost:5678`。

### 2.2 生产环境
- **用途**：7×24 自动运行工作流。
- **要求**：稳定、持久化存储、自动重启。
- **推荐**：云端 VPS + Docker 或 n8n Cloud。
- **Docker Compose 示例**：
  ```yaml
  version: "3.8"
  services:
    n8n:
      image: n8nio/n8n
      restart: unless-stopped
      ports:
        - "5678:5678"
      environment:
        - GENERIC_TIMEZONE=Asia/Shanghai
        - TZ=Asia/Shanghai
      volumes:
        - n8n_data:/home/node/.n8n
  volumes:
    n8n_data:
  ```

### 2.3 获取 n8n 链接与 API（可选）
- n8n 编辑器地址：`http://<host>:5678`。
- 如需 API 管理，配置 `N8N_API_URL` 与 `N8N_API_KEY`（面板 → Settings → API）。

---

## 3. DeepSeek 环境

### 3.1 获取 API Key
1. 前往 [DeepSeek 开放平台](https://platform.deepseek.com) 注册/登录。
2. 创建 API Key。
3. 在 n8n 创建 **DeepSeek** 类型凭据，填入 Key。

### 3.2 注意事项
- API Key 属敏感信息，仅存于 n8n 凭据系统。
- 关注调用额度与费用，避免欠费导致工作流失败。

---

## 4. WordPress 环境

### 4.1 站点要求
- 启用 **REST API**（默认启用）。
- 账号具备文章发布、分类/标签创建权限。

### 4.2 创建应用密码
1. 登录 WordPress 后台 → **用户 → 个人资料**。
2. 滚动到 **应用程序密码** → 输入名称（如 `n8n`）→ **新增应用程序密码**。
3. 复制生成的应用密码（格式如 `xxxx xxxx xxxx xxxx xxxx xxxx`）。

> 应用密码仅显示一次，请妥善保存。

### 4.3 创建 n8n WordPress 凭据
1. n8n → **凭据** → 新增 → **WordPress**。
2. 填写：站点地址（如 `https://word.apexck.com`）、用户名、应用密码。
3. 保存。

---

## 5. 本工作流依赖的站点地址

以下节点默认指向 `https://word.apexck.com`，切换站点需同步修改：
- `HTTP Request - 查重`
- `HTTP - 创建分类`
- `HTTP - 创建标签`
- `HTTP - 创建文章`

---

## 6. 环境变量参考（自托管 n8n）

| 变量 | 说明 | 示例 |
|------|------|------|
| `GENERIC_TIMEZONE` | 时区 | `Asia/Shanghai` |
| `TZ` | 容器时区 | `Asia/Shanghai` |
| `N8N_API_URL` | API 地址（可选） | `http://localhost:5678/api/v1` |
| `N8N_API_KEY` | API Key（可选） | `n8n_api_xxx` |

---

## 7. 环境对比表

| 维度 | 开发环境 | 生产环境 |
|------|----------|----------|
| 目的 | 编辑/测试 | 自动运行 |
| n8n 实例 | 本地/临时 | 独立稳定实例 |
| 持久化 | 可临时 | 必须持久化并自动重启 |
| 工作流状态 | 未激活 | 已激活 |
| 凭据 | 测试凭据 | 正式凭据 |
| 监控 | 无 | Executions 定期检查 |