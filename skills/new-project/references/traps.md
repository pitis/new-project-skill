# Traps

Each one already cost a debugging session. Copy the ones that apply into the new project's
CLAUDE.md.

## Tooling

- **pnpm 12 fails the install on any unapproved build script** (`ERR_PNPM_IGNORED_BUILDS`).
  `onlyBuiltDependencies` is no longer read: use the `allowBuilds:` map in
  `pnpm-workspace.yaml`. Deny `@scarf/scarf` (telemetry) explicitly.
- **Prisma's npm `latest` is an 8.0 RC.** Pin 7.10.0 for `prisma`, `@prisma/client`,
  `@prisma/adapter-pg`, `@prisma/client-runtime-utils`.
- **Vitest + the generated Prisma client:** the client is a workspace file, so Vite transforms
  it and prints "Failed to load source map … client.js.map" on every run. Set
  `test.server.deps.external: [/\/generated\/client\//]`. With `unplugin-swc`, also
  `swcrc: false` (`.swcrc` excludes `*.spec.ts`) and `inputSourceMap: false`.
- **Generate the Prisma client once**, in `db#build`. Generating it in another package's test
  or typecheck races the suites that import it (half-written client errors).
- **Stale Prisma client / build after a branch switch:** rebuild `packages/db` and the API
  before trusting any test result.
- **Zod 4's `z.email()` requires a TLD of 2+ letters**: `a@b.c` is invalid. Use `a@b.co` in
  tests.
- **TypeScript 7 has no compiler API**: stay on 6.

## API

- **`bodyParser: false` is load-bearing.** Better Auth (and the Stripe webhook signature) needs
  the raw body: mount `/api/auth/*splat` before `express.json()`.
- **Sentry must be preloaded** (`node --import ./dist/instrument.js`). An init in `main.ts`
  catches errors but never traces (ESM evaluates imports first), and Prettier's
  organize-imports would sort a top-of-file import out of position anyway.
- **Sentry 11 collects user info, cookies, headers, bodies and query strings by default.**
  Every `init` (API, app, extension) sets `dataCollection: { userInfo: false, cookies: false,
  httpHeaders: false, httpBodies: [], urlQueryParams: false }`. `Sentry.setUser` gets the id
  only.
- **Sample-rate parsing:** `Number(process.env.X ?? 0.1)` turns `''` (Compose's rendering of
  an unset `${X:-}`) into 0 and silently disables tracing. Treat empty/NaN/out-of-range as the
  default.
- **Prisma error classes** must come from the generated runtime, not
  `@prisma/client-runtime-utils` (`instanceof` silently false).
- **`set-auth-token` must be in CORS `exposedHeaders`** for bearer clients.
- **Swagger cookie auth:** pass the scheme name as `addCookieAuth`'s third argument or
  `@ApiCookieAuth` references a missing scheme.
- **Better Auth sign-up with a NOT NULL column it doesn't set 500s every sign-up**: give new
  user columns a database default.
- **`FRONTEND_URL`'s first entry is the address put in every link the API sends** (emails,
  Stripe return URLs). Unset falls back to localhost: every link points at localhost.

## Database

- **`uuidv7()` needs Postgres 18.**
- **Never edit or rename an applied migration**: Prisma checksums them and `migrate deploy`
  refuses.
- **The dev database is shared by all worktrees** and runs ahead of any one branch; for
  `migrate dev` drift, diff against a shadow database instead of resetting dev.
- **`make test-int` empties the test database**: the name must end in `_test`, and two sessions
  running it at once break each other.

## Process

- **Never `--no-verify`, never force-push, never rebase a pushed branch** (merge `main` in).
- **No AI attribution** in commits, PRs or comments.
- **Screenshots and evidence stay out of the repo.**
- **A test proves a fix only if it failed first.**
- **Merged is not deployed** unless the project deploys on merge; never call something live
  because it merged.
- **Rotating `BETTER_AUTH_SECRET`** signs everyone out and breaks any HMAC link signed with it.
