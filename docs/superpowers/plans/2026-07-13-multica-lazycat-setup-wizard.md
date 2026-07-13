# Multica LazyCat Setup Wizard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a LazyCat installation wizard, current Multica self-host environment coverage, optional custom-origin support, production email delivery, and a safe private-install fallback.

**Architecture:** `lzc-deploy-params.yml` collects the one required login email plus optional domain, mail, signup, and OAuth settings. `lzc-manifest.yml` renders those values into the backend environment, selects production mode whenever Resend or SMTP is configured, and otherwise keeps the existing fixed-code private-install path. The package also gains stable generated secrets, the official LazyCat file-picker intercept, current upstream Compose content, and matching operator documentation.

**Tech Stack:** LazyCat LPK v2 YAML, Go `text/template` manifest rendering, Docker Compose, `lzc-cli` 2.0.8, Markdown.

## Global Constraints

- Preserve package ID `community.lazycat.app.multica`.
- Preserve `min_os_version: 1.5.0` or higher.
- Add no new software dependencies.
- Use only `id`, `type`, `name`, `description`, `optional`, `default_value`, and `hidden` inside deploy parameter items.
- Deploy parameter types are limited to `bool`, `lzc_uid`, `string`, and `secret`.
- `login_email` is the only parameter without a default and must use `optional: false`.
- A non-empty `SMTP_HOST` takes precedence over Resend, matching Multica upstream behavior.
- A non-empty `SMTP_HOST` or `RESEND_API_KEY` selects `APP_ENV=production` and disables `MULTICA_DEV_VERIFICATION_CODE`.
- Empty mail-provider settings select `APP_ENV=development` and use the configured private verification code.
- `custom_domain` contains a hostname only, without a scheme or trailing slash.
- Arbitrary external DNS, TLS, and reverse-proxy provisioning remain outside the LPK.
- Keep `COOKIE_DOMAIN` and `MULTICA_PUBLIC_URL` empty for the same-origin layout.
- Package the official file-picker interceptor because Multica supports attachment upload and download.
- All commits must follow the repository Lore commit protocol.

---

### Task 1: Add the installation wizard and rendered backend configuration

**Files:**
- Create: `lzc-deploy-params.yml`
- Modify: `lzc-manifest.yml`
- Modify: `package.yml`

**Interfaces:**
- Consumes: LazyCat system domain from `.S.AppDomain`; operator values from `.U`.
- Produces: backend environment variables, stable database/JWT secrets, and a `net.internet` permission declaration.

- [ ] **Step 1: Record the expected pre-change failures**

Run:

```bash
test -f lzc-deploy-params.yml
rg -q 'SMTP_HOST=' lzc-manifest.yml
rg -q 'stable_secret "jwt_secret"' lzc-manifest.yml
```

Expected: each command exits non-zero because the wizard, SMTP mapping, and generated JWT secret do not exist yet.

- [ ] **Step 2: Create the deploy parameter file**

Create `lzc-deploy-params.yml` with this exact parameter set and matching `zh` translations:

```yaml
params:
  - id: login_email
    type: string
    name: "Login Email"
    description: "Email address allowed to create the first account and sign in"
    optional: false

  - id: custom_domain
    type: string
    name: "Custom Domain"
    description: "External hostname without https://, http://, or a trailing slash. Leave empty to use the LazyCat application domain."
    default_value: ""
    optional: true

  - id: verification_code
    type: secret
    name: "Private Verification Code"
    description: "Fixed login code used only when neither Resend nor SMTP is configured. Do not use this fallback on a publicly reachable instance."
    default_value: "123456"
    optional: true

  - id: allow_signup
    type: bool
    name: "Allow Signup"
    description: "Allow new accounts that match the configured email allowlist"
    default_value: "true"
    optional: true

  - id: allowed_email_domains
    type: string
    name: "Allowed Email Domains"
    description: "Optional comma-separated additional email domains allowed to register"
    default_value: ""
    optional: true

  - id: disable_workspace_creation
    type: bool
    name: "Disable Workspace Creation"
    description: "Prevent all users from creating additional workspaces after initial setup"
    default_value: "false"
    optional: true

  - id: resend_api_key
    type: secret
    name: "Resend API Key"
    description: "Optional Resend API key. When set, production email verification is enabled unless SMTP Host is also set."
    default_value: ""
    optional: true

  - id: resend_from_email
    type: string
    name: "Sender Email"
    description: "Sender address used by Resend or SMTP. Resend requires an address on a verified domain."
    default_value: "noreply@multica.ai"
    optional: true

  - id: smtp_host
    type: string
    name: "SMTP Host"
    description: "Optional SMTP relay hostname. When set, SMTP overrides Resend and production email verification is enabled."
    default_value: ""
    optional: true

  - id: smtp_port
    type: string
    name: "SMTP Port"
    description: "SMTP relay port: 25 for relay, 587 for STARTTLS, or 465 for SMTPS"
    default_value: "25"
    optional: true

  - id: smtp_username
    type: string
    name: "SMTP Username"
    description: "Optional SMTP authentication username"
    default_value: ""
    optional: true

  - id: smtp_password
    type: secret
    name: "SMTP Password"
    description: "Optional SMTP authentication password"
    default_value: ""
    optional: true

  - id: smtp_tls
    type: string
    name: "SMTP TLS Mode"
    description: "Leave empty for STARTTLS, or use implicit for SMTPS on a non-standard port"
    default_value: ""
    optional: true

  - id: smtp_tls_insecure
    type: bool
    name: "Allow Insecure SMTP TLS"
    description: "Skip SMTP certificate verification only for a private CA or self-signed certificate"
    default_value: "false"
    optional: true

  - id: smtp_ehlo_name
    type: string
    name: "SMTP EHLO Name"
    description: "Optional public FQDN announced to strict SMTP relays"
    default_value: ""
    optional: true

  - id: google_client_id
    type: string
    name: "Google Client ID"
    description: "Optional Google OAuth client ID"
    default_value: ""
    optional: true

  - id: google_client_secret
    type: secret
    name: "Google Client Secret"
    description: "Optional Google OAuth client secret"
    default_value: ""
    optional: true

locales:
  zh:
    login_email:
      name: "登录邮箱"
      description: "允许创建首个账号并登录的邮箱地址"
    custom_domain:
      name: "自定义域名"
      description: "外部访问域名，不含 https://、http:// 和末尾斜杠；留空则使用懒猫应用域名"
    verification_code:
      name: "私有验证码"
      description: "仅在未配置 Resend 和 SMTP 时使用的固定登录验证码；公网实例不要使用此模式"
    allow_signup:
      name: "允许注册"
      description: "允许符合邮箱白名单的新账号注册"
    allowed_email_domains:
      name: "允许的邮箱域名"
      description: "可选的额外注册邮箱域名，多个域名使用逗号分隔"
    disable_workspace_creation:
      name: "禁止创建工作区"
      description: "初始化完成后禁止所有用户继续创建工作区"
    resend_api_key:
      name: "Resend API Key"
      description: "可选；填写后启用生产邮件验证码，除非同时填写了 SMTP 主机"
    resend_from_email:
      name: "发件邮箱"
      description: "Resend 或 SMTP 使用的发件地址；Resend 要求该地址属于已验证域名"
    smtp_host:
      name: "SMTP 主机"
      description: "可选 SMTP 中继主机；填写后优先于 Resend 并启用生产邮件验证码"
    smtp_port:
      name: "SMTP 端口"
      description: "25 用于中继，587 用于 STARTTLS，465 用于 SMTPS"
    smtp_username:
      name: "SMTP 用户名"
      description: "可选的 SMTP 认证用户名"
    smtp_password:
      name: "SMTP 密码"
      description: "可选的 SMTP 认证密码"
    smtp_tls:
      name: "SMTP TLS 模式"
      description: "留空使用 STARTTLS；非标准 SMTPS 端口可填写 implicit"
    smtp_tls_insecure:
      name: "允许不安全 SMTP TLS"
      description: "仅私有 CA 或自签证书场景跳过 SMTP 证书校验"
    smtp_ehlo_name:
      name: "SMTP EHLO 名称"
      description: "严格 SMTP 中继要求的可选公网 FQDN"
    google_client_id:
      name: "Google Client ID"
      description: "可选的 Google OAuth 客户端 ID"
    google_client_secret:
      name: "Google Client Secret"
      description: "可选的 Google OAuth 客户端密钥"
```

- [ ] **Step 3: Replace hard-coded secrets and render the selected origin**

In `lzc-manifest.yml`, use:

```yaml
application:
  public_path:
    - /
    - /api

services:
  postgres:
    environment:
      - POSTGRES_DB=multica
      - POSTGRES_USER=multica
      - POSTGRES_PASSWORD={{ stable_secret "postgres_password" }}

  backend:
    environment:
      - DATABASE_URL=postgres://multica:{{ stable_secret "postgres_password" }}@postgres:5432/multica?sslmode=disable
      - JWT_SECRET={{ stable_secret "jwt_secret" }}
      - FRONTEND_ORIGIN={{if .U.custom_domain}}https://{{ .U.custom_domain }}{{else}}https://{{ .S.AppDomain }}{{end}}
      - CORS_ALLOWED_ORIGINS=https://{{ .S.AppDomain }}{{if .U.custom_domain}},https://{{ .U.custom_domain }}{{end}}
      - RESEND_API_KEY={{ .U.resend_api_key }}
      - RESEND_FROM_EMAIL={{ .U.resend_from_email }}
      - SMTP_HOST={{ .U.smtp_host }}
      - SMTP_PORT={{ .U.smtp_port }}
      - SMTP_USERNAME={{ .U.smtp_username }}
      - SMTP_PASSWORD={{ .U.smtp_password }}
      - SMTP_TLS={{ .U.smtp_tls }}
      - SMTP_TLS_INSECURE={{ .U.smtp_tls_insecure }}
      - SMTP_EHLO_NAME={{ .U.smtp_ehlo_name }}
      - GOOGLE_CLIENT_ID={{ .U.google_client_id }}
      - GOOGLE_CLIENT_SECRET={{ .U.google_client_secret }}
      - GOOGLE_REDIRECT_URI={{if .U.custom_domain}}https://{{ .U.custom_domain }}{{else}}https://{{ .S.AppDomain }}{{end}}/auth/callback
      - COOKIE_DOMAIN=
      - APP_ENV={{if or .U.smtp_host .U.resend_api_key}}production{{else}}development{{end}}
      - MULTICA_DEV_VERIFICATION_CODE={{if or .U.smtp_host .U.resend_api_key}}{{else}}{{ .U.verification_code }}{{end}}
      - MULTICA_APP_URL={{if .U.custom_domain}}https://{{ .U.custom_domain }}{{else}}https://{{ .S.AppDomain }}{{end}}
      - ALLOW_SIGNUP={{ .U.allow_signup }}
      - ALLOWED_EMAILS={{ .U.login_email }}
      - ALLOWED_EMAIL_DOMAINS={{ .U.allowed_email_domains }}
      - DISABLE_WORKSPACE_CREATION={{ .U.disable_workspace_creation }}
```

Also add the official defaults and empty opt-ins exactly once:

```yaml
      - S3_BUCKET=
      - S3_REGION=us-west-2
      - AWS_ENDPOINT_URL=
      - S3_USE_PATH_STYLE=
      - AWS_ACCESS_KEY_ID=
      - AWS_SECRET_ACCESS_KEY=
      - ATTACHMENT_DOWNLOAD_MODE=auto
      - ATTACHMENT_DOWNLOAD_URL_TTL=30m
      - CLOUDFRONT_DOMAIN=
      - CLOUDFRONT_KEY_PAIR_ID=
      - CLOUDFRONT_PRIVATE_KEY=
      - GITHUB_APP_SLUG=
      - GITHUB_WEBHOOK_SECRET=
      - MULTICA_PUBLIC_URL=
      - MULTICA_TRUSTED_PROXIES=
      - MULTICA_LARK_SECRET_KEY=
      - MULTICA_LARK_HTTP_BASE_URL=
      - MULTICA_LARK_CALLBACK_BASE_URL=
      - MULTICA_SLACK_SECRET_KEY=
```

- [ ] **Step 4: Declare external network permission**

Append to `package.yml`:

```yaml
permissions:
  required:
    - net.internet
```

- [ ] **Step 5: Validate Task 1**

Run:

```bash
test -f lzc-deploy-params.yml
rg -q 'optional: false' lzc-deploy-params.yml
test "$(rg -c 'optional: false' lzc-deploy-params.yml)" -eq 1
rg -q 'APP_ENV=\{\{if or \.U\.smtp_host \.U\.resend_api_key\}\}production' lzc-manifest.yml
rg -q 'POSTGRES_PASSWORD=\{\{ stable_secret "postgres_password" \}\}' lzc-manifest.yml
rg -q 'JWT_SECRET=\{\{ stable_secret "jwt_secret" \}\}' lzc-manifest.yml
```

Expected: all commands exit zero.

- [ ] **Step 6: Commit Task 1**

Commit the three files with a Lore message whose intent line is:

```text
Make Multica authentication configurable without exposing shared secrets
```

Record the optional-mail constraint, rejected always-production and always-development alternatives, lint status, and any untested runtime behavior in trailers.

---

### Task 2: Add the mandatory LazyCat file-picker integration

**Files:**
- Create: `content/lazycat-injects/lzc-file-chooser-inject.js`
- Modify: `lzc-build.yml`
- Modify: `lzc-manifest.yml`

**Interfaces:**
- Consumes: official asset at `https://developer.lazycat.cloud/lazycat-injects/lzc-file-chooser-inject.js`.
- Produces: a packaged browser injection active on all Multica pages.

- [ ] **Step 1: Confirm the integration is absent**

Run:

```bash
test -f content/lazycat-injects/lzc-file-chooser-inject.js
rg -q 'open-save-chooser' lzc-manifest.yml
rg -q '^contentdir: ./content$' lzc-build.yml
```

Expected: all commands exit non-zero.

- [ ] **Step 2: Fetch and inspect the official asset outside the workspace**

Run:

```bash
curl -fsSL https://developer.lazycat.cloud/lazycat-injects/lzc-file-chooser-inject.js -o /tmp/lzc-file-chooser-inject.js
test -s /tmp/lzc-file-chooser-inject.js
head -n 5 /tmp/lzc-file-chooser-inject.js
sha256sum /tmp/lzc-file-chooser-inject.js
```

Expected: the download is non-empty and JavaScript source is shown. Record the checksum in the commit body.

- [ ] **Step 3: Add the asset through `apply_patch` without modifying its content**

Create `content/lazycat-injects/lzc-file-chooser-inject.js` with byte-for-byte content from `/tmp/lzc-file-chooser-inject.js`. Verify:

```bash
cmp /tmp/lzc-file-chooser-inject.js content/lazycat-injects/lzc-file-chooser-inject.js
```

Expected: exit zero.

- [ ] **Step 4: Package and load the interceptor**

Add to `lzc-build.yml`:

```yaml
contentdir: ./content
```

Add under `application` in `lzc-manifest.yml`:

```yaml
  injects:
    - id: open-save-chooser
      on: browser
      when:
        - "/*"
      do:
        - src: file:///lzcapp/pkg/content/lazycat-injects/lzc-file-chooser-inject.js
```

- [ ] **Step 5: Validate Task 2**

Run:

```bash
test -s content/lazycat-injects/lzc-file-chooser-inject.js
rg -q '^contentdir: ./content$' lzc-build.yml
rg -q 'file:///lzcapp/pkg/content/lazycat-injects/lzc-file-chooser-inject.js' lzc-manifest.yml
git diff --check
```

Expected: all commands exit zero.

- [ ] **Step 6: Commit Task 2**

Commit the asset, build config, and manifest with a Lore intent line:

```text
Keep Multica file flows compatible with LazyCat storage
```

Record the App Store interception requirement, official asset checksum, and browser/runtime verification gap.

---

### Task 3: Synchronize the official Docker Compose reference

**Files:**
- Modify: `docker-compose.selfhost.yml`

**Interfaces:**
- Consumes: `https://raw.githubusercontent.com/multica-ai/multica/main/docker-compose.selfhost.yml` and the Compose supplied in the user request.
- Produces: a local reference Compose identical to current upstream.

- [ ] **Step 1: Demonstrate the local Compose is stale**

Run:

```bash
curl -fsSL https://raw.githubusercontent.com/multica-ai/multica/main/docker-compose.selfhost.yml -o /tmp/multica-compose-upstream.yml
cmp /tmp/multica-compose-upstream.yml docker-compose.selfhost.yml
```

Expected: `cmp` exits non-zero.

- [ ] **Step 2: Replace the local reference using `apply_patch`**

Make `docker-compose.selfhost.yml` byte-for-byte identical to `/tmp/multica-compose-upstream.yml`. The resulting file must include:

```yaml
    ports:
      - "127.0.0.1:${BACKEND_PORT:-8080}:8080"
```

for the backend, this frontend binding:

```yaml
    ports:
      - "127.0.0.1:${FRONTEND_PORT:-3000}:3000"
```

and no PostgreSQL `ports` entry.

- [ ] **Step 3: Validate Task 3**

Run:

```bash
cmp /tmp/multica-compose-upstream.yml docker-compose.selfhost.yml
docker compose -f docker-compose.selfhost.yml config >/tmp/multica-compose-rendered.yml
test -s /tmp/multica-compose-rendered.yml
```

Expected: all commands exit zero.

- [ ] **Step 4: Commit Task 3**

Commit the Compose file with a Lore intent line:

```text
Keep the packaging reference aligned with Multica self-host security defaults
```

Record the loopback-binding constraint, upstream URL, exact comparison, and Compose validation.

---

### Task 4: Update operator documentation

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: the final wizard IDs and behavior from Tasks 1-3.
- Produces: installation, authentication, domain, CLI, and security guidance matching the package.

- [ ] **Step 1: Confirm stale guidance exists**

Run:

```bash
rg -n '邮箱随意|cloud\.lazycat\.app\.multica-v' README.md
```

Expected: both stale instructions are reported.

- [ ] **Step 2: Replace login and domain guidance**

Document these exact behaviors:

```markdown
## 安装向导

- 登录邮箱为必填项，首次注册只能使用安装时填写的邮箱。
- 未配置 Resend 或 SMTP 时，应用使用安装向导中的私有验证码；该模式只适合私有网络。
- 配置 `RESEND_API_KEY` 或 `SMTP_HOST` 后，后端自动切换到 production，并发送真实邮件验证码。
- SMTP 配置优先于 Resend。
- 自定义域名只填写主机名，不含协议；DNS、TLS 和反向代理需在外部完成。
```

Update CLI examples to show both the LazyCat domain and custom-domain command:

```bash
multica setup self-host \
  --server-url https://multica.example.com \
  --app-url https://multica.example.com
```

Update the LPK installation filename to use `community.lazycat.app.multica`.

- [ ] **Step 3: Validate Task 4**

Run:

```bash
! rg -q '邮箱随意|cloud\.lazycat\.app\.multica-v' README.md
rg -q '登录邮箱为必填项' README.md
rg -q 'SMTP 配置优先于 Resend' README.md
rg -q 'community\.lazycat\.app\.multica' README.md
git diff --check
```

Expected: all commands exit zero.

- [ ] **Step 4: Commit Task 4**

Commit `README.md` with a Lore intent line:

```text
Make Multica installation choices explicit for operators
```

Record the production-mail warning, external-domain boundary, and documentation-only verification.

---

### Task 5: Run release-grade verification

**Files:**
- Verify: `package.yml`
- Verify: `lzc-build.yml`
- Verify: `lzc-build.dev.yml`
- Verify: `lzc-deploy-params.yml`
- Verify: `lzc-manifest.yml`
- Verify: `docker-compose.selfhost.yml`
- Verify: `content/lazycat-injects/lzc-file-chooser-inject.js`
- Verify: `README.md`

**Interfaces:**
- Consumes: all deliverables from Tasks 1-4.
- Produces: lint, package, asset-presence, and repository-cleanliness evidence.

- [ ] **Step 1: Parse all YAML files**

Run:

```bash
ruby -e 'require "yaml"; ARGV.each { |f| YAML.parse_file(f); puts "OK #{f}" }' \
  package.yml lzc-build.yml lzc-build.dev.yml lzc-deploy-params.yml docker-compose.selfhost.yml
```

Expected: one `OK` line per file. Do not parse the templated manifest as plain YAML before rendering because Go template control expressions are not YAML tokens.

- [ ] **Step 2: Run LazyCat lint for release and development configs**

Run:

```bash
lzc-cli project lint .
lzc-cli project lint . -f lzc-build.dev.yml
```

Expected: both commands complete without errors. Existing external-image App Store warnings are acceptable and must be reported.

- [ ] **Step 3: Build and inspect the LPK**

Run:

```bash
rm -f /tmp/community.lazycat.app.multica-test.lpk
lzc-cli project release . -o /tmp/community.lazycat.app.multica-test.lpk
lzc-cli lpk info /tmp/community.lazycat.app.multica-test.lpk
```

Expected: release succeeds and package info reports package ID `community.lazycat.app.multica`.

- [ ] **Step 4: Inspect package contents**

Run:

```bash
tar -tf /tmp/community.lazycat.app.multica-test.lpk | rg 'lzc-deploy-params.yml|lzc-file-chooser-inject.js|package.yml|manifest'
```

Expected: deploy parameters, file-picker asset, package metadata, and manifest entries are all listed.

- [ ] **Step 5: Check deploy parameter requirements and repository state**

Run:

```bash
test "$(rg -c 'optional: false' lzc-deploy-params.yml)" -eq 1
rg -n -B4 -A1 'optional: false' lzc-deploy-params.yml
git diff --check
git status --short
```

Expected: the single required parameter is `login_email`, diff check passes, and only intentional plan-tracking changes remain.

- [ ] **Step 6: Commit verification-only fixes if required**

If verification required edits, commit only those edits with a Lore intent line describing the failure prevented. If no edits were required, do not create an empty commit.
