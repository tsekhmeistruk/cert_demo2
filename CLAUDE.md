# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface; Claude AI modifies a virtual file system (VFS) in real-time, with results rendered in a live preview iframe.

## Commands

```bash
npm run setup        # First-time setup: install deps + Prisma generate/migrate
npm run dev          # Development server with Turbopack (localhost:3000)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest tests
npm run db:reset     # Destructive: reset SQLite database
```

Run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

## Environment

Create `.env` with `ANTHROPIC_API_KEY=...` for real AI generation. Without it, the app falls back to mock mode with static component generation.

## Architecture

### Data Flow

User chat → `POST /api/chat` → Claude AI with tool context → `str_replace_editor`/`file_manager` tool calls → VirtualFileSystem mutations → `FileSystemContext` dispatches state updates → `PreviewFrame` re-renders + `CodeEditor` reflects changes.

### Key Concepts

**Virtual File System (VFS)** — In-memory file tree (`src/lib/file-system.ts`). No disk writes. Serialized to JSON for database storage in `Project.data`. The AI only mutates files through two tools:
- `str_replace_editor`: create/view/replace/insert file content (`src/lib/tools/str-replace.ts`)
- `file_manager`: rename/delete files (`src/lib/tools/file-manager.ts`)

**Tool Execution Bridge** — `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) receives tool calls streamed from the AI and executes them against the VFS state. The chat API route (`src/app/api/chat/route.ts`) streams responses and serializes the updated VFS back to the DB on each turn.

**Dual Mode** — `src/lib/provider.ts` returns a real Anthropic model when `ANTHROPIC_API_KEY` is set, otherwise a mock provider.

**Auth** — JWT sessions via `jose` (7-day expiry), stored in `auth-token` cookie. Server actions in `src/actions/index.ts` handle sign-in/sign-up/sign-out. `src/lib/auth.ts` manages session creation/verification.

**Preview Rendering** — `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) renders an iframe. The JSX transformer (`src/lib/transform/`) compiles generated component code client-side.

### State Management

Two React Contexts own all runtime state:
- `ChatContext` (`src/lib/contexts/chat-context.tsx`): wraps Vercel AI SDK `useChat`, manages messages and streaming
- `FileSystemContext`: owns the VFS instance and dispatches tool-call results back into the chat stream

### Database

Prisma + SQLite. Two models:
- `User`: email/password auth
- `Project`: stores `messages` (JSON chat history) and `data` (JSON-serialized VFS), optionally linked to a user

### UI Layout

Main layout (`src/app/main-content.tsx`) uses `react-resizable-panels` to split into three panes: chat, Monaco code editor, and live preview iframe.

## Testing

Tests live in `**/__tests__/*.test.{ts,tsx}` and use Vitest + jsdom + React Testing Library. Coverage includes file-system logic, contexts, JSX transformer, and chat/editor components.
