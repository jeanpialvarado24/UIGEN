# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# First-time setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (uses Turbopack + node-compat shim)
npm run dev

# Build for production
npm run build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/path/to/file.test.ts

# Reset database (destructive)
npm run db:reset

# After changing prisma/schema.prisma
npx prisma migrate dev
npx prisma generate
```

The `NODE_OPTIONS='--require ./node-compat.cjs'` prefix is required for all Next.js commands — it's already embedded in the npm scripts.

## Environment

Copy `.env` and set:
- `ANTHROPIC_API_KEY` — if omitted, a `MockLanguageModel` is used instead (see `src/lib/provider.ts`)
- `JWT_SECRET` — defaults to `"development-secret-key"` if omitted

## Architecture Overview

UIGen is a Next.js 15 (App Router) app that lets users describe React components in a chat, generates them via Claude, and renders a live preview — all without writing files to disk.

### Request / AI flow

1. User submits a message in `ChatInterface` → `ChatProvider` (`src/lib/contexts/chat-context.tsx`) calls `POST /api/chat` via Vercel AI SDK's `useChat`.
2. The route handler (`src/app/api/chat/route.ts`) reconstructs a `VirtualFileSystem` from the serialized file state sent by the client, calls `streamText` with two tools (`str_replace_editor`, `file_manager`), and streams the response back.
3. Tool calls are forwarded client-side via `onToolCall` → `handleToolCall` in `FileSystemContext`, which mutates the in-memory `VirtualFileSystem` and triggers a React re-render.
4. If the user is authenticated and a `projectId` exists, `onFinish` persists messages + VFS state to the Prisma `Project` record.

### Virtual File System

`VirtualFileSystem` (`src/lib/file-system.ts`) is an in-memory tree of `FileNode` objects. It never touches the disk. It serializes to/from a flat `Record<string, FileNode>` for transport (sent with every API request as `files` in the request body). The AI tools operate on the server-side copy; tool-call results are applied to the client-side copy via `FileSystemContext`.

### Live Preview

`PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) renders an `<iframe>`. When the VFS changes, `createImportMap` + `createPreviewHTML` (`src/lib/transform/jsx-transformer.ts`) are called:

- Each `.jsx/.tsx/.js/.ts` file is transpiled client-side via `@babel/standalone` into a Blob URL.
- An ES module import map is injected into the iframe HTML mapping file paths and `@/` aliases to those Blob URLs.
- Third-party packages (any non-relative import) are resolved via `https://esm.sh/`.
- The preview entry point is always `/App.jsx`.

### AI Provider

`getLanguageModel()` (`src/lib/provider.ts`) returns the real `claude-haiku-4-5` model when `ANTHROPIC_API_KEY` is set, or a `MockLanguageModel` (same file) when it isn't. The mock simulates multi-step tool use with static component templates.

### Auth

Custom JWT auth (`src/lib/auth.ts`) using `jose`. Sessions are stored in an `httpOnly` cookie (`auth-token`). Auth is enforced in middleware (`src/middleware.ts`). Passwords are hashed with `bcrypt`. Anonymous users can generate components; work is tracked in `src/lib/anon-work-tracker.ts` for prompting sign-up.

### Database

Prisma with SQLite (`prisma/dev.db`). Two models: `User` (email + hashed password) and `Project` (stores `messages` and `data` as JSON strings). The Prisma client is generated to `src/generated/prisma`.

### AI tool conventions

The generation system prompt (`src/lib/prompts/generation.tsx`) instructs the model to:
- Always create `/App.jsx` as the entry point
- Use `@/` path aliases for local imports (e.g., `@/components/Button`)
- Style exclusively with Tailwind CSS (no hardcoded styles)
- Never create HTML files

### Context providers

Two React contexts wrap the main UI:
- `FileSystemProvider` — owns the `VirtualFileSystem` instance, exposes CRUD operations, and processes AI tool calls.
- `ChatProvider` — wraps Vercel AI SDK's `useChat`, wires tool calls to `FileSystemProvider`, and tracks anonymous work.

Both are mounted in `MainContent` (`src/app/main-content.tsx`).
