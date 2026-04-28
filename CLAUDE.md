# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server (Next.js + Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest (unit tests)
npm run setup        # Install deps + generate Prisma client + migrate DB
npm run db:reset     # Reset SQLite database (destructive)
```

Run a single test file:
```bash
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx
```

## Environment

- `ANTHROPIC_API_KEY` — if set, uses real Claude `claude-haiku-4-5`; if absent, the app falls back to a `MockLanguageModel` with pre-scripted responses (see `src/lib/provider.ts`)
- `JWT_SECRET` — defaults to `"development-secret-key"`; override in production
- All scripts prepend `--require ./node-compat.cjs` to `NODE_OPTIONS` to strip `localStorage`/`sessionStorage` from `globalThis` at the Node level — this prevents SSR crashes on Node 25+

## Architecture

### Data flow: chat → code → preview

1. User types in `ChatInterface` → `MessageInput` → Vercel AI SDK `useChat` hook (via `ChatContext`)
2. `POST /api/chat` (route.ts) calls the language model with two AI tools: `str_replace_editor` and `file_manager`
3. Claude responds with tool calls that create/edit files in a **virtual file system** (in-memory `Map<string, string>` — see `src/lib/file-system.ts`)
4. `FileSystemContext` propagates updates to both the Monaco `CodeEditor` and `PreviewFrame`
5. `PreviewFrame` compiles the current entry file through Babel standalone at runtime and renders it in an `<iframe>` with an ImportMap injecting React/Tailwind

### AI tooling

`src/lib/tools/str-replace.ts` — primary tool: creates files or applies targeted string replacements (no full rewrites needed for edits).  
`src/lib/tools/file-manager.ts` — secondary tool: moves/deletes/renames files inside the virtual FS.  
`src/lib/prompts/generation.tsx` — system prompt sent to the model; uses `providerOptions.anthropic.cacheControl: "ephemeral"` for prompt caching to reduce token costs.

### Authentication & persistence

- JWT stored in an httpOnly cookie; verified in `src/middleware.ts` before every protected route
- Passwords hashed with bcrypt; JWT signed/verified with `jose` (`src/lib/auth.ts`)
- Prisma/SQLite stores `User` and `Project`; a Project serialises the full virtual FS + message history as JSON strings
- **Anonymous users** have their in-progress work tracked via `localStorage` (`src/lib/anon-work-tracker.ts`); projects are optionally saved after sign-in

### Path alias

`@/*` resolves to `src/*` (configured in `tsconfig.json` and mirrored in `components.json` for shadcn/ui).

### Testing

Tests live in `__tests__/` sub-directories next to the code they test. The Vitest config uses `jsdom` as the test environment with `@testing-library/react`.

## Database schema

The database schema is defined in the `prisma/schema.prisma` file. Reference it anytime you need to understand the structure of data stored in the database.

## Code style

- **Comments:** Only add a comment when the *why* is non-obvious (hidden constraint, subtle invariant, non-obvious workaround). Skip comments that describe what the code does — well-named identifiers already do that.
