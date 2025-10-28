# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Quick Start Commands

```bash
# Development
npm run dev              # Start Next.js dev server (Turbopack) on port 3008

# Building & Testing
npm run build           # Production build
npm start              # Start production server
npm run lint           # ESLint + Next.js linting

# Database
npx prisma migrate dev  # Create and apply migrations
npx prisma studio     # Open Prisma Studio GUI

# Background Jobs
npm run dev            # Inngest runs alongside Next.js at http://localhost:8288
```

## Architecture Overview

This is a **Next.js 15 AI Code Generation Platform** that lets users create interactive web applications (like Netflix clones, Kanban boards, etc.) by describing what they want. The system generates code in isolated sandboxes, with authentication, rate limiting, and background job orchestration.

### Core Stack

- **Frontend:** React 19 + TypeScript + shadcn/ui (Radix UI) + Tailwind CSS
- **Backend:** Next.js 15 App Router (server components + API routes)
- **RPC Layer:** tRPC (type-safe client-server communication) with React Query
- **Database:** PostgreSQL + Prisma ORM
- **Auth:** Clerk (user authentication & plan detection)
- **Background Jobs:** Inngest (serverless workflow orchestration)
- **Code Sandbox:** E2B (isolated Node.js/Next.js execution environment)
- **AI Agent:** Inngest Agent Kit (multi-agent LLM orchestration)

### Directory Structure

```
src/
├── app/                  # Next.js App Router pages & API routes
│   ├── (home)/          # Public pages (landing, pricing)
│   ├── (auth)/          # Clerk auth routes
│   ├── projects/        # Dynamic [projectId] editor page
│   ├── api/
│   │   ├── trpc/[trpc]/ # tRPC handler (all server procedures)
│   │   └── inngest/     # Inngest webhook for background jobs
│   └── layout.tsx       # Root layout with providers
├── modules/             # Feature modules (server + UI)
│   ├── projects/        # Project management (create, fetch, list)
│   ├── messages/        # Chat messages for projects
│   └── usage/           # Credit/rate-limit tracking
├── trpc/               # tRPC setup & routing
├── inngest/            # Background job functions (code generation agent)
├── components/         # Shared UI components (navbar, file explorer, etc.)
├── lib/                # Utilities (Prisma, usage, helpers)
├── hooks/              # React hooks
├── config/             # Constants & config
├── middleware.ts       # Clerk auth middleware
└── prompt.ts          # LLM system prompts
```

## Key Architectural Patterns

### 1. tRPC + React Query Pattern

**File:** `/src/trpc/`, `/src/modules/*/server/procedures.ts`

tRPC exposes type-safe RPC procedures grouped by feature:

```
GET /api/trpc/projects.getMany          # Fetch user's projects
GET /api/trpc/projects.getOne           # Fetch single project
POST /api/trpc/projects.create          # Create project + trigger code generation
POST /api/trpc/messages.create          # Add message + regenerate code
GET /api/trpc/messages.getMany          # Fetch messages for a project
GET /api/trpc/usage.status              # Check remaining credits
```

All procedures are **protected** with `isAuthed` middleware. The client uses React Query for automatic caching and refetching. When adding new features:

1. Define server procedure in `modules/[feature]/server/procedures.ts`
2. Export from router in `modules/[feature]/server/router.ts`
3. Aggregate in `/src/trpc/routers/_app.ts`
4. Use in components via `trpc.[router].[procedure]`

### 2. Authentication (Clerk)

**Files:** `middleware.ts`, `app/layout.tsx`, `trpc/init.ts`

- Middleware protects all routes except: `/`, `/sign-in`, `/sign-up`, `/api/*`
- Clerk JWT token available as `auth.userId` in tRPC context
- User plan detected via `has({ plan: "pro" })` for premium features
- All database records use `userId` for isolation

### 3. Database Schema (Prisma)

**File:** `/prisma/schema.prisma`

Three main models:

```typescript
Project {
  id, name, userId, createdAt, updatedAt
  messages -> Message[]
}

Message {
  id, projectId, role (USER|ASSISTANT), content, type (RESULT|ERROR)
  createdAt, updatedAt
  fragment -> Fragment?  // Optional; code output
}

Fragment {
  id, messageId, title (AI-generated)
  sandboxUrl (E2B sandbox URL)
  files (JSON) // { filename: code, ... }
}

Usage {  // Rate limiting state
  key (userId), points (credits remaining), expire
}
```

### 4. Code Generation Pipeline (Inngest)

**Files:** `/src/inngest/functions.ts`, `/src/app/api/inngest/route.ts`

When a project or message is created, an async event is sent:

```typescript
inngest.send({
  name: "code-agent/run",
  data: { projectId, messageId, userId }
})
```

The `codeAgentFunction` (async, up to 30 mins):

1. Retrieves last 5 messages from DB
2. Spins up E2B sandbox (Node.js/Next.js environment)
3. Runs multi-agent orchestration:
   - **Code Generation Agent:** Uses terminal & file tools to generate React/Next.js code
   - **Title Generator:** Creates a title for the generated code
   - **Response Generator:** Creates a summary message for the user
4. Saves assistant message + fragment (with code + sandbox URL) to database
5. Frontend polls message list (5s interval) and displays results

**Important:** The sandbox persists for 30 mins, allowing users to interact with the generated code via the preview iframe.

### 5. Rate Limiting

**File:** `/src/lib/usage.ts`

- Uses `rate-limiter-flexible` library with Prisma backend
- **Free Plan:** 1 point per 30 days
- **Pro Plan:** 2 points per 30 days
- Cost: 1 point per project/message creation
- Checked in `projects.create` and `messages.create` procedures

### 6. State Management

**Client-Side:**
- **React Query:** Server state (projects, messages, usage)
- **React Hooks:** Local UI state (active tabs, scroll position, etc.)
- **react-hook-form:** Form submission in `ProjectForm`

**Server-Side:**
- **tRPC Context:** Authentication state (userId, Clerk auth)
- **Database:** Persistent state (projects, messages, fragments)
- **E2B Sandbox:** Temporary state (generated files, running processes)

## Common Development Tasks

### Adding a New tRPC Procedure

1. Create file: `src/modules/[feature]/server/procedures.ts`
2. Export procedure:
   ```typescript
   export const createProcedure = protectedProcedure
     .input(z.object({ ... }))
     .mutation(async ({ ctx, input }) => { ... })
   ```
3. Create router: `src/modules/[feature]/server/router.ts`
4. Add to `src/trpc/routers/_app.ts`
5. Use in component:
   ```typescript
   const { mutate } = trpc.[feature].[procedure].useMutation()
   ```

### Adding a New UI Component

1. Create component file: `src/components/[name].tsx` or `src/modules/[feature]/ui/[name].tsx`
2. Use shadcn/ui components from `/src/components/ui/`
3. Import & use in pages or other components
4. Style with Tailwind classes

### Adding a Database Model

1. Update `prisma/schema.prisma`
2. Create migration: `npx prisma migrate dev --name [description]`
3. Regenerate types: `npx prisma generate` (auto-run on `npm install`)
4. Use typed queries in server procedures

### Triggering Background Jobs

When you need async processing (code generation, exports, etc.):

```typescript
await inngest.send({
  name: "function-name",
  data: { /* payload */ }
})
```

The function runs serverlessly via Inngest webhook. Check progress via frontend polling or database queries.

### Testing Rate Limits

```typescript
import { checkUsage } from "@/lib/usage"

const { isAllowed } = await checkUsage(userId, 1)
if (!isAllowed) {
  throw new Error("Rate limit exceeded")
}
```

## Important Files & Conventions

### Configuration

- `constants.ts` - Project templates, default prompts
- `prompt.ts` - LLM system messages for code agent
- `config/` - App-wide configuration

### Environment Variables

Create `.env.local`:

```
DATABASE_URL=postgresql://user:pass@host/db
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=...
CLERK_SECRET_KEY=...
INNGEST_EVENT_KEY=...
INNGEST_SIGNING_KEY=...
E2B_API_KEY=...
OPENAI_API_KEY=...  # For LLM agent
```

### Type Safety

- All tRPC procedures are typed end-to-end
- Prisma generates types in `/src/generated/prisma`
- Use Zod schemas for input validation
- No `any` types; prefer strict TypeScript

### Error Handling

- tRPC automatically catches errors and returns as `TRPCError`
- Use `try-catch` in server procedures
- Frontend shows errors via Sonner toast notifications
- React Error Boundary wraps components for graceful error display

### Styling

- Tailwind CSS v4 for all styling
- Use class merging with `clsx` and `tailwind-merge`
- shadcn/ui components for UI consistency
- Dark mode support via `next-themes`

## Performance Considerations

1. **React Query:** Results are cached and refetched automatically (5s default for messages)
2. **E2B Sandbox:** Persists for 30 mins to avoid re-creation overhead
3. **Fragment Caching:** Messages with fragments are stored in DB, not regenerated
4. **Turbopack:** Dev server uses Turbopack for fast compilation
5. **Code Splitting:** Dynamic imports for large components (none currently, but use `next/dynamic` if needed)

## Troubleshooting

### Build Errors
```bash
npm run lint  # Check for type/lint errors
npm run build # Full build test
```

### Database Issues
```bash
npx prisma reset         # Drop & recreate schema (dev only!)
npx prisma db push      # Sync schema to DB
npx prisma studio      # GUI for DB inspection
```

### Inngest Issues
- Check `/api/inngest` endpoint is running
- Monitor via Inngest dashboard (if configured)
- Logs in server console during `npm run dev`

### Rate Limit Not Working
- Ensure `DATABASE_URL` is set
- Check `Usage` table in database: `select * from "Usage"`
- Verify timestamp format matches `DateTime` in Prisma schema

## Deployment Notes

- Use Vercel for seamless Next.js deployment
- Database must be PostgreSQL (Prisma configured for it)
- Inngest requires webhook endpoint: `/api/inngest`
- E2B sandbox requires API key (check environment)
- Clerk requires production keys (different from dev keys)
- Rate limits use database TTL (ensure indexes on `Usage.expire`)
