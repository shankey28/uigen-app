# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It uses Claude AI (via Anthropic API) to generate React components based on natural language descriptions, with a real-time preview environment powered by a virtual file system.

## Tech Stack

- **Frontend**: Next.js 15 (App Router), React 19, TypeScript
- **Styling**: Tailwind CSS v4
- **Database**: Prisma with SQLite
- **AI**: Anthropic Claude API (via Vercel AI SDK)
- **Testing**: Vitest with React Testing Library
- **Build**: Turbopack for development

## Common Commands

### Setup
```bash
npm run setup              # Install deps, generate Prisma client, run migrations
```

### Development
```bash
npm run dev                # Start dev server with Turbopack on port 3000
npm run dev:daemon         # Start dev server in background, logs to logs.txt
```

### Database
```bash
npx prisma generate        # Generate Prisma client
npx prisma migrate dev     # Run migrations
npm run db:reset           # Reset database (force)
```

### Testing
```bash
npm test                   # Run tests with Vitest
npm test -- --watch        # Run tests in watch mode
npm test -- <pattern>      # Run tests matching pattern
```

### Build & Deploy
```bash
npm run build              # Build for production
npm start                  # Start production server
npm run lint               # Run ESLint
```

## Architecture

### Virtual File System (VFS)

The core innovation is a virtual file system (`src/lib/file-system.ts`) that operates entirely in memory without writing to disk.

**Key classes:**
- `VirtualFileSystem`: In-memory file system with tree structure using `Map<string, FileNode>`
- `FileNode`: Represents files or directories with path, name, type, content, and children

**Important patterns:**
- All paths are normalized with leading `/` (e.g., `/App.jsx`, `/components/Counter.jsx`)
- Parent directories are created automatically when needed
- Supports serialization/deserialization for persistence in database
- Can be converted to/from `Record<string, FileNode>` or `Map<string, string>`

### AI-Powered Code Generation

The AI chat flow (`src/app/api/chat/route.ts`) uses:

1. **Vercel AI SDK** with streaming responses (`streamText`)
2. **Custom tools** that manipulate the VFS:
   - `str_replace_editor`: Create, view, and edit files (str_replace, insert)
   - `file_manager`: Rename and delete files/directories
3. **System prompt** (`src/lib/prompts/generation.tsx`): Instructs Claude to:
   - Create React components with `/App.jsx` as entry point
   - Use `@/` import alias for local files (e.g., `import Counter from '@/components/Counter'`)
   - Style with Tailwind CSS
   - Keep responses brief

### JSX Transformation & Preview

**Transform pipeline** (`src/lib/transform/jsx-transformer.ts`):
1. Parse JSX/TSX with Babel standalone
2. Transform to ES modules with React automatic runtime
3. Create blob URLs for each module
4. Generate import map with CDN URLs for external deps (esm.sh)
5. Handle `@/` alias resolution
6. Inject into HTML preview with error boundary

**Preview rendering** (`src/components/preview/PreviewFrame.tsx`):
- Sandboxed iframe with generated HTML
- Tailwind CDN for styling
- Error boundary catches runtime errors
- Syntax errors displayed with formatted messages

### Context System

Two main React contexts manage state:

1. **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`):
   - Wraps VirtualFileSystem instance
   - Provides hooks for file operations
   - Handles tool calls from AI responses
   - Auto-selects `/App.jsx` or first file

2. **ChatContext** (`src/lib/contexts/chat-context.tsx`):
   - Manages chat messages and streaming
   - Integrates with Vercel AI SDK's `useChat`
   - Persists messages to database for authenticated users

### Authentication & Projects

**Auth** (`src/lib/auth.ts`):
- JWT-based sessions with Jose library
- 7-day expiration
- HTTP-only cookies
- Supports anonymous users (no persistence)

**Database schema** (`prisma/schema.prisma`):
- `User`: email/password authentication
- `Project`: Stores serialized VFS (`data` JSON field) and chat messages
- Projects can be owned by users or anonymous (userId nullable)

**Anonymous work tracking** (`src/lib/anon-work-tracker.ts`):
- Tracks anonymous user activity for prompting sign-up

### Mock Provider

When `ANTHROPIC_API_KEY` is not set, a mock provider (`src/lib/provider.ts`) generates static components:
- Detects component type from prompt (counter/form/card)
- Simulates multi-step tool calling
- Limits to 4 steps to prevent repetition

## Key Patterns

### Import Alias Resolution

The `@/` alias maps to the root `/` of the VFS:
- `@/components/Foo` → `/components/Foo.jsx`
- Import map includes all variations (with/without extension, with/without leading slash)

### File Paths

Always use absolute paths starting with `/`:
- ✅ `/App.jsx`
- ✅ `/components/Button.tsx`
- ❌ `App.jsx`
- ❌ `components/Button.tsx`

### Tool Call Handling

AI tool calls are processed in two places:
1. **Server-side**: VFS modifications during streaming (`route.ts`)
2. **Client-side**: UI updates via `handleToolCall` in FileSystemContext

### Next.js App Router Structure

```
src/app/
  page.tsx              # Landing page with auth dialog
  layout.tsx            # Root layout
  main-content.tsx      # Main app UI (chat + editor + preview)
  [projectId]/page.tsx  # Project detail page
  api/chat/route.ts     # AI streaming endpoint
```

## Configuration Notes

- **Node compatibility**: `node-compat.cjs` provides Node.js polyfills for Turbopack
- **Path aliases**: `@/` configured in `tsconfig.json` for src imports
- **Prisma output**: Client generated to `src/generated/prisma` (non-standard location)
- **Environment**: Optional `ANTHROPIC_API_KEY` in `.env` (falls back to mock provider)
- **Max duration**: API route set to 120 seconds for long-running AI requests

## Testing Conventions

Tests use Vitest + React Testing Library:
- Test files: `__tests__/*.test.tsx` or `*.test.ts`
- JSDOM environment for React component tests
- Test file system operations use real VirtualFileSystem instances
- Mock `useChat` hook for testing chat components
