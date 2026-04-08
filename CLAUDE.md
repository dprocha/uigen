# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe React components in natural language; Claude generates the code using a tool-based approach, and the result renders instantly in a sandboxed iframe via in-browser Babel JSX transformation.

## Commands

```bash
npm run setup       # First-time setup: install, prisma generate + migrate
npm run dev         # Dev server (Next.js + Turbopack)
npm run build       # Production build
npm run lint        # ESLint
npm run test        # Vitest (all tests)
npm run db:reset    # Reset SQLite database
```

Run a single test file:
```bash
npx vitest run src/lib/transform/__tests__/jsx-transformer.test.ts
```

## Environment Variables

```bash
ANTHROPIC_API_KEY=   # Optional — uses mock provider if absent
JWT_SECRET=          # Defaults to "development-secret-key"
```

## Architecture

### Request Flow

1. User submits chat message → `ChatInterface` → `POST /api/chat`
2. `/api/chat/route.ts` calls `getLanguageModel()` (real Claude or mock)
3. Claude uses two AI tools to manipulate files:
   - `str_replace_editor`: create / view / edit / insert in files
   - `file_manager`: rename / delete files
4. Tool calls update the `VirtualFileSystem` (in-memory, no disk I/O)
5. `FileSystemContext` propagates changes to all components
6. `PreviewFrame` receives updated files, transforms JSX via `@babel/standalone`, builds an import map pointing to `esm.sh` CDN, and renders in a sandboxed `<iframe>`
7. On completion, project state is serialized and saved to SQLite via Prisma

### Key Modules

| Path | Role |
|---|---|
| `src/app/api/chat/route.ts` | Streaming AI endpoint; orchestrates tool calls |
| `src/lib/file-system.ts` | `VirtualFileSystem` class — Map-based in-memory FS |
| `src/lib/contexts/file-system-context.tsx` | React context wrapping VirtualFileSystem |
| `src/lib/contexts/chat-context.tsx` | Chat state + `useChat` (Vercel AI SDK) |
| `src/lib/transform/jsx-transformer.ts` | Babel JSX → ESM + import map generation |
| `src/lib/tools/str-replace.ts` | AI tool definition for file editing |
| `src/lib/tools/file-manager.ts` | AI tool definition for rename/delete |
| `src/lib/provider.ts` | Selects real or mock language model |
| `src/lib/prompts/generation.tsx` | System prompt for component generation |
| `src/components/preview/PreviewFrame.tsx` | Sandboxed iframe renderer |
| `src/lib/auth.ts` | JWT session management (httpOnly cookies, 7-day expiry) |
| `src/middleware.ts` | Auth middleware for protected routes |
| `src/actions/` | Next.js Server Actions for auth and project CRUD |

### Layout

`MainContent` (`src/app/main-content.tsx`) wraps everything in `FileSystemProvider` + `ChatProvider` and renders a resizable three-pane layout:
- **Left panel**: `ChatInterface` (messages + input)
- **Right panel**: tabs for `PreviewFrame` (live preview) and `FileTree` + `CodeEditor` (Monaco)

### Data Persistence

- Database: SQLite via Prisma (`prisma/dev.db`)
- Schema: `User` (email/password) + `Project` (messages JSON, VirtualFileSystem JSON)
- Anonymous users can generate; only authenticated users persist projects
- Project data is stored as stringified JSON in the `data` column

### Mock Provider

When `ANTHROPIC_API_KEY` is not set, `src/lib/provider.ts` returns a `MockLanguageModel` that generates static demo components (Counter, Form, Card) in 4 simulated steps. Max steps for real API: 40; max tokens: 10,000.

## Tech Stack

- **Next.js 15** (App Router) + **React 19** + **TypeScript**
- **Tailwind CSS v4** with shadcn/ui components (Radix UI primitives)
- **Vercel AI SDK** (`ai` package) for streaming + tool calls
- **@babel/standalone** for in-browser JSX transformation
- **Monaco Editor** for code editing
- **Prisma 6** with SQLite
- **Vitest** + **@testing-library/react** for tests
