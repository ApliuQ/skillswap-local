# SkillSwap Local — Engineering Specification

## 1. Overview

SkillSwap Local is a hyperlocal skill-bartering platform. Users list skills they can teach and want to learn. The system matches them by skill complementarity, availability overlap, proximity, and trust score. Exchanges are settled in **time credits**: teaching earns credits, learning spends them. Community “quests” enable group projects.

**Stack**: Next.js 14 (App Router), TypeScript, Tailwind, shadcn/ui, Supabase (Auth / Postgres / Storage / RLS), Zod + React Hook Form, Vitest + Playwright, Vercel.

## 2. Data Model

| Table | Key Columns | RLS |
|---|---|---|
| `users` | id (auth.uid), display_name, trust_score, location (geography), created_at | Self read/update; public read of display_name/trust_score |
| `skills` | id, name, category | Public read |
| `user_skills` | user_id, skill_id, type (teach/learn), proficiency | Owner CRUD; matched users can read |
| `availability` | user_id, day_of_week, start_time, end_time | Owner CRUD; matched users can read |
| `swap_requests` | id, requester_id, provider_id, skill_offered, skill_requested, status (pending/accepted/declined/completed/cancelled), meeting_spot, created_at | Participants only |
| `credit_ledger` | id, user_id, amount (+/-), swap_id, created_at, idempotency_key (unique) | Owner read; insert via server-side RPC only |
| `reviews` | id, swap_id, reviewer_id, reviewee_id, rating (1-5), comment | Participants write; public read |
| `reports` | id, reporter_id, reported_user_id, reason, status | Reporter insert; admin read |
| `blocked_users` | blocker_id, blocked_id | Owner CRUD |
| `quests` | id, title, description, location, start_time, max_participants, created_by | Public read; authenticated create |
| `quest_participants` | quest_id, user_id, joined_at | Authenticated join |

**Credit ledger idempotency**: `idempotency_key` is unique per swap+side. The settlement RPC checks for an existing row before inserting, ensuring exactly-once credit movement even under retry.

## 3. Matching Formula (`/api/match`)

Score = `0.40 * skill_fit + 0.25 * availability_overlap + 0.20 * proximity + 0.15 * trust_score`

- **skill_fit**: binary complementarity — candidate teaches what requester wants to learn AND wants to learn what requester teaches. 1.0 or 0.0.
- **availability_overlap**: fraction of requester’s available hours that overlap with candidate’s within a 7-day window.
- **proximity**: inverse distance normalized to [0,1] within a configurable radius (default 5 km).
- **trust_score**: 0–1, derived from completed swaps and review average (Bayesian prior on cold start).

## 4. API Routes (App Router)

| Route | Method | Purpose |
|---|---|---|
| `/api/profile` | GET/PATCH | User profile CRUD |
| `/api/skills` | GET/POST/DELETE | User skill listings |
| `/api/availability` | GET/POST/DELETE | Availability windows |
| `/api/match` | POST | Return ranked matches for authenticated user |
| `/api/swaps` | POST/GET | Create / list swap requests |
| `/api/swaps/[id]` | PATCH | Accept / decline / complete / cancel |
| `/api/ledger` | GET | User’s credit balance and history |
| `/api/quests` | GET/POST | List / create quests |
| `/api/quests/[id]/join` | POST | Join a quest |

## 5. Acceptance Criteria (First Milestone)

- [ ] Deploys to Vercel on first commit with no build errors.
- [ ] Supabase migrations run cleanly from empty DB; all RLS policies verified with `anon` and `authenticated` roles.
- [ ] A user can sign up, create a profile, add teach/learn skills, and set availability.
- [ ] `/api/match` returns ranked results using seeded demo data with the exact scoring weights above.
- [ ] Playwright test: two users complete a swap → credits move once → ledger row count increases by exactly 2 (one debit, one credit). Re-running settlement with same idempotency key does not duplicate.
- [ ] Repo is public; SPEC.md committed; README points to SPEC.

## 6. Milestones

| Milestone | Deliverable |
|---|---|
| M1 | Scaffold + Vercel deploy + Supabase schema/RLS + auth + profile/skills/availability |
| M2 | `/api/match` + swap lifecycle (request → accept → complete) + credit settlement |
| M3 | Reviews, trust score computation, blocking/reporting |
| M4 | Quests, community features, demo seed data, Playwright E2E suite |