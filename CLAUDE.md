# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Database

The database schema is defined in `prisma/schema.prisma`. Reference it anytime you need to understand the structure of data stored in the database.

## Git

Commit message format: `[ANDPOS]: <message>`

## Code Style

Use comments sparingly. Only comment complex code.

## Commands

```bash
# First-time setup
npm run setup          # install deps + prisma generate + migrate

# Development
npm run dev            # Next.js dev server with Turbopack at http://localhost:3000
npm run dev:daemon     # same, but backgrounded, output to logs.txt

# Build / lint
npm run build
npm run lint

# Tests (Vitest + jsdom)
npm test               # run all tests
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx  # single file

# Database
npx prisma migrate dev   # apply schema changes
npm run db:reset         # wipe and re-migrate (destructive)
npx prisma generate      # regenerate client after schema edit
```

> **Do not run `npm audit fix`.** Dependencies are pinned to specific versions; `audit fix` can break compatibility.

## Architecture

### Overview

UIGen is an AI-powered React component generator. Users describe a component in a chat interface; the AI writes files into a **virtual file system** (in-memory, never on disk); the result is transpiled in-browser and rendered in a sandboxed `<iframe>` as a live preview.

### Data flow

```
Chat prompt
  → POST /api/chat/route.ts
    → Vercel AI SDK streamText (claude-haiku-4-5 or MockLanguageModel)
    → Tools: str_replace_editor / file_manager operate on VirtualFileSystem
    → Streaming response → ChatInterface
      → tool call events → FileSystemContext.handleToolCall (updates VFS)
      → refreshTrigger change → PreviewFrame re-renders
        → jsx-transformer (Babel standalone) transpiles JSX in-browser
        → sandboxed iframe srcdoc updated
```

### Key modules

| Path | Purpose |
|---|---|
| `src/lib/file-system.ts` | `VirtualFileSystem` class — in-memory tree of files/dirs; serialize/deserialize to plain objects for API transport and Prisma storage |
| `src/lib/contexts/file-system-context.tsx` | React context wrapping VFS; `handleToolCall` dispatches AI tool outputs (`str_replace_editor`, `file_manager`) into the VFS |
| `src/lib/transform/jsx-transformer.ts` | Client-side Babel transform (JSX→JS, TS support), import-map construction, CSS stripping — runs entirely in-browser |
| `src/components/preview/PreviewFrame.tsx` | Renders the live preview in a sandboxed `<iframe>` using `srcdoc`; entry point auto-detected (App.jsx → App.tsx → index.*) |
| `src/lib/provider.ts` | Returns `anthropic("claude-haiku-4-5")` when `ANTHROPIC_API_KEY` is set, otherwise `MockLanguageModel` (canned responses for dev without a key) |
| `src/app/api/chat/route.ts` | Streaming API route; reconstructs VFS from request, runs AI with tools, persists to Prisma on finish |
| `src/lib/auth.ts` | JWT-based sessions (jose), stored in an httpOnly cookie (`auth-token`); `getSession()` (server), `verifySession()` (middleware) |
| `src/lib/anon-work-tracker.ts` | Persists anonymous users' in-progress work to `sessionStorage`; offered to save on sign-up |
| `src/actions/` | Next.js server actions for project CRUD (create, get, list) |

### Database

SQLite via Prisma. Two models:
- `User` — email + bcrypt-hashed password
- `Project` — belongs to an optional `User`; `messages` (JSON string) and `data` (serialized VFS JSON string) stored as plain text columns

Prisma client is generated to `src/generated/prisma/` (non-standard output path — import from there, not `@prisma/client`).

### Auth

Custom JWT auth — no NextAuth. Sessions are 7-day JWTs signed with `JWT_SECRET` (falls back to a hard-coded development key). `/api/projects` and `/api/filesystem` are protected by `src/middleware.ts`; `/api/chat` is open (anonymous users can generate, but projects only save for authenticated sessions).

### Testing

Vitest with jsdom and React Testing Library. Tests live alongside source in `__tests__/` subdirectories. Run with `npm test` or target a single file with `npx vitest run <path>`.

## Claude PR test

Validates the Claude GitHub Actions integration for automated PR review and assistance.
