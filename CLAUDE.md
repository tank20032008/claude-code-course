# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Development server with Turbopack
npm run build        # Production build
npm run lint         # Run ESLint
npm run test         # Run tests with Vitest
npm run setup        # Install deps + Prisma setup + DB migrations
npm run db:reset     # Reset SQLite database
```

Run a single test file:
```bash
npx vitest run src/components/chat/__tests__/MessageList.test.tsx
```

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates them using tool calls that operate on an in-memory virtual file system.

### Request Flow

1. User sends a chat message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. Server streams Claude's response via Vercel AI SDK `streamText()`
3. Claude uses tools (`str_replace_editor`, `file_manager`) to create/edit files
4. Tool results update **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`)
5. **PreviewFrame** (`src/components/preview/PreviewFrame.tsx`) re-renders the component in an iframe

### Key Abstractions

**VirtualFileSystem** (`src/lib/file-system.ts`) — In-memory file system (no disk writes). Backed by `Map<string, FileNode>`. Supports create/update/delete/rename with `serialize()`/`deserializeFromNodes()` for DB persistence. The server and client each hold their own independent copy; they sync only at request/response boundaries (client serializes FS into the POST body; server saves back on `onFinish`).

**ChatContext** (`src/lib/contexts/chat-context.tsx`) — Wraps Vercel's `useChat` hook, connects AI responses to file system mutations. Uses `onToolCall` to replay each tool invocation against the client-side VirtualFileSystem, then increments `refreshTrigger` (an integer counter in FileSystemContext) so preview/editor re-render without deep object diffing.

**Tools** (`src/lib/tools/`) — Claude tool definitions:
- `str-replace.ts` — String replacement for editing existing file contents
- `file-manager.ts` — File creation and deletion

**Provider** (`src/lib/provider.ts`) — Configures Claude Haiku 4.5 with prompt caching (ephemeral, 24h window). Falls back to `MockLanguageModel` when `ANTHROPIC_API_KEY` is absent; the mock simulates a realistic 4-step generation sequence with streaming delays.

**System Prompt** (`src/lib/prompts/generation.tsx`) — Instructs Claude to generate React + TailwindCSS components with `App.jsx` as the root entry point, using `@/` import aliases.

### JSX Preview Pipeline

Generated files are never served from disk. `PreviewFrame` uses `src/lib/transform/jsx-transformer.ts` to:
1. Resolve `@/` import aliases to blob URLs
2. Build a browser import map pointing React/etc. at `esm.sh` CDN
3. Transpile JSX with `@babel/standalone` (browser-side)
4. Inject Tailwind CSS via `<style>`
5. Render everything in a sandboxed `<iframe>`

Entry point auto-detection order: `/App.jsx` → `/App.tsx` → `/index.jsx` → `/index.tsx` → `/src/App.jsx` → `/src/App.tsx` → first `.jsx`/`.tsx` file found.

### Data Persistence

- **Authenticated users:** Projects (chat messages + file system state serialized as JSON) stored in SQLite via Prisma (`prisma/schema.prisma`)
- **Anonymous users:** Tracked in-browser via `src/lib/anon-work-tracker.ts`

### UI Layout

Split-pane (`src/app/main-content.tsx`):
- Left 35%: Chat interface (`src/components/chat/`)
- Right 65%: Preview/Code tabs
  - Preview tab: iframe rendering generated components
  - Code tab: Monaco editor + file tree (`src/components/editor/`)

### Auth

JWT-based with bcrypt. Server actions in `src/actions/`. Middleware at `src/middleware.ts` protects authenticated routes. Anonymous users are supported without login.

## Environment

`ANTHROPIC_API_KEY` in `.env` is optional — the app uses a mock model if absent.
