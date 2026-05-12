# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (with Turbopack)
npm run dev

# Build for production
npm run build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx

# Reset database (drops and re-applies all migrations)
npm run db:reset

# Regenerate Prisma client after schema changes
npx prisma generate

# Create a new migration after schema changes
npx prisma migrate dev --name <migration-name>
```

## Environment

Copy `.env` and set `ANTHROPIC_API_KEY` to use Claude for generation. Without a key, the app uses `MockLanguageModel` in `src/lib/provider.ts`, which returns static component code.

The JWT secret defaults to `"development-secret-key"` when `JWT_SECRET` is not set.

## Architecture

### Overview
UIGen is a Next.js 15 App Router app where users describe React components in a chat interface and see them rendered live in a sandboxed iframe preview — no files are written to disk.

### Virtual File System
`src/lib/file-system.ts` — `VirtualFileSystem` is an in-memory tree of `FileNode` objects. All AI-generated code lives here. It serializes/deserializes to plain JSON for API transport and Prisma storage. The `@/` import alias used in generated components maps to the virtual root `/`.

### AI Tool Loop (`src/app/api/chat/route.ts`)
The chat API uses Vercel AI SDK's `streamText` with two tools the model calls to edit the virtual FS:
- `str_replace_editor` — creates files or performs string replacements (`buildStrReplaceTool` in `src/lib/tools/str-replace.ts`)
- `file_manager` — renames and deletes files (`buildFileManagerTool` in `src/lib/tools/file-manager.ts`)

Tool call results are intercepted on the client by `handleToolCall` in `FileSystemContext` and applied to the client-side `VirtualFileSystem` instance in real time.

### Live Preview Pipeline
`src/lib/transform/jsx-transformer.ts` drives the preview:
1. `createImportMap` — Babel-transforms every `.jsx/.tsx` file in the VFS to plain JS, wraps each in a blob URL, and builds an ES module import map. Third-party packages are resolved via `esm.sh`. Missing local imports get placeholder stub modules.
2. `createPreviewHTML` — generates a full HTML document with the import map and a `<script type="module">` that imports `/App.jsx` as the entry point and mounts it with `ReactDOM.createRoot`.
3. `PreviewFrame` sets the iframe's `srcdoc` to that HTML, re-running on every `refreshTrigger` from `FileSystemContext`.

### Auth
Custom JWT-based auth in `src/lib/auth.ts` using `jose`. Sessions are stored as httpOnly cookies (`auth-token`). `src/middleware.ts` can verify sessions on protected routes. Anonymous users can chat without signing in; project persistence requires authentication.

### Data Model (`prisma/schema.prisma`)
- `User` — email + bcrypt-hashed password
- `Project` — belongs to an optional `User`; stores `messages` (JSON array, Vercel AI SDK format) and `data` (serialized `VirtualFileSystem` nodes) as text columns

### Context Providers
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — owns the `VirtualFileSystem` instance and exposes `handleToolCall`, which bridges AI tool calls to FS mutations
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK's `useChat`, passes the serialized file system to the API on each message

### Key conventions
- Generated components must have `/App.jsx` as the root entry point with a default export
- All inter-file imports in generated code use the `@/` alias (e.g., `import Foo from '@/components/Foo'`)
- Style with Tailwind only — no inline styles; the preview iframe loads Tailwind via CDN
- Prisma client is generated to `src/generated/prisma` (non-standard output path set in `schema.prisma`)
