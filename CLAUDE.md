# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies and initialize DB
npm run setup           # shorthand for: npm install && prisma generate && prisma migrate dev

# Development server (Turbopack)
npm run dev             # foreground
npm run dev:daemon      # background, logs to logs.txt

# Production build / start
npm run build
npm start

# Lint
npm run lint

# Tests
npm test                # run all tests
npx vitest run <file>   # run a single test file

# Database
npx prisma migrate dev  # apply migrations
npm run db:reset        # reset database (destructive)
npx prisma studio       # browse data
```

`NODE_OPTIONS='--require ./node-compat.cjs'` is automatically injected by the npm scripts — it is required for Next.js to run.

## Architecture

UIGen is a Next.js 15 (App Router) application that lets users describe React components in a chat interface, then streams AI-generated code into a virtual file system that renders in a live preview iframe.

### Core Data Flow

```
User prompt → /api/chat (streamText) → Claude tool calls
    → FileSystemContext (in-memory VirtualFileSystem)
    → PreviewFrame (Babel + iframe re-render)
    → Prisma (SQLite) for authenticated users
```

### Virtual File System

`src/lib/file-system.ts` — `VirtualFileSystem` class is the heart of the app. It stores all generated files in memory as a flat map. Files are serialized to JSON for database persistence. There is no disk I/O for user project files.

### AI Integration (`src/app/api/chat/`)

The streaming chat endpoint uses the Vercel AI SDK (`streamText`) with two tools:
- `str_replace_editor` — create and patch files via str_replace semantics
- `file_manager` — rename and delete files

Request limits: 120 s timeout, 10 000 max output tokens, 40 max steps.

The system prompt lives in `src/lib/prompts/generation.tsx` and instructs Claude to always maintain a root `App.jsx` file and use Tailwind for styling. Generated files must use `@` import aliases (e.g. `@/components/Foo`).

The live model is `claude-haiku-4-5`. When no API key is configured, a mock provider (`src/lib/provider.ts`) simulates multi-step tool calls and returns placeholder React components so the UI remains functional.

### Two-Context Architecture

`src/lib/contexts/file-system-context.tsx` — owns `VirtualFileSystem` state, exposes file CRUD, and receives tool-call results from the AI stream.

`src/lib/contexts/chat-context.tsx` — wraps Vercel AI SDK's `useChat`, wires tool invocations back into `FileSystemContext`, and coordinates the optimistic message list.

Both contexts are provided near the root in `src/app/[projectId]/page.tsx`.

### Preview Rendering

`src/components/preview/` — the preview tab renders the virtual file system in an `<iframe>`. Babel standalone transpiles JSX at runtime inside the iframe. File changes in `FileSystemContext` trigger a re-render automatically.

### Authentication & Persistence

Anonymous users work entirely in memory; `src/lib/anon-work-tracker.ts` tracks in-progress work. Authenticated users have projects saved to SQLite via Prisma (`src/lib/prisma.ts`). Session management uses JWTs in HTTP-only cookies (`src/lib/auth.ts`). Next.js middleware (`src/middleware.ts`) protects `/api/projects` and `/api/filesystem`.

### Key Paths

| Path | Purpose |
|---|---|
| `src/app/api/chat/route.ts` | Streaming AI endpoint |
| `src/lib/file-system.ts` | VirtualFileSystem class |
| `src/lib/prompts/generation.tsx` | Claude system prompt |
| `src/lib/provider.ts` | Claude / mock model provider |
| `src/lib/contexts/` | FileSystem and Chat React contexts |
| `src/components/preview/` | iframe live preview |
| `src/components/editor/` | Monaco editor + file tree |
| `src/components/chat/` | Chat UI |
| `prisma/schema.prisma` | DB schema (User, Project) |

### Path Alias

`@/*` resolves to `src/*` throughout the codebase.
