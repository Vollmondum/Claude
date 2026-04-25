# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run start        # Run production build
npm run lint         # ESLint (Next.js config)
npm run test         # Run Vitest tests

# Database
npm run setup        # Install deps + prisma generate + migrate dev
npm run db:reset     # Reset database (destructive)
npx prisma studio    # GUI for SQLite data
npx prisma migrate dev  # Apply pending migrations
```

All dev/build commands require the `node-compat.cjs` shim (handled by `cross-env NODE_OPTIONS`). Run tests with `npm run test -- --watch` for watch mode, or `npm run test -- path/to/file` for a single test file.

## Environment

Copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`. Without it, the app falls back to `MockLanguageModel` (static canned response) — useful for UI work without burning API credits.

## Architecture

**UIGen** is a Next.js 15 (App Router) app that lets users describe React components in a chat interface; Claude generates code files that render live in an iframe preview.

### Request Flow

1. User submits prompt in `ChatInterface`
2. `ChatContext` (Vercel AI SDK `useChat`) POSTs to `/api/chat/route.ts`
3. Route calls Claude with the current virtual FS serialized as context
4. Claude streams text + tool calls (`str_replace_editor`, `file_manager`)
5. Tool call results update `VirtualFileSystem` in `FileSystemContext`
6. `PreviewFrame` re-transforms JSX → executable code via Babel (`lib/transform/jsx-transformer.ts`) and hot-swaps the iframe
7. On stream finish, if user is authenticated, `onFinish` persists messages + FS to SQLite via Prisma

### Virtual File System

`lib/file-system.ts` — in-memory only, no disk I/O. Serialized to JSON for both Claude context and DB storage. AI tool calls are the only write path; the editor can also write directly. The FS state lives in `FileSystemContext` and is the source of truth for both the file tree and the preview.

### Live Preview

`components/preview/PreviewFrame` renders inside a sandboxed iframe. `jsx-transformer.ts` uses Babel to compile JSX and resolves `@/` imports within the iframe's import map. Entry point auto-detected (prefers `App.tsx`/`App.jsx`).

### Auth

JWT in httpOnly cookies (7-day TTL, via `jose`). Server actions in `actions/` handle sign-up/sign-in/sign-out. Middleware (`middleware.ts`) gates `/api/projects` and `/api/filesystem`. Anonymous users can generate components; projects are only persisted when authenticated.

### State Management

Two main React contexts:
- `ChatContext` — messages, streaming status, input handlers
- `FileSystemContext` — virtual FS state, tool execution callbacks

Both wrap the app in `app/main-content.tsx`.

### AI Tools

Defined in `lib/tools/`. Claude uses `str_replace_editor` for file edits and `file_manager` for create/delete/rename. System prompt lives in `lib/prompts/generation.tsx`.

### Database

SQLite via Prisma. Schema: `User` (id, email, hashed password) → `Project` (id, name, userId?, messages JSON, data JSON). Migrations in `prisma/migrations/`.

## Key Files

| File | Purpose |
|---|---|
| `src/app/api/chat/route.ts` | Streaming AI endpoint |
| `src/lib/file-system.ts` | VirtualFileSystem class |
| `src/lib/transform/jsx-transformer.ts` | JSX→runnable code (Babel) |
| `src/lib/provider.ts` | Claude / MockLanguageModel selection |
| `src/lib/prompts/generation.tsx` | System prompt for Claude |
| `src/lib/contexts/` | Chat and FileSystem React contexts |
| `prisma/schema.prisma` | DB schema |
| `node-compat.cjs` | Turbopack Node compatibility shim |
