# Multica LazyCat App

Multica 将编码 Agent 变成真正的队友。像分配给同事一样分配给 Agent——它们会自主接手工作、编写代码、报告阻塞问题、更新状态。

本仓库为 [Multica](https://multica.ai) 的懒猫微服平台打包与部署配置。

## 服务架构

| 服务 | 镜像 | 端口 |
|------|------|------|
| postgres | pgvector/pgvector:pg17 | 5432 |
| backend | multica-ai/multica-backend | 8080 |
| frontend | multica-ai/multica-web | 3000 |

## 快速上手

1. 访问站点，注册账号——邮箱随意，邮箱验证码为 `123456`。

2. 安装 Multica CLI：

```bash
curl -fsSL https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.sh | bash
```

3. 配置（替换为你的专属域名）：

```bash
multica config set server_url https://multica.<your_box>.heiyu.space
multica config set app_url https://multica.<your_box>.heiyu.space
```

4. 认证登录：

```bash
multica login
```

5. 启动 daemon：

```bash
multica daemon start
```

## 开发模式

使用 `lzc-build.dev.yml` 进行开发构建（包名 `cloud.lazycat.app.multica.dev`，`DEV_MODE=1`）。

## 更新镜像与重新构建 LPK

当 multica-backend 或 multica-web 发布新版本时（版本号见 [Releases](https://github.com/multica-ai/multica/releases)）：

1. 复制镜像到懒猫私仓：

```bash
lzc-cli appstore copy-image ghcr.io/multica-ai/multica-backend:v0.2.27
lzc-cli appstore copy-image ghcr.io/multica-ai/multica-web:v0.2.27
```

2. 更新 `lzc-manifest.yml`——替换对应服务的镜像 digest/tag。

3. 更新 `package.yml`——递增 `version` 字段。

4. 构建发布包：

```bash
lzc-cli project release
```

5. 安装到懒猫微服：

```bash
lzc-cli lpk install ./cloud.lazycat.app.multica-v0.2.27.lpk
```
