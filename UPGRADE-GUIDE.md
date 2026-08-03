# grok2api 本地部署升级/维护指南

> 本文档说明本项目的自定义文件和上游文件，方便后续更新项目时知道哪些不能覆盖、哪些需要保留。

## 项目部署方式

本项目使用**独立的本地 Compose 配置**运行，而不是直接使用仓库自带的 `docker-compose.yml`。这样做的目的是：

- 不污染 Docker 全局环境
- 数据全部保存在 `D:\docker-apps\grok2api\data`
- 端口、内存等配置通过 `.env` 管理
- 升级上游代码时不会误覆盖本地配置

---

## 重要：这些文件/目录绝对不能覆盖

### 1. `config.yaml` ⭐⭐⭐

**作用**：grok2api 的核心配置文件，包含：

- JWT 密钥
- 凭据加密密钥
- 管理员账号密码
- 数据库、运行时存储、审计等关键配置

**风险**：如果覆盖，会导致：

- 已保存的账号凭据无法解密
- 管理员密码丢失
- 服务无法启动或安全令牌失效

**建议**：

- 永远不要删除或覆盖
- 定期备份到安全位置
- 升级前必备份

---

### 2. `.env`

**作用**：本地 Docker 部署的环境变量：

```env
GROK2API_PORT=8000
GROK2API_IMAGE=ghcr.io/chenyme/grok2api:latest
TZ=Asia/Shanghai
```

**风险**：覆盖后端口、镜像地址可能恢复默认值，导致端口冲突或拉取错误镜像。

---

### 3. `docker-compose.local.yml` ⭐⭐⭐

**作用**：本项目的**实际运行配置**，定义了：

- 容器名称 `grok2api`
- 端口映射
- 数据目录绑定到 `D:\docker-apps\grok2api\data`
- 内存限制
- 安全配置

**风险**：如果误用上游的 `docker-compose.yml` 启动，会：

- 使用默认的命名卷（数据位置不可控）
- 端口可能被其他项目占用
- 不再隔离于 `D:\docker-apps\grok2api`

**建议**：

- 日常启动/停止/重启都用这个文件
- 不要把它和仓库的 `docker-compose.yml` 混淆

---

### 4. `data/` 目录 ⭐⭐⭐

**作用**：所有持久化数据，包括：

- `backend.db`：SQLite 数据库（账号、配置、审计记录等）
- `media/`：生成的图片、视频等媒体文件

**风险**：删除或覆盖会导致所有账号、历史记录、媒体丢失。

**建议**：

- 定期备份整个 `data/` 目录
- 升级前必须备份

---

### 5. 自定义辅助文件

以下文件是本部署过程中为方便使用生成的，建议保留：

| 文件 | 作用 |
|------|------|
| `clash-egress-nodes.md` | Clash 代理节点与 grok2api 的对应关系表 |
| `clash-egress-nodes.csv` | 代理节点 CSV，方便查看 |
| `clash-egress-urls.txt` | 可直接批量导入 grok2api 的代理 URL |
| `cookie-to-sso-tool.html` | 本地 Cookie 转 SSO Token 工具 |
| `rename_egress.sql` | 节点重命名 SQL 脚本（历史记录） |

---

## 可以更新的上游文件

以下文件来自 GitHub 仓库 `https://github.com/chenyme/grok2api`，升级时可以安全覆盖：

```
Dockerfile
Makefile
VERSION
docker/
backend/
frontend/
.github/
backend/README.md
frontend/README.md
```

以及上游默认配置示例：

```
docker-compose.yml      ← 可以覆盖，但我们不用它启动
config.example.yaml     ← 可以覆盖，只是参考模板
```

> ⚠️ 注意：`config.example.yaml` 可以覆盖，但 `config.yaml` 绝对不能覆盖。

---

## 安全升级步骤

### 升级前

1. 停止容器：

    ```powershell
    cd D:\docker-apps\grok2api
    docker compose -f docker-compose.local.yml down
    ```

2. 备份关键文件：

    ```powershell
    Copy-Item config.yaml config.yaml.bak
    Copy-Item .env .env.bak
    Copy-Item docker-compose.local.yml docker-compose.local.yml.bak
    Copy-Item data\backend.db data\backend.db.bak
    ```

3. 备份整个 `data/` 目录（推荐）：

    ```powershell
    Copy-Item -Path data -Destination data-backup -Recurse -Force
    ```

### 升级上游代码

```powershell
cd D:\docker-apps\grok2api
git pull
```

> 如果 `git pull` 提示本地文件冲突，优先保留 `config.yaml`、`.env`、`docker-compose.local.yml`，其他文件可以按仓库版本更新。

### 升级后

1. 拉取最新镜像：

    ```powershell
    docker compose -f docker-compose.local.yml pull
    ```

2. 启动服务：

    ```powershell
    docker compose -f docker-compose.local.yml up -d
    ```

3. 检查状态：

    ```powershell
    docker compose -f docker-compose.local.yml ps
    docker compose -f docker-compose.local.yml logs --tail 30
    ```

---

## 常见问题

### Q: 升级后账号不见了？

A: 大概率是启动时用了上游的 `docker-compose.yml`，数据卷没有挂载到 `D:\docker-apps\grok2api\data`。请立即停止，用 `docker-compose.local.yml` 启动。

### Q: 升级后提示凭据解密失败？

A: 可能是 `config.yaml` 里的 `credentialEncryptionKey` 被覆盖了。从 `config.yaml.bak` 恢复。

### Q: 升级后端口冲突？

A: 检查 `.env` 文件里的 `GROK2API_PORT` 是否被覆盖。从 `.env.bak` 恢复。

### Q: 新版本的 `config.example.yaml` 有新增配置项怎么办？

A: 打开 `config.example.yaml`，把新增的配置项手动合并到 `config.yaml` 中，不要直接覆盖 `config.yaml`。

---

## 一句话总结

> **保留 `config.yaml`、`.env`、`docker-compose.local.yml`、`data/`，其他都可以从上游更新。**
