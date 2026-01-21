# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

NextChat is a cross-platform AI chat application supporting multiple LLM providers (OpenAI, Anthropic, Google Gemini, DeepSeek, Azure, and many more). It runs as a web app (Next.js) and desktop app (Tauri).

## Common Commands

```bash
# Install dependencies
yarn install

# Development (web)
yarn dev

# Build for production
yarn build

# Lint
yarn lint

# Run tests (watch mode)
yarn test

# Run tests in CI
yarn test:ci

# Desktop app development (requires Tauri)
yarn app:dev

# Desktop app build
yarn app:build

# Build masks (pre-defined chat personas)
yarn mask
```

## Architecture

### Directory Structure

- `app/` - Main Next.js application (App Router)
  - `api/` - Server-side API routes for all AI providers
  - `client/` - Client-side API implementations
  - `components/` - React UI components
  - `store/` - Zustand state management
  - `locales/` - i18n translations
  - `masks/` - Pre-defined chat personas
  - `mcp/` - Model Context Protocol integration
- `src-tauri/` - Tauri desktop app configuration
- `test/` - Jest tests

### State Management

Uses Zustand with IndexedDB persistence via `createPersistStore()` in `app/utils/store.ts`. Key stores:
- `app/store/chat.ts` - Chat sessions and messages
- `app/store/access.ts` - API keys and endpoints
- `app/store/config.ts` - App configuration
- `app/store/mask.ts` - Chat personas

### AI Provider System

**Two-layer architecture:**

1. **Server API Routes** (`app/api/[provider]/[...path]/route.ts`) - Dynamic routing handles all providers via Edge Runtime
2. **Client API Clients** (`app/client/platforms/`) - Each provider implements `LLMApi` interface with `chat()`, `speech()`, `usage()`, `models()` methods

**Adding a new provider requires:**
- Client implementation in `app/client/platforms/`
- Server handler in `app/api/common.ts` or dedicated route
- Constants in `app/constant.ts` (ServiceProvider enum, API paths)
- Access store updates in `app/store/access.ts`

### Key Files

- `app/constant.ts` - All constants, model lists, API paths, enums
- `app/config/server.ts` - Server-side env var configuration
- `app/client/api.ts` - Client API factory and `LLMApi` interface
- `app/components/chat.tsx` - Main chat component
- `app/components/home.tsx` - Main layout with React Router routes

### Build Modes

- `standalone` (default) - Standard Next.js server build
- `export` - Static export for Tauri desktop app (`BUILD_MODE=export`)

### MCP (Model Context Protocol)

Enable with `ENABLE_MCP=true` environment variable. Implementation in `app/mcp/`.

## Environment Variables

Key variables (see README for full list):
- `OPENAI_API_KEY` - OpenAI API key
- `CODE` - Access password(s), comma-separated
- `BASE_URL` - Override OpenAI API base URL
- `CUSTOM_MODELS` - Add/remove models (e.g., `+llama,-gpt-3.5-turbo`)
- Provider-specific: `ANTHROPIC_API_KEY`, `GOOGLE_API_KEY`, `DEEPSEEK_API_KEY`, etc.

## Testing

Tests use Jest with jsdom environment. Test files match `**/*.test.{js,ts,jsx,tsx}`.

```bash
# Run specific test file
yarn test -- test/model-provider.test.ts
```

## Deployment

This fork (`Seandor/Next-Chat`) deploys to Azure Web App using GitHub Actions.

- **Branch**: `siyuanling/next-chat` triggers deployment to Azure
- **Workflow**: `.github/workflows/main_next-chat.yml.yml`
- **Target**: Azure Web App named `next-chat`
- **URL**: https://nextchat.siyuan0.tech/

**Important Notes:**
- The app runs on Azure with Node.js standalone mode (not Edge Runtime locally)
- However, API routes use `runtime = "edge"` which behaves differently in production
- Local development (`yarn dev`) uses Node.js runtime, which is more lenient than Edge Runtime
- Always test Edge Runtime-specific behavior (like GET request body handling) in the deployed environment
