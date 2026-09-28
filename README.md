# MightyWorks Job Coordinator

MVP design for an AI job-coordinator platform: a job seeker uploads a CV, employers post
listings, and the platform scores the match, badges strong fits, and generates a short
candidate persona for each listing.

## Status

Design phase. No code written yet.

## Locked decisions

| Area | Choice |
|---|---|
| Stack | TypeScript end-to-end — Bun + Hono API + React/Vite frontend |
| Database | PostgreSQL 17 in Docker from day one (one dialect everywhere) |
| CV intake | Manual paste, PDF/DOCX upload, and optional public-URL fetch |
| Job source | Pluggable `JobSource` interface; paste ships first, board API adapters later |
| Notification | Auto-applied 👍 badge on the job card when `score >= threshold` |
| Product sides | Seeker-first, but the persona engine ships in the same MVP |
| Auth | Seeded demo user, no login screen (schema already carries `user_id`) |
| Deploy target | Single VPS, one `docker compose` file |

---

## 1. The core design decision

The naive way to do "job listing percentage matching" is embedding the CV text and the job
text, then taking cosine similarity as a percentage. **Don't do this.** Cosine similarity is
not a match percentage — a well-written CV for a backend role scores ~0.72 against a frontend
JD because they share vocabulary like "team", "cloud", "API", without sharing a single
required skill. It's also unexplainable, so a user staring at "71% match" has no idea whether
to trust it, and therefore won't act on the badge.

Instead: **deterministic weighted scoring, with the LLM as a bounded reranker instead of the
score generator.**

The LLM's job is extraction and nuance, not arithmetic. It extracts structure from
unstructured text (once, cached), and it makes bounded judgment calls on adjacency. The
percentage is computed by code the user can inspect and that returns the same answer every
time.

```
base = 100 × (0.60·required_skill_coverage
            + 0.20·preferred_skill_coverage
            + 0.10·seniority_fit
            + 0.10·domain_fit)

final = clamp(base + llm_adjustment, 0, 100)   // llm_adjustment ∈ [−15, +15]
final = min(final, hard_cap)                    // e.g. 40 if location gate fails
```

This gives three properties that matter commercially:

- **Explainable.** The UI renders the four components as bars and the LLM's cited evidence.
  The user sees *why* it's 78%.
- **Cheap.** Scoring is pure code. LLM calls happen only on extraction (cached by content
  hash) and on first-time reranking.
- **Hard to break.** A hallucinating model can move the number 15 points, not 60. The
  classic failure — a model confidently reporting "92% match" for a completely mismatched
  pair — is structurally impossible.

The `hard_cap` concept matters as much as the weighting. A failed work-authorization or
location requirement is not "10% less relevant", it's a non-starter. Capping expresses that
honestly where a small penalty would not.

---

## 2. Architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌──────────────┐
│  React 19 +     │────▶│  Hono API (Bun)  │────▶│ PostgreSQL 17 │
│  Vite 8 + TW 4  │◀────│  :3001            │◀────│ (Docker)     │
└─────────────────┘     └────────┬─────────┘     └──────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │   LLMProvider (1 iface) │
                    └────────────┬────────────┘
                        ┌────────┴────────┐
              baseUrl env│                │baseUrl env
              ┌─────────▼──────┐   ┌─────▼──────────────┐
              │ Ollama (local) │   │ Any OpenAI-compat   │
              │ :11434         │   │ cloud key           │
              │ no API key     │   │ OpenAI/Groq/etc     │
              └────────────────┘   └────────────────────┘
```

The "local model or cloud key" requirement collapses into a **single adapter**. Ollama,
LM Studio, vLLM, llama.cpp's server, and every major cloud provider expose the OpenAI
chat-completions API. So there is one `openaiCompatible` provider and configuration is
entirely `LLM_BASE_URL` + `LLM_API_KEY` + `LLM_MODEL`. Flipping from local to cloud is a
`.env` edit, not a code change. An optional Anthropic adapter can be added only if Claude
specifically is wanted.

---

## 3. Repository layout

Bun workspaces monorepo. The `shared` package is load-bearing — it's the single source of
truth for the LLM's structured-output contract, so the API, the prompt, and the UI can never
drift.

```
MightyWorksJobCoordinator/
├── docker-compose.yml          # postgres:17-alpine, named volume, healthcheck
├── .env.example                # DATABASE_URL, LLM_BASE_URL, LLM_API_KEY, LLM_MODEL
├── package.json                # bun workspaces
├── packages/
│   ├── shared/                 # zod schemas + TS types (the contract)
│   │   └── src/
│   │       ├── candidate-profile.ts   # LLM output schema for CV parsing
│   │       ├── job-profile.ts         # LLM output schema for JD parsing
│   │       ├── rerank.ts              # LLM output schema for match reasoning
│   │       ├── persona.ts             # LLM output schema for employer persona
│   │       ├── score.ts               # ScoreBreakdown + scoring constants
│   │       └── job-source.ts          # JobSource interface
│   ├── db/                     # drizzle schema, migrations, seed
│   │   └── src/
│   │       ├── schema.ts
│   │       ├── client.ts              # postgres.js, reads DATABASE_URL
│   │       └── seed.ts                # demo user + sample profile + 5 jobs
│   └── llm/
│       └── src/
│           ├── provider.ts            # LLMProvider interface
│           ├── openai-compatible.ts   # the one adapter
│           └── prompts.ts
└── apps/
    ├── api/src/
    │   ├── index.ts            # Hono app, CORS, error handling, /api/health
    │   ├── routes/
    │   │   ├── profile.ts      # paste / file / url intake
    │   │   ├── preferences.ts
    │   │   ├── jobs.ts
    │   │   ├── matches.ts
    │   │   └── personas.ts
    │   ├── services/
    │   │   ├── cv/ingest.ts    # text / PDF (unpdf) / DOCX (mammoth) / URL (cheerio)
    │   │   ├── extract.ts      # cached LLM extraction, keyed by content hash
    │   │   ├── score.ts        # the deterministic formula above
    │   │   ├── rerank.ts       # bounded LLM adjustment
    │   │   ├── persona.ts
    │   │   └── jobsource/      # paste.ts, registry.ts, (adapters/adzuna.ts later)
    │   └── middleware/
    └── web/src/
        ├── pages/  Dashboard, Profile, JobDetail, Settings
        └── components/  ScoreRing, BreakdownBars, MatchBadge, PersonaCard
```

---

## 4. Database schema

```ts
users            id, email, name, created_at
profiles         id, user_id, headline, cv_text, cv_source,   // paste|file|url
                 source_url, years_experience, timestamps
profile_skills   id, profile_id, name, category, years, proficiency(1-5)
profile_roles    id, profile_id, title, seniority
profile_industries id, profile_id, name

preferences      user_id PK, desired_roles[], desired_locations[],
                 remote_only, min_salary, currency, match_threshold (default 70)

jobs             id, source,        // paste | api:adzuna | ...
                 external_id, title, company, location, remote,
                 description_text, posted_at, raw_payload jsonb

// Caching layer — keyed by content hash so re-scoring is free
profile_extractions  id, profile_id, content_hash, provider, model,
                     schema_version, extracted jsonb
job_extractions      id, job_id, content_hash, provider, model,
                     schema_version, extracted jsonb

job_personas    id, job_id, archetype, summary, must_haves[],
                nice_to_haves[], screening_questions[], provider, model

matches         id, profile_id, job_id, score, baseline_score,
                llm_adjustment, verdict, breakdown jsonb,
                evidence jsonb, gaps jsonb, blocked_by, cap_applied,
                model, created_at
                UNIQUE(profile_id, job_id)     // idempotent recompute
```

Two decisions worth calling out:

**There is no `notifications` table.** The requirement is an indicator, and an indicator is
not an event. `aboveThreshold` is derived at read time as
`score >= preferences.match_threshold`. So moving the threshold slider instantly re-badges
every card with zero queries, zero backfill jobs, and zero chance of stale or duplicated
notifications. If real push/email is wanted later, that becomes a small feature reading the
same derived flag — but it isn't in the MVP.

**`matches` is cached with a unique constraint.** Scoring is deterministic, so
`(profile_id, job_id)` recomputation is idempotent and skipped entirely on cache hit.
Combined with content-hash-keyed extractions, the only situation that costs an LLM call is
genuinely new content.

---

## 5. LLM contract

Four structured outputs, all defined in `packages/shared` as zod schemas and all passed to
the provider as a JSON schema for constrained decoding:

| Call | Input | Output | Cached |
|---|---|---|---|
| `extractCandidate` | CV text | skills w/ proficiency+years, roles, seniority, industries, years exp | by CV hash |
| `extractJob` | JD text | required/preferred skills, seniority, domain, hard requirements, responsibilities | by JD hash |
| `rerankMatch` | both extractions + baseline | `adjustPoints` ∈ [−15,15], verdict, evidence[], gaps[] | per match pair |
| `generatePersona` | job extraction | archetype, summary, must-haves, nice-to-haves, screening questions | per job |

Note the persona reuses the existing `job_extractions` row rather than re-reading the JD — so
the employer-side feature costs one extra cheap call, not a second parse.

---

## 6. API surface

```
GET    /api/health

GET    /api/profile                    # profile + extracted skills
POST   /api/profile/cv/paste           { cvText }
POST   /api/profile/cv/file            multipart
POST   /api/profile/cv/url             { url }  → fetch, strip to text
PUT    /api/profile/skills             # user corrections to extracted data
DELETE /api/profile

GET    /api/preferences
PUT    /api/preferences                # includes match_threshold

POST   /api/jobs                       { title, company, location, descriptionText }
GET    /api/jobs?minScore=&q=&page=    # list w/ score + aboveThreshold
GET    /api/jobs/:id                   # + full match breakdown
POST   /api/jobs/:id/match             # compute or return cached
POST   /api/jobs/match/batch           # recompute all
GET    /api/jobs/:id/persona
POST   /api/jobs/:id/persona
```

---

## 7. Frontend

- **Dashboard** — the job feed. Each card: score ring, 👍 badge (auto, derived),
  title/company/location, and a "why" popover with the breakdown bars and cited evidence.
- **Profile** — CV intake (paste / upload / URL), a review-and-edit surface for extracted
  skills (the LLM's output is a *draft*, not truth), and preferences with a threshold slider.
- **Job detail** — full breakdown, evidence, gaps, and the employer-side persona tab.
- **Settings** — LLM provider status and a test-call button, so local-model misconfiguration
  is visible immediately.

The editable extraction surface is deliberate: LLM-extracted skills will be occasionally
wrong, and a user who can't fix them will stop trusting the score permanently.

---

## 8. Build phases

| # | Phase | Done when |
|---|---|---|
| 1 | Scaffolding | `bun install` works; `docker compose up` + migrations + seed; `GET /api/health` returns DB status |
| 2 | Profile intake | CV paste/PDF/DOCX/URL lands in Postgres; extracted skills render in UI |
| 3 | LLM provider | One adapter drives both Ollama and a cloud key; a test call proves either config works |
| 4 | Scoring + jobs | Paste a JD → score appears with breakdown bars; matches cached; `UNIQUE` prevents duplicates |
| 5 | Threshold + badge | Slider re-badges the feed instantly; batch match endpoint |
| 6 | Persona | Employer persona generated per job from the existing extraction |

---

## 9. Risks

1. **Local models are weaker at strict JSON schema adherence than cloud models.**
   Mitigation: tolerant JSON extraction (brace-matching + repair), zod validation, and one
   repair retry. Budget for the first local-model iteration to be fiddly.
2. **LinkedIn URL fetch will usually fail.** Profiles sit behind an auth wall and LinkedIn's
   ToS prohibits automated collection. Anti-bot evasion will not be built. The adapter
   attempts a public fetch and, on failure, returns a clear "paste your CV instead" message.
   Paste and file are the real intake mechanism; URL fetch is a convenience.
3. **Prompt injection via job descriptions.** A hostile JD could try to manipulate the model.
   Mitigation: JD text is passed as delimited data with explicit instructions, and the model
   may only return a schema-validated object. Combined with the ±15 clamp, the blast radius is
   small.
4. **TypeScript 7.0 is now `latest`** (the Go-native compiler). Vite/Drizzle tooling
   compatibility is less battle-tested than TS 5.9. Recommendation: pin `typescript@5.9` for
   the MVP and revisit once the ecosystem catches up.

---

## 10. Out of scope for MVP

Real auth, job-board API adapters, email/push notifications, multi-employer workflows,
embeddings-based semantic matching, and CV version history.

---

## Planned dependency versions

Verified against the registry at time of writing:

| Package | Version |
|---|---|
| hono | 4.13.10 |
| zod | 4.6.5 |
| drizzle-orm / drizzle-kit | 0.45.3 / 0.31.11 |
| postgres (postgres.js) | 3.4.9 |
| react | 19.3.0 |
| vite | 8.3.1 |
| tailwindcss | 4.3.3 |
| typescript | 5.9 (pin; 7.0 is latest but riskier for tooling) |
| unpdf (PDF text) | 1.8.1 |
| mammoth (DOCX text) | 1.13.0 |
| cheerio (HTML → text) | 1.2.0 |
