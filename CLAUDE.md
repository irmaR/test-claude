# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (Turbopack)
npm run dev

# Run all tests
npm test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Lint
npm run lint

# Production build
npm run build

# Reset database
npm run db:reset
```

The app requires `ANTHROPIC_API_KEY` in `.env`. Without it, a MockLanguageModel returns static dummy components — the app still runs.

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat; Claude generates them into a virtual file system; the result renders live in an iframe.

### Core Data Flow

```
User message → ChatContext (useChat) → POST /api/chat → streamText (Claude)
  → AI tools (str_replace_editor, file_manager)
  → FileSystemContext (in-memory state)
  → JSXTransformer (Babel standalone → blob URLs + import map)
  → PreviewFrame (iframe with esm.sh CDN imports)
```

### Key Layers

**Virtual File System (`src/lib/file-system.ts`)**
All generated files live in memory — nothing is written to disk. The `VirtualFileSystem` class manages a tree structure with `createFile`, `updateFile`, `deleteFile`, `renameFile`, `viewFile`. Serialized to JSON for project persistence.

**AI Integration (`src/app/api/chat/route.ts`)**
Uses Vercel AI SDK's `streamText` with two tools:
- `str_replace_editor` (`src/lib/tools/str-replace.ts`) — view/create/edit file contents
- `file_manager` (`src/lib/tools/file-manager.ts`) — rename/delete files

Supports up to 40 tool-use steps per request. Uses Anthropic prompt caching (`cacheControl: { type: "ephemeral" }`). On stream completion, serializes the project to Prisma/SQLite for logged-in users.

**Provider (`src/lib/provider.ts`)**
Exports `getModel()` which returns Claude Haiku 4.5 when `ANTHROPIC_API_KEY` is set, or `MockLanguageModel` otherwise. The mock returns hardcoded `str_replace_editor` tool calls generating a Counter/Form/Card component.

**JSX Transformer (`src/lib/transform/jsx-transformer.ts`)**
Client-side Babel compilation: JSX+TypeScript → JS → blob URL. Builds an import map pointing React 19 and common libraries to `esm.sh`. Used by `PreviewFrame` to run generated code directly in the browser without a build step.

**Contexts**
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — global VFS state; handles `onToolCall` callbacks from the AI stream, applies tool results to the VFS
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — wraps `useChat`, manages messages and submission, tracks anonymous session work

**Authentication (`src/lib/auth.ts`)**
JWT (jose, 7-day expiry) stored in httpOnly cookie. Bcrypt (10 rounds) for passwords. Server actions in `src/actions/index.ts`. Middleware at `src/middleware.ts` protects routes. Anonymous users can use the app without auth; projects are not persisted for them.

**Database**
Prisma + SQLite. See `prisma/schema.prisma` for the full data model. `Project.messages` and `Project.data` are JSON strings (serialized chat history and VFS state respectively).

### UI Structure

Split-pane layout (`react-resizable-panels`): chat on the left, editor/preview on the right. The right panel has tabs for Preview (iframe) and Code (Monaco editor + FileTree). shadcn/ui components with Tailwind CSS v4.

### Path Alias

`@/*` maps to `./src/*` throughout the codebase.

### Testing

Vitest with jsdom. Tests live alongside source in `__tests__/` subdirectories. Key test areas: VFS operations, JSX transformer, chat/file-system contexts, editor components.
