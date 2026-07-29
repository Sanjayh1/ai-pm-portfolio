# ContractIQ

AI-powered contract review. Upload an NDA or MSA, get key terms extracted with page-level citations and confidence scores, then ask follow-up questions grounded in the actual document.

**Live:** [contractiq-sanjay.netlify.app](https://contractiq-sanjay.netlify.app/)

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Frontend | Next.js | Fixed by design docs; App Router handles both UI and API in one deployable |
| Backend | Next.js API Routes | PRD left this ambiguous (Edge Functions vs. hosted Node API). Route Handlers avoid a second deployment target/runtime since Next.js was already the frontend |
| Database | Supabase (Postgres) | Managed Postgres with built-in Row-Level Security — fits a single-tenant, per-user data model without custom auth-layer plumbing |
| Auth | Supabase Auth | Native fit with Supabase Postgres; RLS policies key off `auth.uid()` directly |
| File storage | Supabase Storage | Same platform as DB/auth, same RLS model extends to storage buckets |
| LLM | OpenAI GPT-4o | PRD's cost model, JSON-mode strategy, and prompt design were all built around GPT-4o specifically — not a swappable choice at this stage |
| Hosting | Netlify | Git-connected deploys, built-in env var management, native Next.js support |
| Access control | Postgres RLS | Enforced at the database layer, not just app code — a second, independent line of defense against data leaking across users |

## Architecture Decisions

Three decisions were made explicitly during planning rather than left to default:

- **Single user role for MVP.** PRD scoped team workspaces as post-MVP. RLS on `user_id` covers "every user sees only their own data" without building out a permissions/roles system for a feature not yet in scope.
- **First-class PDF fallback.** Functional requirements required the text-viewer fallback to match the primary PDF.js viewer's page-navigation and highlight behavior exactly — not a degraded secondary path.
- **OpenAI over Anthropic.** A stale README line referenced Claude; the PRD's actual cost/latency assumptions and JSON-mode strategy were built around GPT-4o. PRD treated as source of truth, README corrected to match.

## Security

Verified before deploy, not after:

- Security headers present (`X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection`)
- Unauthenticated requests to protected API routes return `401`, not data
- No API key values present in client-rendered page source
- `.env.local` confirmed absent from `git status` and from full git history (`git log --all --full-history`)
- RLS policies scoped to `auth.uid()` on both database tables and storage bucket objects

## Deploy Notes

Two build failures on first deploy, both resolved without code changes:

1. **Secrets scanning false positive** — Netlify flagged `NEXT_PUBLIC_SUPABASE_ANON_KEY` as an exposed secret. It's intentionally public (protected by RLS, not secrecy) — suppressed via `SECRETS_SCAN_OMIT_KEYS`, not disabled globally.
2. **Missing peer dependency** — `@opentelemetry/api` wasn't resolving during edge function bundling. Added via `npm install @opentelemetry/api`; only surfaced after the first error was cleared.

## Build Process

Built stage-by-stage with Claude Code: engineering plan → implementation specs → frontend scaffold → feature implementation (auth, upload, extraction, chat), with an explicit human sign-off gate before each stage's code was written.
