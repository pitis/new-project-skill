---
name: new-project
description: Start a new project on the owner's standard stack and structure (pnpm + Turborepo monorepo, NestJS API, Postgres 18 with uuidv7, Prisma 7, Better Auth, Stripe, PostHog, Sentry, Luxon, Zod both ways; Astro web, Nuxt/Vue app, WXT+Vue extension) and generate a working, verified monorepo with its CLAUDE.md, vault and ticket→worktree→PR flow. Use when the user says "new project", "start a project", "scaffold", "bootstrap", or describes an app they want to begin building.
---

# New project

Generate a **working** project in the owner's structure. The structure and rules are in
`references/` — read all of them before writing a file; they are the whole spec.

## 1. Ask (one AskUserQuestion call)

1. **Name and one-line description.** Derive the slug: lowercase, non-alphanumerics → `-`,
   prefix `app-` if it starts with a digit. Create it at `./<slug>` in the current
   directory; refuse if that exists and is non-empty.
2. **Surfaces** (multi-select). Always: `apps/api`, `packages/db`, `packages/core`,
   `packages/api-client`. Optional: `apps/web` (Astro landing), `apps/app` (Nuxt/Vue app),
   `apps/extension` (WXT + Vue).
3. **Tickets**: GitHub Projects board / Gitea issues / pasted text only. Decides the git
   remote, PR mechanics and ticket steps in CLAUDE.md.
4. **Stripe billing?** **Background jobs (BullMQ + Redis)?**

## 2. Generate

Follow `references/structure.md` (what goes where) and `references/stack.md` (versions and
choices). Use current stable versions at generation time, except the pins listed in
`stack.md`. Write the project's CLAUDE.md from `references/claude-md.md`, the vault from
`references/vault.md`, attribution from `references/utm.md`. Avoid everything in
`references/traps.md` — each entry already cost a debugging session.

**Secrets.** Put the owner's age public key (`age-keygen -y ~/.config/sops/age/keys.txt`) in
`.sops.yaml`; if there is no key, ask before creating one with `age-keygen -o` at that path.
Write `envs/local.env` (local URLs and ports, the dev auth secret) and encrypt it in place
with `sops encrypt -i`.

Order: root workspace → `packages/db` → `packages/core` → `apps/api` → `packages/api-client`
→ frontends → CLAUDE.md + vault + scripts → `git init && git add -A` (staged, not
committed: `api-client:check` compares the regenerated client with the index).

## 3. Prove it works — the project is not done until all of this passes

1. `pnpm install` (no ignored-build errors — see traps).
2. `make up`, then `pnpm env:pull`, then `make migrate-dev name=init` creates and applies the
   first migration, then `make seed-dev` creates the dev account.
3. `pnpm format && git add -A && pnpm quality:check` (env check, Prettier, lint, typecheck,
   unit tests, api-client drift).
4. `make test-int`: the API integration suite against the `_test` database, which must
   prove: `/health` 200; a response field not in the `@ZodResponse` schema is stripped; an
   invalid body is 400; sign-up through Better Auth stores a user whose id has version
   nibble `7`; `/v1/me` without a session is 401; sign-up puts a verification email in the
   mail outbox and its link marks the email verified; a password reset through the outbox
   link lets the user sign in with the new password; with jobs, both emails went through the
   `mail` queue.
5. Boot the API (`pnpm --filter @<slug>/api dev`), `curl /health` and `/docs-json`, and sign
   in as the dev account (`POST /api/auth/sign-in/email` → 200).
6. Each frontend builds. With `apps/app`, `make test-e2e` passes.

A failing step is fixed at its cause. Never `--no-verify`, never weaken a test to pass.
Then `git add -A` and the first commit `Initial scaffold` (hooks run; no AI attribution).

## 4. Report

What was generated, the gate results, and the manual setup left (from CLAUDE.md's checklist):
Stripe products and webhook, PostHog project, Sentry projects, Resend domain and key, sops
age key, DNS, API hosting, Vercel.
