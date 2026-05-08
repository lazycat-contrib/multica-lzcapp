# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

LazyCat app packaging/deployment config for **multica** (`cloud.lazycat.app.multica`). No application source code lives here — backend and frontend are in separate repos, referenced as Docker images.

## Architecture

- **postgres**: pgvector/pgvector:pg17 (vector search)
- **backend**: ghcr.io/multica-ai/multica-backend (port 8080)
- **frontend**: ghcr.io/multica-ai/multica-web (port 3000)

## Developer Docs

When you need LazyCat platform documentation, start from https://developer.lazycat.cloud/llms.txt to discover available doc pages.

## Build & Deploy

```bash
lzc-cli project deploy    # Build and install to target box
lzc-cli project release   # Build release package
lzc-cli project lint      # Lint manifest compatibility
```

## Dev vs Release

- `lzc-build.yml` — release config (uses `contentdir: selfhost`)
- `lzc-build.dev.yml` — dev overrides: package name `multica.dev`, `DEV_MODE=1`, no contentdir
