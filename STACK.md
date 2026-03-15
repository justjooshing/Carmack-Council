# Stack Assumptions

The Carmack Council skills are opinionated for the stack below. If you use a different stack, fork and adapt. This file lists every stack-specific assumption, where it appears, and what to change.

## The Stack

| Layer | Technology | Alternative examples |
|-------|-----------|---------------------|
| Framework | TanStack Start | Next.js App Router, Remix, SvelteKit |
| Language | TypeScript (strict mode) | — |
| Data fetching | TanStack Query | tRPC, SWR, React Query |
| ORM | Drizzle | Prisma, Kysely, TypeORM |
| Database | Postgres | Neon, Supabase, RDS |
| Caching | Redis | Upstash, Vercel KV, in-memory |
| E-commerce | Shopify (Admin API, Storefront API, Polaris, Webhooks, Functions) | — |
| Styling | Inline styles, Tailwind, or CSS — check `conventions.md` | CSS Modules + BEM, styled-components |
| Auth | Project-specific — check `conventions.md` | Clerk, NextAuth, Lucia, Auth0 |

---

## Where Assumptions Live

### SKILL.md files — Stack Context sections

Every SKILL.md has a "Stack Context" section near the top. This is the primary place to update.

| File | What to change |
|------|---------------|
| `skills/council-review/SKILL.md` | Stack Context section. Update framework, data fetching, ORM, DB, caching, e-commerce, styling. |
| `skills/council-plan/SKILL.md` | Stack Context section. Same stack list. |
| `skills/council-implement/SKILL.md` | Stack Context section. Same stack list. |
| `skills/spec-writer/SKILL.md` | No stack context section — spec-writer is stack-agnostic by design. |

### SKILL.md files — Subagent prompts

The council-review and council-plan SKILL.md files contain subagent prompt templates that reference specific technologies.

| Pattern | Where it appears | What to change |
|---------|-----------------|---------------|
| "TanStack Query" / "server functions" | council-plan/review Collina and Dodds subagents | Replace with your data-fetching and API layer |
| "Drizzle" | council-plan/review Leach subagent | Replace with your ORM |
| "Postgres" / "Redis" | council-plan/review Leach and Collina subagents | Replace with your DB and cache |
| "Shopify" | council-plan Shopify subagent, council-review domain assignments | Remove or replace if not using Shopify |
| "Inline styles / Tailwind / CSS — check conventions.md" | council-plan Dodds and Saarinen subagents | Replace with your styling approach |
| Auth references | council-review Hunt subagent | Replace with your auth provider |
| `references/quality-shopify.md` | council-plan Shopify subagent, council-implement reference table | Remove or replace if not using Shopify |
| `references/quality-vertical-slice.md` | council-plan Cockburn subagent, council-implement reference table | Keep — delivery structure applies to all stacks |

### Reference documents

The reference docs contain stack-specific patterns and examples throughout. These are the most work to adapt.

| Reference | Stack assumptions | What to change |
|-----------|-----------------|---------------|
| `references/security.md` | Auth middleware, server function patterns, Drizzle queries | Replace with your auth/API/ORM security patterns |
| `references/quality-backend.md` | TanStack Start server functions, TanStack Query invalidation, Drizzle, Redis | Replace with your API layer, ORM, and cache patterns |
| `references/quality-frontend.md` | React, TanStack Start, TanStack Query hooks, styling check via conventions.md | Replace with your frontend framework and styling approach |
| `references/quality-postgres.md` | Drizzle ORM, Postgres | Core Postgres principles (schema design, migrations, transactions) are universal; replace ORM-specific examples |
| `references/quality-shopify.md` | Shopify Admin API, Storefront API, Polaris (React + Web Components), Webhooks, Functions, Metafields | Remove or adapt if not using Shopify |
| `references/quality-vertical-slice.md` | Stack-agnostic delivery methodology | No changes needed — applies to any stack |
| `references/quality-testing.md` | Vitest, Cypress | Replace with your test framework and tooling |
| `references/quality-llm.md` | General LLM pipeline principles — largely stack-agnostic | — |
| `references/quality-ui.md` | Styling varies by project — check conventions.md | Update for your design system, theme, fonts, and aesthetic |
| `references/quality-ux.md` | General UX patterns | Adjust for your product type and user context |
| `references/refactoring.md` | TanStack Start, Drizzle patterns | Replace with your stack's patterns. Core refactoring principles are universal |

### Spec-writer references

The spec-writer references (`references/spec-writer/`) are stack-agnostic. No changes needed for a different stack.

---

## How to Fork

1. **Edit this file** — Update "The Stack" table with your technologies.
2. **Update Stack Context sections** — In each SKILL.md, replace the Stack Context block.
3. **Update subagent prompts** — Search for the patterns listed above and replace.
4. **Update reference docs** — This is the heavy lift. Each reference doc has stack-specific examples woven throughout. Replace them with equivalent patterns from your stack. The principles themselves are universal — only the examples change.
5. **Rebuild** — Run `./scripts/build.sh` to produce updated `.skill` packages.

Start with the SKILL.md files (fast, high impact), then work through the reference docs one at a time.
