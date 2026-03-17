# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps + generate Prisma client + migrate DB
npm run dev          # Start dev server with Turbopack
npm run build        # Build for production
npm run lint         # Run ESLint
npm run test         # Run all tests with Vitest
npm run db:reset     # Force reset the SQLite database (destructive)
```

To run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

## Architecture

UIGen is an AI-powered React component generator. Users chat with Claude to generate and iterate on React component code, which is previewed live in the browser.

### Core Concepts

**Virtual File System (`src/lib/file-system.ts`):** All generated code lives in an in-memory file system — no disk writes. It supports full CRUD operations and serializes to/from JSON for persistence in the database. `/App.jsx` is always the component entrypoint.

**AI Chat Flow (`src/app/api/chat/route.ts`):**
1. POST receives message history + serialized file system state
2. VirtualFileSystem is reconstructed from the serialized state
3. Claude receives the system prompt, message history, file system contents, and two tools: `str_replace_editor` (edit files) and `file_manager` (create/delete/rename files)
4. Streamed tool calls mutate the in-memory file system
5. Final file system state is serialized and saved to the Prisma database (if authenticated)

**Provider (`src/lib/provider.ts`):** When `ANTHROPIC_API_KEY` is set, uses Claude Haiku 4.5. When absent, falls back to `MockLanguageModel` which generates pre-built demo components — useful for offline development.

**Auth:** JWT tokens stored in httpOnly cookies (7-day expiry). Middleware at `src/middleware.ts` protects `/api/projects` and `/api/filesystem`. Passwords are bcrypt-hashed.

### Data Model (Prisma + SQLite)

- `User` — id, email, password
- `Project` — id, name, userId, messages (JSON string), data (serialized file system JSON)

### State Management

Two React context providers wrap the app:
- `FileSystemProvider` (`src/lib/contexts/`) — virtual file system state, current open file
- `ChatProvider` — chat messages, project metadata

### UI Layout

Three-panel layout (via `react-resizable-panels`):
- Left: `ChatInterface` — message input and history
- Right top/bottom tabs: live preview iframe ↔ Monaco code editor with file tree

The system prompt (`src/lib/prompts/generation.tsx`) instructs the model to use Tailwind CSS and the `@` import alias.
