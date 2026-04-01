# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup       # First-time setup: install deps, generate Prisma client, run migrations
npm run dev         # Start dev server with Turbopack
npm run build       # Production build
npm run lint        # ESLint
npm run test        # Vitest test suite
npm run db:reset    # Reset SQLite database
```

Tests use Vitest with JSDOM. There is no command to run a single test file via a package.json script — use `npx vitest run src/path/to/file` directly.

## Environment

- `ANTHROPIC_API_KEY` — required for real AI generation; if absent or empty, the app uses a mock provider that generates static demo components.
- `JWT_SECRET` — used for session signing; falls back to a dev default if unset.

## Architecture

UIGen is a Next.js 15 (App Router) application that lets users describe React components in chat, and Claude AI generates them with a live preview.

### Core Flow

```
User chat input
  → /api/chat (stream via Vercel AI SDK)
  → Claude uses tools (str_replace_editor, file_manager) to create/edit files
  → FileSystemContext updates virtual FS in memory
  → PreviewFrame reads FS state, transforms JSX via Babel standalone, renders in sandboxed iframe
```

### Virtual File System (`src/lib/file-system.ts`)

All files live in memory — nothing is written to disk. The `VirtualFileSystem` class is a tree of `FileNode` objects accessed by path. It supports CRUD, rename, directory creation, and serialization (for database persistence). The FS root is `/` and every project must have `/App.jsx` as the entry point.

### AI Tools (`src/lib/tools/`)

Claude is given two tools:
- **str_replace_editor** — view, create, str_replace, insert, undo_edit operations on the virtual FS
- **file_manager** — rename and delete operations

Tool schemas are validated with Zod. Tool calls are executed client-side inside `FileSystemContext` and `ChatContext`.

### Live Preview (`src/lib/transform/jsx-transformer.ts`)

Transforms the virtual FS into a runnable HTML page:
1. Detects the entry point (`/App.jsx`, `/App.tsx`, `src/App.jsx`, etc.)
2. Transpiles JSX/TSX with `@babel/standalone`
3. Builds an import map pointing to `esm.sh` CDN for React and other packages
4. Injects everything into a sandboxed iframe

CSS imports are stripped (not applied). Missing imported files generate placeholder components.

### AI Provider (`src/lib/provider.ts`)

- With API key: uses `anthropic()` with `claude-haiku-4-5` and prompt caching
- Without API key: `MockLanguageModel` simulates tool calls and returns canned Counter/ContactForm/Card components

### State Management

Two React contexts manage global state:
- **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`) — virtual FS state; also handles tool call execution for FS mutations
- **ChatContext** (`src/lib/contexts/chat-context.tsx`) — chat messages and AI streaming state; delegates tool calls to FileSystemContext

### Auth & Persistence

- JWT sessions via `jose` stored in httpOnly cookies (7-day expiry); logic in `src/lib/auth.ts`
- Server actions in `src/actions/` handle sign-up/sign-in and project CRUD
- Prisma + SQLite: `User` and `Project` models; `Project.messages` and `Project.data` store JSON-serialized chat history and FS state

### Generation Prompt (`src/lib/prompts/generation.tsx`)

System prompt instructs Claude to:
- Use Tailwind CSS for all styling
- Keep `/App.jsx` as the project entry point
- Use `@/` import aliases for custom modules
- Not create HTML files
