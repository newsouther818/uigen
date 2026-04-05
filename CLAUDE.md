# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface, an LLM generates code via tool calls that manipulate a virtual file system, and the result renders in a live preview iframe. Works with or without an Anthropic API key (falls back to a mock provider with canned responses).

## Commands

```bash
npm run setup          # Install deps + generate Prisma client + run migrations
npm run dev            # Dev server with Turbopack (localhost:3000)
npm run build          # Production build
npm run lint           # ESLint
npm test               # Vitest (all tests)
npx vitest run src/lib/__tests__/file-system.test.ts  # Single test file
npx prisma migrate dev # Run pending migrations
npx prisma generate    # Regenerate Prisma client after schema changes
npm run db:reset       # Reset database (destructive)
```

All `npm run dev/build/start` commands use `NODE_OPTIONS='--require ./node-compat.cjs'` to patch Node 25+ Web Storage SSR issues.

## Architecture

### Request Flow

1. **Chat UI** (`src/components/chat/`) sends messages to `/api/chat` route
2. **API route** (`src/app/api/chat/route.ts`) uses Vercel AI SDK `streamText` with Claude (or mock provider) and two custom tools: `str_replace_editor` and `file_manager`
3. **Tools** modify a `VirtualFileSystem` instance reconstructed from the client's serialized file state
4. **Client** receives streamed tool call results, updates its own VFS via `FileSystemContext`, triggering preview re-render

### Key Abstractions

- **VirtualFileSystem** (`src/lib/file-system.ts`): In-memory tree of `FileNode`s. All generated code lives here — nothing written to disk. Supports serialize/deserialize for client-server round-trips.
- **JSX Transformer** (`src/lib/transform/jsx-transformer.ts`): Uses `@babel/standalone` to transform JSX/TSX in the browser for live preview. Resolves imports between virtual files, stubs missing modules.
- **Provider** (`src/lib/provider.ts`): Returns real Anthropic model (`claude-haiku-4-5`) if `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that returns canned component code. Model constant is `MODEL` in this file.
- **LLM Tools** (`src/lib/tools/`): `str-replace.ts` (view/create/replace/insert in VFS files) and `file-manager.ts` (rename/delete). These are the tools the AI calls during generation.

### State Management

- **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`): React context holding the client-side VFS, syncs with API responses
- **ChatContext** (`src/lib/contexts/chat-context.tsx`): Manages chat messages, handles `useChat` from Vercel AI SDK

### Data Model

SQLite via Prisma. Two models: `User` (email/password auth with bcrypt + JWT via `jose`) and `Project` (stores messages and VFS data as JSON strings). Prisma client output goes to `src/generated/prisma/`.

### Routing

- `/` — Anonymous users see main content; authenticated users redirect to most recent project
- `/[projectId]` — Project page (auth required)
- `/api/chat` — Streaming chat endpoint

### Auth

JWT-based. `src/lib/auth.ts` handles session creation/verification. `src/middleware.ts` protects `/api/projects` and `/api/filesystem`. Components in `src/components/auth/`.

### UI

shadcn/ui (new-york style) with Radix primitives. Components in `src/components/ui/`. Tailwind CSS v4. Monaco editor for code view (`@monaco-editor/react`). Resizable panels via `react-resizable-panels`.

## Path Alias

`@/*` maps to `./src/*` (configured in tsconfig.json).

## Testing

Vitest with jsdom environment and React Testing Library. Tests live in `__tests__/` directories next to their source files.
