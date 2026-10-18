# The generated CLAUDE.md

Write it at the repo root (and `AGENTS.md` as a one-line `@CLAUDE.md` import or symlink).
Fill every `<…>`; drop sections for surfaces that were not generated.

---

```markdown
# <Name>

<one-line description>

## Working principles

1. **Don't assume. Don't hide confusion. Surface tradeoffs.**
2. **Minimum code that solves the problem. Nothing speculative.**
3. **Touch only what you must. Clean up only your own mess.**
4. **Define success criteria. Loop until verified.**

## Ticket flow

A ticket pasted in chat — <a card link | an issue link | any task text> — **is the
instruction to run this whole flow, end to end, without asking whether to start.** Stop and
ask only for product decisions the ticket does not answer (pricing, copy promises, abuse
limits), never for mechanics.

1. Read the ticket fully. Check for an existing PR and branch (<`gh pr list --state all` |
   `git ls-remote origin 'refs/pull/*'` | `git branch -a`>).
2. Work in **a worktree**, never by switching branches in the main checkout (other sessions
   share it). New work: `make worktree name=<slug> branch=<type>/<slug>` (`feat/`, `fix/`,
   `chore/`, `perf/`; named right from the start — a PR's branch cannot be renamed); add
   `base=origin/<branch>` to stack on an open PR. An existing branch: its worktree, then
   **`make update` before writing code**. <Move the card to In progress.>
3. Implement test-first. A test proves a fix only if it failed before the fix.
4. Verify: `pnpm format && make check`, then check the change in a browser against the local
   stack, signed in as the dev account (below). A change to the app also extends the E2E
   suite.
5. Update the vault on this branch (docs/vault/CLAUDE.md).
6. Push and open the PR: <`gh pr create` | `git push origin <branch>:refs/for/main/<topic>
   -o title="…" -o description="…"` (AGit; pushing again updates it) | `gh pr create`>.
   Title: the effect in a plain sentence. Body: what changed, why, how it was tested, any
   deploy step, the ticket. <Move the card to In review.>
7. **Right after opening the PR, without being asked,** run `/pr-review-toolkit:review-pr` on
   the PR's own diff (`git diff origin/main...HEAD`; a stacked PR's base is its parent branch).
   There are no other reviewers, so this is the review. Fix every Critical and Important
   finding on the same branch without waiting, re-run the checks, push, and post one PR comment
   listing each finding and its fix; declining a finding is a stated ruling, not silence. A
   finding that needs a product decision stays unfixed and goes to the owner. Then report the
   same in chat. The PR is ready for the owner only after this step.
8. **Merge only on the owner's go.**

## Worktrees

- `make worktree` creates `worktrees/<PR number>--<slug>` from the latest `origin/main`
  (the number is the branch's PR if it has one, else the next free number), links the
  gitignored env files from the main checkout, runs `pnpm install` and builds the Prisma
  client. A worktree made any other way: run `make worktree-setup` first (without its own
  `node_modules`, Node resolves the main checkout's and tests run against `main`'s deps).
- `make update` merges `origin/main` (or the stacked base) into the branch and syncs deps; on
  a conflict it changes nothing and prints the `git merge` to run.
- Remove a worktree with `git worktree remove` once its PR is merged or closed (check
  `git status` there first).

## Local login

`dev@<slug>.localhost` / `<slug>-dev`, email verified. `make seed-dev` creates or resets it.
Use it for every browser check; never type a real person's credentials anywhere. Locally,
emails aren't sent: the API logs each one with its link.

## Git

- **Never `--no-verify`.** There is no CI: the husky hooks (`quality:check`) and `make check`
  before every push are the whole gate. Fix the cause or stop and ask.
- **Never force-push, never rebase a pushed branch.** To catch up: `git fetch origin main &&
  git merge origin/main`. Same for `commit --amend` or `reset --hard` on pushed commits.
- One feature = one PR; review fixes go on the same branch.
- No AI attribution anywhere (no `Co-Authored-By`, no "Generated with").
- Screenshots and test evidence stay out of the repo.

## Stack rules

<the table from references/stack.md, the "Zod both ways" and "Luxon only" sections, verbatim>

## Commands

`make up` · `make migrate-dev name=…` · `make seed-dev` · `pnpm dev` · `pnpm quality:check` ·
`make test-int` · `make test-e2e` · `make check` (all of the gate: quality:check, test-int,
test-e2e) · `pnpm api-client:generate` (after any API change) · `pnpm env:pull` (after
editing `sops envs/local.env`).

## Env

Source of truth: `envs/*.env`, encrypted with sops + age. Edit with `sops envs/<name>.env`,
then `pnpm env:pull`. Never edit the plain `.env` by hand; never commit a plain file in
`envs/` (pre-commit refuses). A new variable goes into `envs/schemas/<env>/.env.schema` and
the env file together; `pnpm env:validate` checks both.

## Traps

<the entries from references/traps.md that apply>

## Attribution

<a short version of references/utm.md, including the consent note: the attribution cookie is
written only after analytics consent; whether that is legally required is the owner's call>

## Keeping the docs current

`docs/vault/` is part of the work (see docs/vault/CLAUDE.md). A Stop hook blocks once per
branch when code changed and the vault did not: update it, or say why no update is needed.

## Manual setup (not done by the scaffold)

- [ ] sops: age key in `~/.config/sops/age/keys.txt`, its public key in `.sops.yaml`
- [ ] Stripe: products and prices, webhook to `<api>/api/auth/stripe/webhook`, price ids in envs
- [ ] PostHog project (EU) → key in envs
- [ ] Sentry projects (api, app, extension) → DSNs in envs
- [ ] Resend: sending domain verified (an unverified sender fails every email),
      `RESEND_API_KEY` and `MAIL_FROM` in `envs/production.env`
- [ ] DNS, Dokploy app (API), Vercel projects (web, app)
```
