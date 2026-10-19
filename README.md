# new-project-skill

Start a SaaS project for Claude Code in one command. You describe the app; Claude asks four
questions, generates a monorepo on a fixed, opinionated stack, and doesn't report until the
project installs, migrates and passes its own gate.

```
/new-project-skill:new-project
/new-project-skill:new-project a CRM for wedding photographers
```

(Installed as plain skills, it is just `/new-project`.)

## What it asks

| Question | Options |
|---|---|
| Name and one-line description | The slug is derived from the name; the project lands in `./<slug>` |
| Surfaces | Always: `apps/api`, `packages/db`, `packages/core`, `packages/api-client`. Optional: `apps/web` (Astro landing), `apps/app` (Nuxt), `apps/extension` (WXT) |
| Where tickets live | GitHub Projects / Gitea issues / pasted text. Sets the remote, PR commands and ticket steps in the generated CLAUDE.md |
| Extras | Stripe billing? Background jobs (BullMQ + Redis)? |

## What you get

| Area | Choice |
|---|---|
| Monorepo | pnpm 12 workspaces + Turborepo, Node 24, ESM, TypeScript 6 |
| API | NestJS 12 (SWC), helmet, throttler, Swagger, **Zod both ways**: requests validated, responses parsed and stripped, OpenAPI built from the same schemas |
| Database | Postgres 18, Prisma 7, `uuidv7()` ids |
| Auth / payments | Better Auth (cookie + bearer), Stripe through `@better-auth/stripe` |
| Frontends | Astro landing, Nuxt 4 app, WXT + Vue extension, Tailwind 4, Orval-generated vue-query client |
| Email | Resend: verification and password-reset emails work from the first run; locally they're logged, not sent |
| Jobs (optional) | BullMQ + Redis, with email sending as the first real queue |
| Observability | Sentry 11 (privacy-safe defaults), PostHog EU |
| Attribution | UTMs carried end to end: landing → account → Stripe checkout → PostHog |
| Dates | Luxon only, enforced by ESLint |
| Quality | ESLint, Prettier, Vitest, integration tests against a real Postgres `_test` database, a Playwright smoke test, husky hooks + `make check` as the gate |
| Secrets | sops + age encrypted `envs/`, varlock schemas |
| Deploy | API: Dockerfile, for any container host. Web and app: static builds on Vercel or Cloudflare |

Plus the working setup around the code:

- **`CLAUDE.md`** (and `AGENTS.md`): working principles, git rules, stack rules, known traps,
  and a **ticket flow**. Paste a ticket and Claude creates a worktree, works test-first,
  passes the gate, updates the docs, opens the PR, reviews it with
  `/pr-review-toolkit:review-pr`, fixes the findings, and waits for your go to merge.
- **`docs/vault/`**: the project's knowledge base. A Stop hook blocks once per branch when
  code changed and the vault didn't.
- **`make worktree` / `make update`**: one worktree per ticket, with envs linked and deps
  installed.
- **A dev account** (`make seed-dev`) whose login is written in CLAUDE.md, so every browser
  check signs in the same way.

## Done means verified

The run ends only when all of these pass in the new project: `pnpm install`, the first
migration and the dev account, `pnpm quality:check` (env check, format, lint, typecheck,
unit tests, api-client drift), `make test-int` (health, response stripping, 400 on a bad
body, sign-up storing a uuidv7 id, 401 without a session, the verification and
password-reset emails end to end), the API booting and signing in the dev account, every
frontend building, and the Playwright smoke test (sign up, sign out, log in). Then it makes
the first commit.

## Install

**Claude Code plugin** (inside Claude Code):

```
/plugin install new-project-skill --marketplace pitis/new-project-skill
```

On Claude Code before 2.1.275, from a terminal:

```
claude plugin marketplace add pitis/new-project-skill
claude plugin install new-project-skill
```

**Skill only** (Claude Code, Cursor, Codex, via [skills.sh](https://skills.sh)):

```
npx skills add pitis/new-project-skill
```

The repo is private: both need git access to it (accept the collaborator invite first).
Restart Claude Code after installing.

## Requirements

- Node 24 with corepack (pnpm comes from the generated `packageManager` field)
- Docker (local Postgres 18, and Redis with background jobs)
- Playwright's Chromium (`pnpm exec playwright install chromium`) when the app is generated
- `make`, and bash ≥ 4 (macOS ships 3.2: `brew install bash`)
- `sops` and `age` for the encrypted env files, with an age key at
  `~/.config/sops/age/keys.txt` (the skill asks before creating one)
- `gh` if tickets live on GitHub
- The [`pr-review-toolkit`](https://github.com/anthropics/claude-plugins-official) plugin for
  the review step of the ticket flow

The scaffold doesn't create external accounts. The generated CLAUDE.md ends with a
checklist: age key, Stripe products and webhook, PostHog, Sentry, Resend domain and key, DNS,
API hosting, Vercel or Cloudflare.

## Layout

```
.claude-plugin/plugin.json        plugin manifest
.claude-plugin/marketplace.json   lets the repo be added as a marketplace
skills/new-project/SKILL.md       the workflow: ask, generate, verify, report
skills/new-project/references/    the spec: structure, stack, CLAUDE.md, vault, UTM, traps
```
