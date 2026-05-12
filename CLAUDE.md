# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup       # First-time setup: install deps + prisma generate + DB migrations
npm run dev         # Start dev server with Turbopack
npm run build       # Production build
npm run lint        # ESLint
npm run test        # Run Vitest
npm run db:reset    # Reset database (destructive)
```

To run a single test file: `npx vitest run <path/to/test>`

**Important**: Do not run `npm audit fix` — package versions are pinned for compatibility.

## Environment

Requires `ANTHROPIC_API_KEY` in `.env`. If missing or placeholder, the app falls back to `MockLanguageModel` (canned responses, for demo only).

## Architecture

UIGen is an AI-powered React component generator. Users describe components in chat; Claude calls file-manipulation tools to build them; a live preview renders the result.

### Request flow

```
User chat input
  → POST /api/chat/route.ts
  → Claude (claude-haiku-4-5) with tool definitions
  → Tool calls: str_replace_editor + file_manager
  → VirtualFileSystem (in-memory, never disk)
  → FileSystemContext (React state)
  → PreviewFrame (Babel JSX transform + iframe)
```

### Virtual file system

`src/lib/file-system.ts` — All generated files live in memory only. No files are ever written to disk. The serialized file system is persisted to the Prisma `Project.data` column as a JSON string on each chat turn completion (`onFinish` in the API route).

### AI tools (`src/lib/tools/`)

- `str-replace.ts` — `str_replace_editor` tool: create/view/replace/insert file contents
- `file-manager.ts` — `file_manager` tool: rename/delete files

Both tools operate on a `VirtualFileSystem` instance passed at request time.

### Contexts

- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — owns the `VirtualFileSystem` instance, selected file, and refresh trigger
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — wraps `@ai-sdk/react`'s `useChat` hook; manages messages and input state

### Live preview

`src/lib/transform/jsx-transformer.ts` uses Babel standalone (in the browser) to transform JSX → JavaScript. It generates an HTML document with an import map so React, Tailwind, and other modules resolve without a bundler. The result is injected into an iframe in `PreviewFrame`.

### Persistence

Authenticated users get full persistence: messages + serialized file system are saved to the SQLite `Project` model via Prisma. Anonymous users lose state on refresh. Auth is JWT-based, stored in HTTP-only cookies (`src/lib/auth.ts`).

### System prompt & caching

The Claude system prompt lives in `src/lib/prompts/generation.tsx`. It instructs Claude to use Tailwind-only styling, create `/App.jsx` as the component entrypoint, and never create HTML files. The prompt uses Anthropic's ephemeral `cacheControl` to reduce cost/latency on repeated requests.

### Key paths

| Path | Purpose |
|------|---------|
| `src/app/api/chat/route.ts` | AI endpoint — tool setup, streaming response, persistence |
| `src/lib/file-system.ts` | `VirtualFileSystem` class |
| `src/lib/provider.ts` | Claude / MockLanguageModel selection |
| `src/lib/transform/jsx-transformer.ts` | Browser-side Babel transform |
| `src/lib/prompts/generation.tsx` | Claude system prompt |
| `src/actions/` | Next.js Server Actions (auth, project CRUD) |
| `prisma/schema.prisma` | `User` and `Project` models (SQLite) |
