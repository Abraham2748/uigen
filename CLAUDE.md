# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface, and Claude generates React code that renders in real-time without writing files to disk.

## Commands

### Development
- `npm run dev` - Start Next.js dev server with Turbopack (http://localhost:3000)
- `npm run dev:daemon` - Start dev server in background, logs to logs.txt
- `npm run build` - Build for production
- `npm start` - Start production server

### Database
- `npm run setup` - Install dependencies, generate Prisma client, and run migrations
- `npm run db:reset` - Reset database (use with caution)
- `npx prisma generate` - Regenerate Prisma client after schema changes
- `npx prisma migrate dev` - Create and apply new migrations

### Testing
- `npm test` - Run Vitest tests
- `npm run lint` - Run ESLint

### Important
All npm scripts use `cross-env NODE_OPTIONS=--require=./node-compat.cjs` to fix Node.js 25+ Web Storage SSR compatibility issues.

## Architecture

### Virtual File System
The core of UIGen is a **VirtualFileSystem** (`src/lib/file-system.ts`) that stores all generated code in memory. No files are written to disk during code generation. The file system:
- Maintains a tree structure with `FileNode` objects (files and directories)
- Serializes to/from JSON for database persistence
- Supports standard file operations: create, read, update, delete, rename
- Provides specialized methods for AI tool calls: `viewFile`, `createFileWithParents`, `replaceInFile`, `insertInFile`

### AI Component Generation Flow

1. **Chat Interface** (`src/components/chat/ChatInterface.tsx`): User sends a prompt
2. **API Route** (`src/app/api/chat/route.ts`): Receives messages and current virtual file system state
3. **System Prompt** (`src/lib/prompts/generation.tsx`): Injected with cache control to guide Claude
4. **Tool Calling**: Claude uses two tools to modify the virtual file system:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`): Create/view/edit files with string replacement
   - `file_manager` (`src/lib/tools/file-manager.ts`): Rename and delete files/directories
5. **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`): React context manages VFS state and applies tool calls client-side
6. **Preview Rendering**: JSX transformer converts code to preview HTML

### Live Preview System

The preview system (`src/lib/transform/jsx-transformer.ts`) enables live React component rendering:

1. **Transformation**: Babel standalone transforms JSX/TSX to plain JavaScript
2. **Import Maps**: Creates browser-native import maps where:
   - React/React-DOM: Loaded from esm.sh CDN
   - Local files: Converted to blob URLs with all path variations (`/App.jsx`, `App.jsx`, `@/App.jsx`)
   - Missing imports: Stub modules generated to prevent errors
   - CSS imports: Extracted and injected as `<style>` tags
3. **Preview HTML**: Full HTML document with Tailwind CDN, import map, error boundary, and dynamic module loading
4. **PreviewFrame** (`src/components/preview/PreviewFrame.tsx`): Renders preview in sandboxed iframe

Key insight: The `@/` import alias maps to the root `/` directory in the virtual file system.

### Mock Provider

When no `ANTHROPIC_API_KEY` is set, a `MockLanguageModel` (`src/lib/provider.ts`) provides static responses to demonstrate the app without requiring API access. It generates predefined Counter/Form/Card components based on user prompts.

### Database Schema

Prisma with SQLite (`prisma/schema.prisma`):
- **User**: Email/password authentication
- **Project**: Stores serialized chat messages and virtual file system state (JSON in `messages` and `data` fields)
- Projects can be anonymous (userId is optional) or owned by authenticated users

### Authentication

JWT-based auth (`src/lib/auth.ts`):
- Session tokens stored in httpOnly cookies
- `createSession`, `getSession`, `deleteSession`, `verifySession` functions
- Middleware (`src/middleware.ts`) protects routes
- Anonymous users can use the app but data isn't persisted across sessions

### Context Architecture

Two primary React contexts manage application state:
- **FileSystemContext**: Manages virtual file system, selected file, CRUD operations, tool call handling
- **ChatContext** (`src/lib/contexts/chat-context.tsx`): Manages AI chat state with Vercel AI SDK

## Development Guidelines

### File Conventions
- Entry point for generated apps: Always `/App.jsx` (required by the system prompt)
- Import alias: `@/` resolves to virtual FS root (`/`)
- Preview uses Tailwind CSS loaded from CDN (not installed locally in virtual FS)

### Testing
- Tests use Vitest with jsdom environment
- Component tests in `__tests__` directories alongside source files
- Testing Library for React component testing

### Adding AI Tool Capabilities
To add new file operations the AI can perform:
1. Extend `VirtualFileSystem` class with the operation
2. Create a tool definition in `src/lib/tools/` using Vercel AI SDK's `tool()` function
3. Register the tool in `/api/chat/route.ts`
4. Add handling in `FileSystemContext.handleToolCall()`

### Database Migrations
After modifying `prisma/schema.prisma`:
1. Run `npx prisma migrate dev --name <description>`
2. Commit the migration files in `prisma/migrations/`
3. Run `npx prisma generate` to update client types

### Prisma Client Location
Prisma client is generated to `src/generated/prisma/` (not the default `node_modules/.prisma/client`). Import from `@/lib/prisma` which re-exports the generated client.

## Tech Stack
- **Frontend**: Next.js 15 (App Router), React 19, TypeScript, Tailwind CSS v4
- **AI**: Anthropic Claude (claude-haiku-4-5), Vercel AI SDK, Prompt caching
- **Database**: Prisma ORM, SQLite (dev.db)
- **Code Transformation**: Babel Standalone
- **Editor**: Monaco Editor
- **Testing**: Vitest, Testing Library
