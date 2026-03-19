# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (Next.js + Turbopack)
npm run dev

# Build for production
npm run build

# Run tests
npm run test

# Lint
npm run lint

# Database reset
npm run db:reset
```

Tests use **Vitest** with jsdom. Run a single test file:
```bash
npx vitest run src/components/chat/ChatInput.test.tsx
```

The `NODE_OPTIONS='--require ./node-compat.cjs'` flag is required for all Next.js commands — the npm scripts handle this automatically.

## Architecture

UIGen is an AI-powered React component generator. Users describe a component in chat, Claude generates it, and the result renders in a live preview — all without writing files to disk.

### Three-Panel Layout

```
Chat Panel | Preview / Code Editor
```

The top-level layout (`src/app/[projectId]/page.tsx`) wraps everything in two contexts that drive the entire app.

### Core Contexts (`src/lib/contexts/`)

**`ChatContext`** — manages chat messages and sends requests to `/api/chat`. Receives a streaming response that may include text and tool calls.

**`FileSystemContext`** — owns the virtual in-memory file system. Listens for tool call events from `ChatContext` and applies `str_replace_editor` and `file_manager` operations to update files. The active file's content is passed to the Preview and Editor.

These two contexts are decoupled: Chat streams tool calls → FileSystem processes them → UI re-renders.

### Virtual File System (`src/lib/file-system.ts`)

An in-memory tree structure representing files and directories. Supports create/read/update/delete and rename operations. Serializes to JSON for persistence in the database (`Project.data`).

### AI Integration (`src/app/api/chat/route.ts`)

- Uses `@ai-sdk/anthropic` with streaming via the `ai` package
- Two tools exposed to Claude:
  - `str_replace_editor` — replaces text ranges within a file (in `src/lib/tools/`)
  - `file_manager` — create, delete, rename files/directories
- System prompt is in `src/lib/prompts/generation.tsx`; it instructs Claude to generate self-contained React components using Tailwind CSS
- Supports prompt caching (`anthropic-beta: prompt-caching-2024-07-31`)

### Authentication (`src/lib/auth.ts`, `src/middleware.ts`)

JWT-based sessions stored in httpOnly cookies (7-day expiry). Anonymous usage is supported — `Project.userId` is nullable. Middleware protects routes; the `useAuth` hook (`src/hooks/useAuth.ts`) exposes session state client-side.

### Database

Prisma ORM with SQLite (`prisma/dev.db`). The canonical schema is in `prisma/schema.prisma` — refer to it for the authoritative model definitions. Two models:

- **`User`**: email + bcrypt password
- **`Project`**: name, optional userId, `messages` (JSON chat history), `data` (JSON serialized file system)

### Tech Stack

- **Next.js 15** (App Router, Turbopack)
- **React 19**
- **TypeScript** (strict mode, path alias `@/*` → `src/*`)
- **Tailwind CSS v4**
- **shadcn/ui** (style: new-york, Radix UI primitives, lucide icons)
- **Monaco Editor** — code editor panel
- **Vitest** + Testing Library — unit tests with jsdom
