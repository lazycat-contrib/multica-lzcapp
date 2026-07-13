# Multica LazyCat Setup Wizard Design

## Objective

Bring the LazyCat package in line with Multica's current official self-hosting configuration while preserving an install-and-use default path. Installation must collect the login email, support an optional external hostname, and allow operators to move from a private fixed-code login flow to production email delivery through Resend or SMTP.

## Chosen approach

Use the hybrid setup model:

- A login email is always required and is passed to `ALLOWED_EMAILS`.
- Resend and SMTP are optional during installation.
- Without either mail provider, Multica runs in a non-production LazyCat mode with a configurable fixed verification code so the owner can sign in immediately.
- When `RESEND_API_KEY` or `SMTP_HOST` is configured, Multica runs with `APP_ENV=production` and the fixed verification code is disabled.
- SMTP takes precedence over Resend, matching Multica upstream behavior.

This preserves the existing package's zero-external-service experience while giving public deployments a supported production path. The setup descriptions and README must warn that fixed verification codes are unsuitable for publicly reachable instances.

## Setup wizard

Create `lzc-deploy-params.yml` using only official LazyCat fields and types.

| Parameter | Type | Required | Default | Purpose |
| --- | --- | --- | --- | --- |
| `login_email` | `string` | yes | none | Email used for first registration and login; mapped to `ALLOWED_EMAILS` |
| `custom_domain` | `string` | no | empty | External hostname without scheme or trailing slash; empty uses `.S.AppDomain` |
| `verification_code` | `secret` | no | `123456` | Private/default login code used only when no mail provider is configured |
| `allow_signup` | `bool` | no | `true` | Multica signup switch |
| `allowed_email_domains` | `string` | no | empty | Optional comma-separated additional signup domains |
| `disable_workspace_creation` | `bool` | no | `false` | Locks creation of new workspaces after bootstrap when enabled |
| `resend_api_key` | `secret` | no | empty | Enables Resend delivery when SMTP is not configured |
| `resend_from_email` | `string` | no | `noreply@multica.ai` | Sender address; operators using Resend must replace it with an address on a verified domain |
| `smtp_host` | `string` | no | empty | Enables SMTP and overrides Resend |
| `smtp_port` | `string` | no | `25` | SMTP relay port |
| `smtp_username` | `string` | no | empty | Optional SMTP authentication username |
| `smtp_password` | `secret` | no | empty | Optional SMTP authentication password |
| `smtp_tls` | `string` | no | empty | SMTP TLS mode, such as `implicit` for non-standard SMTPS ports |
| `smtp_tls_insecure` | `bool` | no | `false` | Allows private or self-signed SMTP certificates |
| `smtp_ehlo_name` | `string` | no | empty | Optional FQDN used for SMTP EHLO/HELO |
| `google_client_id` | `string` | no | empty | Optional Google OAuth client ID |
| `google_client_secret` | `secret` | no | empty | Optional Google OAuth client secret |

The wizard cannot express conditional required fields. Therefore Resend/SMTP parameters remain optional, and their descriptions must explain the valid combinations. No unsupported validation fields such as `type: email`, `regex`, or `placeholder` will be added.

## Domain behavior

LazyCat does not provide a manifest field for binding an arbitrary external FQDN. `custom_domain` configures Multica's understanding of its external origin; DNS, TLS, and reverse-proxy setup remain operator responsibilities.

When `custom_domain` is empty, the selected origin is `https://{{ .S.AppDomain }}`. When it is set, the selected origin is `https://{{ .U.custom_domain }}`.

The selected origin is used for:

- `FRONTEND_ORIGIN`
- `GOOGLE_REDIRECT_URI` with `/auth/callback`
- `MULTICA_APP_URL`

`CORS_ALLOWED_ORIGINS` includes the LazyCat instance origin and, when configured, the custom origin. `COOKIE_DOMAIN` remains empty so cookies are host-only and work on either hostname. `MULTICA_PUBLIC_URL` remains empty for the same-origin proxy layout, following upstream guidance.

## Manifest alignment

Update `lzc-manifest.yml` to match the current upstream Docker Compose environment surface:

- Add SMTP relay variables.
- Add S3-compatible endpoint, addressing, credential, and attachment-download variables.
- Add workspace creation, GitHub App, webhook public URL, trusted proxy, Lark/Feishu, and Slack variables.
- Preserve upstream defaults: SMTP port `25`, `SMTP_TLS_INSECURE=false`, S3 region `us-west-2`, attachment mode `auto`, and download URL TTL `30m`; opt-in integrations remain empty.
- Derive PostgreSQL and JWT credentials with `stable_secret` so users do not configure them and installations do not share public hard-coded secrets.
- Keep PostgreSQL and upload persistence under `/lzcapp/var`.
- Add `/` to `application.public_path` alongside `/api` because Multica provides its own authenticated login UI.
- Declare `net.internet` in `package.yml` for external integrations.

## File handling

Multica supports attachment upload and download. To satisfy LazyCat App Store requirements, package the official `lzc-file-chooser-inject.js`, add the browser inject for `/*`, and add the content directory to `lzc-build.yml`. The injector must be sourced from the official LazyCat developer URL and kept as an unmodified packaged asset.

## Source Compose and documentation

- Replace the repository's stale `docker-compose.selfhost.yml` with the current official upstream Compose supplied in the request, including loopback-only port bindings and all current environment variables.
- Update `README.md` so login instructions reference the required installation email, explain the two mail modes, document the fixed-code security warning, and describe custom-domain CLI setup.
- Preserve the existing package ID `community.lazycat.app.multica`; changing it would create a separate application.

## Verification

Completion requires:

1. YAML parsing of package, build, deploy-params, manifest, and Compose files.
2. `docker compose -f docker-compose.selfhost.yml config` when Docker Compose is available.
3. `lzc-cli project lint` for release and development configurations.
4. `lzc-cli project release` followed by `lzc-cli lpk info` on the produced package.
5. Inspection that the packaged deployment parameters and file-picker asset are present.
6. Confirmation that only the required login email lacks a default and all other setup fields are optional or defaulted.

## Known constraints

- LazyCat setup parameters cannot enforce “Resend or SMTP must be complete” as a cross-field rule.
- The fixed-code fallback is intended for private/default installs. Operators exposing Multica publicly must configure Resend or SMTP.
- Custom-domain support configures application URLs only; it cannot provision DNS, TLS certificates, or an external reverse proxy.
- Existing backend and frontend images are not hosted on `registry.lazycat.cloud`, so App Store lint may continue to warn until the images are copied to the LazyCat registry.
