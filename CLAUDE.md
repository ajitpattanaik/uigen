# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It uses Claude (via Anthropic SDK) to generate React components based on natural language descriptions, displays them in real-time preview, and manages everything through a virtual file system (no files written to disk).

## Essential Commands

### Setup and Database
```bash
npm run setup              # Install deps + generate Prisma client + run migrations
npm run db:reset           # Reset database (caution: deletes all data)
npx prisma generate        # Regenerate Prisma client after schema changes
npx prisma migrate dev     # Create new migration after schema changes
```

### Development
```bash
npm run dev                # Start dev server with turbopack
npm run build              # Production build
npm run lint               # Run ESLint
npm test                   # Run all tests with Vitest
npm test -- --watch        # Run tests in watch mode
npm test -- path/to/test   # Run specific test file
```

### Node Compatibility
All npm scripts use `NODE_OPTIONS='--require ./node-compat.cjs'` to fix Node.js 25+ Web Storage SSR issues. The `node-compat.cjs` file removes non-functional `localStorage`/`sessionStorage` globals during SSR.

## Architecture

### Virtual File System (VFS)
- **Core:** `src/lib/file-system.ts` exports `VirtualFileSystem` class
- **Purpose:** All generated components exist only in memory, never written to disk
- **Persistence:** VFS state serialized to JSON and stored in Prisma `Project.data` field
- **Structure:** Tree of `FileNode` objects (type: "file" | "directory", with path, name, content, children)
- **Key Methods:**
  - `writeFile(path, content)` - Create/update files
  - `readFile(path)` - Read file content
  - `deleteFile(path)` - Remove files/directories
  - `rename(oldPath, newPath)` - Move/rename
  - `serializeToNodes()` / `deserializeFromNodes()` - Persistence

### AI Integration
- **Provider:** `src/lib/provider.ts` - Exports `getLanguageModel()` which returns either:
  - Real: `anthropic("claude-haiku-4-5")` when `ANTHROPIC_API_KEY` is set
  - Mock: `MockLanguageModel` for development without API key (returns hardcoded components)
- **API Route:** `src/app/api/chat/route.ts` - POST endpoint for streaming AI responses
- **Prompt:** `src/lib/prompts/generation.tsx` - System prompt for component generation
- **Tools:** AI has access to two tools defined in `src/lib/tools/`:
  - `str_replace_editor` - Find and replace in files (exact match replacement)
  - `file_manager` - Rename, move, or delete files/directories
- **Streaming:** Uses `streamText()` from Vercel AI SDK with max 40 steps (4 for mock)
- **Caching:** System prompt uses Anthropic prompt caching (`cacheControl: { type: "ephemeral" }`)

### Database & Authentication
- **ORM:** Prisma with SQLite (`prisma/dev.db`)
- **Schema:** Reference `prisma/schema.prisma` for complete database structure
- **Client:** Generated to `src/generated/prisma/` (not standard location)
- **Models:**
  - `User` - id, email, password (bcrypt hashed), timestamps, projects relation
  - `Project` - id, name, userId (nullable for anon), messages (JSON string), data (JSON string for VFS), timestamps
- **Auth:** Session-based JWT using `jose` library
  - `src/lib/auth.ts` - `getSession()`, `login()`, `logout()`, `signup()`
  - Uses httpOnly cookies named "auth-token"
  - Anonymous users can create projects without auth (userId = null)
- **Actions:** Server actions in `src/actions/` for CRUD operations on projects

### Component Structure
- `src/app/` - Next.js App Router pages and API routes
  - `page.tsx` - Landing page with project list
  - `[projectId]/page.tsx` - Main workspace for a project
  - `api/chat/route.ts` - AI streaming endpoint
- `src/components/`
  - `auth/` - Login, signup forms
  - `chat/` - Chat interface, message list, markdown renderer
  - `editor/` - Code editor with Monaco, file tree
  - `preview/` - Live component preview using iframe + Babel standalone
  - `ui/` - Shadcn UI components
- `src/lib/contexts/` - React contexts for file system and chat state
- `src/lib/transform/` - JSX transformer for preview rendering

### Preview System
- Uses `@babel/standalone` to transpile JSX in the browser
- Renders generated components in an iframe sandbox
- Hot reloads when files change in VFS
- Import alias `@/` maps to VFS root for component imports

## Important Conventions

### File Paths
- All VFS paths use forward slashes and start with `/`
- Entry point is always `/App.jsx` (required for every project)
- Non-library imports use `@/` alias: `import Foo from '@/components/Foo'`

### Code Generation
- Style with Tailwind CSS classes, not inline styles
- No HTML files - React components only
- Keep AI responses brief unless user asks for details
- Default export required in `/App.jsx`
- Use comments sparingly - only for complex code where logic isn't self-evident

### Testing
- Test framework: Vitest with jsdom environment
- React Testing Library for component tests
- Tests colocated in `__tests__/` directories
- Config: `vitest.config.mts` with React plugin and tsconfig-paths

### TypeScript
- Path alias `@/*` resolves to `./src/*`
- Target: ES2017
- Strict mode enabled
- Prisma client types imported from `@/generated/prisma`

## Common Tasks

### Adding a new AI tool
1. Create tool definition in `src/lib/tools/<tool-name>.ts`
2. Export a `build<ToolName>Tool(fileSystem)` function that returns `tool()` from Vercel AI SDK
3. Add to tools object in `src/app/api/chat/route.ts`

### Modifying the database schema
1. Edit `prisma/schema.prisma`
2. Run `npx prisma migrate dev --name <description>`
3. Run `npx prisma generate` (generates to `src/generated/prisma/`)

### Changing the AI model
- Edit `MODEL` constant in `src/lib/provider.ts` (currently "claude-haiku-4-5")
- Ensure model supports tool calling

### Adding new components
- Place in appropriate `src/components/<category>/` directory
- Import with `@/components/<category>/<Component>` alias
- Export from category index if needed for cleaner imports
