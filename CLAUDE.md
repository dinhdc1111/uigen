# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # First-time setup: install deps + prisma generate + migrate
npm run dev            # Start dev server with Turbopack
npm run build          # Production build
npm run lint           # Run ESLint
npm run test           # Run Vitest unit tests
npm run db:reset       # Reset SQLite database
```

Single test: `npx vitest run <test-file-pattern>`

Environment: copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`. Without the key, the app falls back to `MockLanguageModel` for demo purposes.

## Architecture

**UIGen** is an AI-powered React component generator. Users describe components in a chat interface; Claude generates code stored in an in-memory virtual file system and rendered live in an iframe preview.

### Request Flow

1. Chat input → `/api/chat` (streaming route using Vercel AI SDK)
2. Claude receives the system prompt (`src/lib/prompts/`) and calls tools
3. Tool calls mutate the virtual file system via `ChatContext`
4. File changes trigger Babel JSX transformation (`src/lib/transform/jsx-transformer.ts`)
5. Transformed code renders in the preview iframe via dynamic import maps (React from esm.sh CDN)

### Virtual File System

`src/lib/file-system.ts` is the core data structure — an in-memory tree with no disk I/O. It is serialized to JSON and persisted in SQLite (via Prisma) for authenticated users. `FileSystemContext` (`src/lib/contexts/`) wraps it as React state.

### AI Tool Definitions

Two tools are exposed to Claude (`src/lib/tools/`):
- `str_replace_editor` — view, create, and edit files via exact-string replacement
- `file_manager` — rename and delete files

The provider is selected in `src/lib/provider.ts`: real Claude (`claude-haiku-4-5`) when `ANTHROPIC_API_KEY` is set, otherwise `MockLanguageModel`.

### Key Contexts

- `FileSystemContext` — owns virtual file system state; consumed by editor, preview, and chat
- `ChatContext` — owns messages and orchestrates tool call results back into the file system

### Auth

JWT sessions (`src/lib/auth.ts`) stored in HTTP-only cookies. Passwords hashed with bcrypt. Server Actions in `src/actions/` handle sign-up, sign-in, and project CRUD.

### Database

Prisma with SQLite. Schema has two models: `User` and `Project` (stores `messages` and file system `data` as JSON). Generated client lives in `src/generated/prisma/`.

### Node Compatibility Shim

`node-compat.cjs` patches `localStorage`/`sessionStorage` for Node 25+ SSR compatibility; required by the dev server startup script.

## Path Alias

`@/*` maps to `src/*` (configured in `tsconfig.json` and shadcn `components.json`).
