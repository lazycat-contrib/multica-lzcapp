# Multica LazyCat App

Multica 将编码 Agent 变成真正的队友。像分配给同事一样分配给 Agent——它们会自主接手工作、编写代码、报告阻塞问题、更新状态。

本仓库为 [Multica](https://multica.ai) 的懒猫微服平台打包与部署配置。

## 服务架构

| 服务 | 镜像 | 端口 |
|------|------|------|
| postgres | pgvector/pgvector:pg17 | 5432 |
| backend | multica-ai/multica-backend | 8080 |
| frontend | multica-ai/multica-web | 3000 |

## 安装向导

- 登录邮箱为必填项，首次注册只能使用安装时填写的邮箱。
- 未配置 Resend 或 SMTP 时，应用使用安装向导中的私有验证码；该模式只适合私有网络。
- 配置 `RESEND_API_KEY` 或 `SMTP_HOST` 后，后端自动切换到 production，并发送真实邮件验证码。
- SMTP 配置优先于 Resend。
- 自定义域名只填写主机名，不含协议；DNS、TLS 和反向代理需在外部完成。

## 快速上手

1. 完成安装向导后，访问站点，使用安装时填写的登录邮箱注册。如未配置 Resend 或 SMTP，请使用安装向导中设置的私有验证码。

2. 安装 Multica CLI：

```bash
curl -fsSL https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.sh | bash
```

3. 使用懒猫应用域名配置 CLI：

```bash
multica setup self-host \
  --server-url https://multica.<your_box>.heiyu.space \
  --app-url https://multica.<your_box>.heiyu.space
```

如安装向导中配置了自定义域名，并已在外部完成 DNS、TLS 和反向代理，则使用该域名：

```bash
multica setup self-host \
  --server-url https://multica.example.com \
  --app-url https://multica.example.com
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
lzc-cli lpk install ./community.lazycat.app.multica-v0.3.43.lpk
```
