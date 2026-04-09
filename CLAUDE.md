# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # Initial setup: install deps + Prisma generate + migrate
npm run dev          # Start dev server with Turbopack (localhost:3000)
npm run build        # Production build
npm run lint         # Run ESLint
npm run test         # Run Vitest unit tests
npm run db:reset     # Reset SQLite database
```

Run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

## Architecture

**UIGen** is an AI-powered React component generator with live preview. Users describe components in natural language; Claude AI generates them into a virtual (in-memory) file system that renders in a live preview panel.

### Key Layers

**AI Generation Pipeline** (`src/app/api/chat/route.ts`, `src/lib/provider.ts`, `src/lib/prompts/`)
- Chat endpoint streams AI responses using Vercel AI SDK
- Two providers: real Anthropic Claude (`claude-haiku-4-5`) when `ANTHROPIC_API_KEY` is set, or `MockLanguageModel` fallback that generates sample components
- AI is given two tools: `str_replace_editor` (create/modify files) and `file_manager` (manage file structure)
- Max 40 steps (real) or 4 steps (mock); max 10,000 tokens per request

**Virtual File System** (`src/lib/file-system.ts`, `src/lib/contexts/file-system-context.tsx`)
- All generated files live in-memory; nothing is written to disk
- Root is `/`; components use the `@/` import alias
- Entry point is always `/App.jsx`
- State is serialized to JSON for database persistence

**Database & Auth** (`src/lib/auth.ts`, `src/lib/prisma.ts`, `prisma/schema.prisma`)
- SQLite via Prisma; two models: `User` and `Project`
- `Project` stores `messages` and `data` (serialized virtual FS) as JSON blobs
- JWT sessions in HTTP-only cookies; 7-day expiry
- `src/middleware.ts` protects `/api/projects` and `/api/filesystem`
- Anonymous users can generate without logging in; projects persist only for authenticated users

**Frontend** (`src/app/`, `src/components/`)
- Next.js 15 App Router with React 19
- `src/app/page.tsx` handles auth redirect; `src/app/main-content.tsx` is the main shell
- `src/components/chat/` — chat interface and message rendering
- `src/components/preview/` — live component preview using Babel standalone for JSX transform
- `src/components/editor/` — Monaco Editor integration
- `src/components/ui/` — Shadcn/ui primitives (New York style)

**Server Actions** (`src/actions/`)
- `create-project`, `get-projects`, `get-project` — thin wrappers around Prisma calls with auth checks

### Import Aliases

`@/*` maps to `./src/*` (configured in `tsconfig.json` and `components.json`).

### AI System Prompt Conventions

The generation prompt (`src/lib/prompts/generation.tsx`) instructs the AI to:
- Always use Tailwind CSS (no plain CSS or CSS modules)
- Start with `/App.jsx` as the entry point
- Use `@/` alias for local imports
- Keep responses brief; no unsolicited summaries
