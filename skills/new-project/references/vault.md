# The vault (docs/vault/) — created and kept current automatically

The vault is the project's memory across sessions: how we know what the system does without
re-deriving it. It is created by the scaffold and updated as part of every ticket.

## Created at scaffold time

```
docs/vault/
├── CLAUDE.md                     conventions (below)
├── index.md                      catalog — read first; one [[wikilink]] per page
├── log.md                        append-only: ## [YYYY-MM-DD HH:MM] <op> | <title>
├── artifacts.md                  one row per published artifact (doc, page, video)
└── wiki/architecture/stack.md    what was generated: surfaces, stack, where things live
```

`docs/vault/` is in `.prettierignore` (Obsidian markdown, kept verbatim).

## Conventions (docs/vault/CLAUDE.md)

- Frontmatter on every page: `type`, `project`, `tags`, `created`, `updated`, `sources`.
- Wikilinks with full paths (`[[wiki/architecture/x|alias]]`).
- Every factual claim traceable; anything uncertain marked **unverified**; never invented
  numbers.
- Architecture pages track the real code; when code moves, fix the page.
- Pages stay short (a 300-line page gets split).
- **Dated decision notes** `wiki/decisions/YYYY-MM-DD-<topic>.md`: date, status with PR
  numbers, what was decided and why.

## Updated with every ticket

Before opening the PR, on the PR's own branch (a note written in the main checkout is never
committed): a feature page in `wiki/architecture/` or a dated decision note, its
`[[wikilink]]` in `index.md`, a line in `log.md`. Only when the change alters how the system
works, settles a decision, records a measurement, or documents a trap. Correct stale pages
you notice.

## Enforced by a Stop hook

`.claude/settings.json`:

```json
{ "hooks": { "Stop": [{ "hooks": [{ "type": "command", "command": "scripts/vault-check.sh" }] }] } }
```

`scripts/vault-check.sh` (exit 0 = let Claude stop, exit 2 + stderr = block with that message):

- exit 0 on `main` or a detached HEAD;
- exit 0 if this branch was already reminded (marker file
  `$(git rev-parse --git-dir)/vault-reminded-<branch>`);
- changed files = `git diff --name-only origin/main...HEAD` plus uncommitted
  (`git status --porcelain`);
- code = changes under `apps/` or `packages/`, excluding `*.spec.ts`, `*.int-spec.ts`,
  `*.test.ts` and `/generated/`;
- if code changed and nothing under `docs/vault/` did: create the marker, print "This branch
  changes code but not docs/vault/. Update the vault on this branch (feature page or dated
  note, index.md link, log.md line) or state in one line why no update is needed." and exit 2.

Test it (bats or a throwaway repo): allows on main; allows test-only changes; blocks once,
then allows; allows when the vault changed too.
