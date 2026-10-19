# Structure

```
<slug>/
├── apps/
│   ├── api/            @<slug>/api        NestJS
│   ├── web/            @<slug>/web        Astro          (optional)
│   ├── app/            @<slug>/app        Nuxt/Vue       (optional)
│   └── extension/      @<slug>/extension  WXT + Vue      (optional)
├── packages/
│   ├── db/             @<slug>/db         Prisma schema, migrations, client, seed
│   ├── core/           @<slug>/core       shared types + helpers (attribution, withUtm)
│   └── api-client/     @<slug>/api-client Orval-generated vue-query hooks + apiFetch
├── docs/vault/         knowledge base (references/vault.md)
├── envs/               *.env (sops-encrypted) + schemas/<env>/.env.schema
├── infra/compose.yaml  local Postgres 18 (+ Redis with jobs)
├── infra/test/reset-test-db.sql
├── scripts/            env.sh, worktree.sh, update.sh, vault-check.sh, check-api-client.mjs
├── worktrees/          gitignored; one per ticket
├── .claude/settings.json   Stop hook → scripts/vault-check.sh
├── .husky/             pre-commit, pre-push
├── CLAUDE.md  Makefile  package.json  pnpm-workspace.yaml  turbo.json
└── tsconfig.base.json  eslint.config.mjs  prettier.config.mjs  .sops.yaml  .nvmrc
```

## Root

- `package.json` scripts: `build`, `dev`, `lint`, `typecheck`, `test`, `test:int` (turbo);
  `format` / `format:check` (Prettier); `quality:check` = `env:check && format:check &&
  turbo run lint typecheck test && api-client:check`; `api-client:generate` (build API →
  export OpenAPI → Orval); `api-client:check` (generate, then `scripts/check-api-client.mjs`);
  `env:pull` / `env:check` / `env:validate` (`scripts/env.sh`); `prepare: husky`.
- `pnpm-workspace.yaml`: packages + an **`allowBuilds:` map** (pnpm 12 fails the install on
  any unlisted build script): `@prisma/client`, `@prisma/engines`, `prisma`, `@swc/core`,
  `esbuild`, `@sentry/cli`, `vue-demi` → `true`; `@scarf/scarf` (install telemetry from
  swagger-ui), `core-js` (posthog-js funding banner), `msgpackr-extract` (BullMQ's optional
  native addon: it falls back to JS, and on Alpine node-gyp would fail the install) →
  `false`. A new dependency with a build script gets a line here, with the reason.
- `.gitignore`: `node_modules/`, build outputs (`dist/`, `.output/`, `.nuxt/`, `.astro/`,
  `.wxt/`, `.turbo/`), `packages/db/generated/` (never a bare `generated/`, see traps),
  `.env`, `.env.*`, `!.env.schema`, `worktrees/`, Playwright's `test-results/` and
  `playwright-report/`.
- ESLint: `defineConfig` from `eslint/config` (typescript-eslint's `tseslint.config()` is
  deprecated), typescript-eslint, `eslint-plugin-vue` `flat/essential` (`flat/recommended`'s
  formatting rules fight Prettier), `eslint-plugin-astro` for web, and the `luxonOnly` block
  (stack.md).
- Telemetry off everywhere: `ASTRO_TELEMETRY_DISABLED=1` in web's scripts, `telemetry:
  false` in `nuxt.config.ts`, `VARLOCK_TELEMETRY_DISABLED=1` in `scripts/env.sh`.
- `turbo.json`: `build` outputs `dist/**`, `generated/**`, `.output/**`, `.nuxt/**`;
  `test` and `test:int` uncached and `dependsOn: ["^build"]`; `typecheck` depends on
  `^build` **and its own `build`** (Nuxt/Astro typecheck against generated types).
- Prettier: `singleQuote`, `trailingComma: 'all'`, `printWidth: 100`, `arrowParens:
  'always'`, `endOfLine: 'lf'`, plugin `prettier-plugin-organize-imports`. `.prettierignore`
  mirrors `.gitignore` plus `docs/vault/`, `envs/`, migrations, `pnpm-lock.yaml`, and Orval's
  output: `packages/api-client/src/generated/` and `packages/api-client/openapi.json`
  (Prettier rewriting them makes `api-client:check` fail forever).
- `tsconfig.base.json`: `module`/`moduleResolution` `nodenext`, `strict`,
  `experimentalDecorators`, `emitDecoratorMetadata`, `isolatedModules`, target ES2023.
- `.husky/pre-commit`: `scripts/env.sh check --staged` then `pnpm quality:check`.
  `.husky/pre-push`: `pnpm quality:check`. There is no CI: these are the gate.
- `scripts/check-api-client.mjs`: fails if the regenerated client differs from the **index**
  or has untracked files (so a staged-but-uncommitted client passes on the first commit;
  `git diff --exit-code` would not).
- `scripts/env.sh`: `pull [--force]` (envs/local.env → root `.env`, validated first,
  refuses to overwrite hand edits unless `--force`), `check [--staged]` (fails on anything
  in `envs/` that is not a sops-encrypted `<name>.env` or a `schemas/<name>/.env.schema`),
  `validate` (each file against its schema with varlock; values reach it through
  `sops exec-env`, never the disk), `show <name>` (decrypted and quoted, to paste into a
  host's env settings). Needs bash ≥ 4 (`mapfile`). In the schemas, connection strings use
  varlock's `url` type with `allowedProtocols` (`postgresql`, `redis`): a bare `url` will
  reject them in varlock's next major.

## Makefile

`up` / `down` / `logs` / `ps` / `psql` (Compose project `-p <slug>`), `migrate`,
`migrate-dev name=…`, `seed`, `seed-dev`, `env-*`, `worktree`, `worktree-setup`, `update`,
`test-int`, `test-e2e` (with `apps/app`), `check`, `wipe` (asks first).

**Ports:** one offset from the slug (`cksum(slug) % 1000`) added to fixed bases, so several
projects run side by side: API `30000`, app `31000`, web `32000`, E2E API `33000`, E2E app
`34000`, Postgres `54000`, Redis `63000`. Containers named `<slug>_postgres`,
`<slug>_redis`.

**`test-int`:** refuse unless `TEST_DB` ends in `_test`; start Postgres (and Redis with
jobs); create the test database if missing; `migrate deploy`; empty every table except
`_prisma_migrations` (`infra/test/reset-test-db.sql`, a `TRUNCATE … CASCADE` over
`pg_tables`, run with `ON_ERROR_STOP=1`); with jobs, `FLUSHDB` Redis database 1 and point
`REDIS_URL` at it; turbo-build the API's dependencies, then run `pnpm --filter @<slug>/api
test:int` with test env vars (directly, not through turbo, whose strict env mode can strip
them).

**`test-e2e`:** the same test-database preparation as `test-int`, then build the API and the
app and run `pnpm --filter @<slug>/app test:e2e` (Playwright, Chromium only).

**`check`:** `pnpm quality:check && make test-int` (and `make test-e2e` with `apps/app`).
The flow's verify step: everything that must pass before a push.

**`seed-dev`:** runs the API's `seed:dev` script (below). Never part of `seed`, which the
production `migrate` target runs.

## packages/db

- Schema is a **folder**: `prisma/schema/schema.prisma` (generator + datasource),
  `auth.prisma` (User, Session, Account, Verification), `billing.prisma` (Stripe
  `Subscription`, only with Stripe: the plugin's fields plus `created_at`/`updated_at`,
  since the plugin writes `updatedAt`). Generator `prisma-client-js`, `output =
  "../../generated/client"`. Datasource has **no `url`**.
- `prisma.config.ts`: loads `../../.env` when it exists (`process.loadEnvFile`; Prisma 7
  doesn't read `.env`, and existing variables win, so `make test-int`'s override holds);
  `schema: 'prisma/schema'`, migrations path, seed, `datasource.url` from `DATABASE_URL`,
  optional `shadowDatabaseUrl` from `SHADOW_DATABASE_URL`.
- `package.json` `"files"`: `dist`, `generated` (and `prisma`), so `pnpm deploy` keeps the
  gitignored build output. Same for `packages/core` (`dist`).
- Every id `String @id @default(dbgenerated("uuidv7()")) @db.Uuid`; columns snake_case via
  `@map`, tables via `@@map`; times `@db.Timestamptz(3)`.
- `users` also has `stripe_customer_id` (nullable) and `attribution Json?`.
- `src/index.ts`: re-export the generated client; re-export the Prisma error classes **from
  the generated runtime** (`../generated/client/runtime/client.js`), never from
  `@prisma/client-runtime-utils` (different copy → `instanceof` silently false);
  `prismaClientOptions(url, { max, applicationName })` building `PrismaPg` with
  `maxLifetimeSeconds: 1800` and `connectionTimeoutMillis: 10000`; `createPrismaClient(url)`.
- Scripts: `build` = `prisma generate && tsc`, `migrate:dev`, `migrate:deploy`, `seed`.
- The first migration (`init`) is created at scaffold time, never shipped pre-made.

## apps/api

```
src/
├── instrument.ts        Sentry.init, preloaded; DATA_COLLECTION; tracesSampleRate()
├── main.ts              NestFactory.create(AppModule, { bodyParser: false }) → configureApp → listen
├── bootstrap.ts         configureApp(app): trust proxy, helmet, CORS, Better Auth, json, swagger
├── app.module.ts        Sentry, Config, Prisma, Throttler, Auth, Health + global pipe/interceptor/filters
├── swagger.ts           buildOpenApiDocument(app) (cleanupOpenApiDoc, 3.1.1), setupSwagger
├── config/configuration.ts   Zod-parsed env; refuses the dev auth secret in production
├── config/trust-proxy.ts
├── prisma/{prisma.service,prisma.module}.ts   global, extends PrismaClient
├── common/prisma-exception.filter.ts          P2002→409, P2025→404, P2003/P2000→400, else 500+Sentry
├── modules/auth/   auth.factory (betterAuth), auth.guard, current-user.decorator,
│                   me.controller (GET /v1/me, @ZodResponse), billing.plugins (Stripe or [])
├── modules/health/ GET /health → { status: 'ok', time } (@SkipThrottle, @ZodResponse)
├── modules/mail/   mail.service (Resend or outbox), templates.ts, mail.processor (jobs only)
├── modules/jobs/   jobs.module: BullModule.forRootAsync from REDIS_URL (jobs only)
├── scripts/openapi-export.ts   NestFactory.create(AppModule, { preview: true }) → openapi.json
├── scripts/seed-dev.ts         the dev account (below)
└── integration/app.int-spec.ts
```

- **`bootstrap.ts` is shared by `main.ts` and the integration tests** so the tests run the
  real middleware order. Order matters: helmet → CORS (`origin: frontendUrls, credentials:
  true, exposedHeaders: ['set-auth-token']`) → `http.all('/api/auth/*splat',
  toNodeHandler(auth))` → `express.json()`.
- Better Auth: `prismaAdapter(prisma, { provider: 'postgresql' })`, `trustedOrigins` =
  frontend URLs + API origin, `emailAndPassword`, `advanced.database.generateId: false`
  (Postgres makes uuidv7), `useSecureCookies` on https, `rateLimit` off when `NODE_ENV=test`,
  plugins `bearer({ requireSignature: true })` + billing plugins.
  `emailVerification: { sendOnSignUp: true, autoSignInAfterVerification: true,
  sendVerificationEmail }` and `emailAndPassword.sendResetPassword`, both through
  `MailService`, not awaited (traps). `requireEmailVerification` stays off: blocking sign-in
  until verified is the owner's call.
- **Mail.** `MailService.send({ to, template, data })`. With `RESEND_API_KEY` it sends
  through Resend from `MAIL_FROM` and throws on the SDK's `error`; without it (local, test)
  it logs the subject and link and appends the message to `outbox`, an array the integration
  tests read. The config refuses production without `RESEND_API_KEY` and `MAIL_FROM`.
  `templates.ts`: `verifyEmail`, `resetPassword` → `{ subject, html, text }`, plain template
  strings with interpolated values escaped. Links are built from Better Auth's `token`, never
  from its `url`: verification is the API's `/api/auth/verify-email?token=…&callbackURL=…`
  with the callback set to the first `FRONTEND_URL` through `withUtm`; reset is
  `<first FRONTEND_URL>/reset-password?token=…` through `withUtm` (utm.md, "Our own links").
- **Jobs** (only when chosen). `JobsModule` registers BullMQ from `REDIS_URL` parsed into
  connection options (`host`, `port`, `password`, `db`) and a `<slug>:` prefix. One real
  queue, `mail`: `MailService.send` enqueues with `jobId: randomUUID()` (`attempts: 5`,
  exponential backoff, delay from a Luxon `Duration`), `MailProcessor` sends, passing the job
  id as Resend's idempotency key so a retry never sends twice. Never BullMQ's default ids: a
  per-queue counter that restarts after Redis loses its data, and Resend silently drops a
  reused key within 24 hours. Processors run in the API process (`app.enableShutdownHooks()`
  so they drain on stop). shortcut: one process; move processors into a `worker.ts`
  entrypoint when a job is CPU-heavy or needs its own scaling.
- **`scripts/seed-dev.ts`** (`seed:dev`): refuses when `NODE_ENV` is `production`;
  creates or resets `dev@<slug>.localhost` / `<slug>-dev` through `auth.api.signUpEmail` (so
  the password hash is Better Auth's own), deleting the old account first, then marks the
  email verified.
- `AuthGuard` resolves the session from headers (cookie or bearer), sets `req.user` (id,
  email, name, emailVerified) and `Sentry.setUser({ id })` only. Every user-scoped query uses
  `req.user.id`, never an id from the body.
- Global providers: `ThrottlerGuard`, `ZodValidationPipe`, `ZodSerializerInterceptor`,
  `SentryGlobalFilter` then `PrismaClientExceptionFilter` (filters resolve
  last-registered-first). Throttler TTL via `Duration.fromObject({ minutes: 1 }).toMillis()`.
- Scripts: `build` (nest build), `dev` (`nest start --watch --exec "node
  --env-file-if-exists=../../.env --import ./dist/instrument.js dist/main.js"`), `start`
  (`node --import ./dist/instrument.js dist/main.js`), `test` (vitest, excluding
  `*.int-spec.ts`), `test:int` (`vitest run src/integration --no-file-parallelism`),
  `openapi:export`, `seed:dev` (`node --env-file-if-exists=../../.env dist/scripts/seed-dev.js`).
- `.swcrc` + `nest-cli.json` (SWC builder) exclude spec files. `vitest.config.ts`: `swc.vite({
  swcrc: false, inputSourceMap: false, … decorators })`, `hookTimeout: 30000`,
  `server.deps.external: [/\/generated\/client\//]` (see traps).
- Integration spec: `Test.createTestingModule({ imports: [AppModule], controllers:
  [TestOnlyController] })` → `createNestApplication({ bodyParser: false })` →
  `configureApp(app)` → `init()`; assertions listed in SKILL.md §3.4. Prove each guard by
  removing it once and watching its test fail.
- Dockerfile: `node:24-alpine`, corepack with the `packageManager` pnpm, `pnpm install --frozen-lockfile
  --filter @<slug>/api...`, build, `pnpm deploy`; targets `api` and one-shot `migrate`
  (`migrate deploy && seed`; it deploys `packages/db` without `--prod`, since the Prisma CLI
  is a devDependency).

## packages/api-client

Orval: input `openapi.json` (written by the API's export), `mode: 'split'`, output
`src/generated/`, `client: 'vue-query'`, `httpClient: 'fetch'`, mutator `src/fetcher.ts`
exporting `apiFetch` that throws `ApiError` (non-2xx, with status and body) or
`NetworkError`, and returns the body; `override.fetch.includeHttpResponseReturnType: false`
so the generated functions return the body too. Split mode emits extensionless imports, so
this package's tsconfig is `module: esnext`, `moduleResolution: bundler` (only bundlers
consume it). Cache keys are always the generated ones.

## Frontends

- **web** (Astro): marketing pages, `Analytics.astro` (PostHog behind the consent banner,
  UTM capture — references/utm.md), links to the app forward UTM params.
- **app** (Nuxt): `ssr: false` (it sits behind a login; SEO is web's job), API URL from
  `runtimeConfig.public.apiUrl`. Plugins for vue-query, PostHog (client), Sentry (client,
  `dataCollection` off); `useAttribution()` composable; Better Auth Vue client; ESLint
  extends the root config's `luxonOnly` block. The same consent banner as web, shown when
  the consent cookie is unset (people often land on the app directly). Pages: `/sign-up`, `/login`,
  `/forgot-password`, `/reset-password` (the email link's target), and `/` showing the
  signed-in user's email from `GET /v1/me` (redirects to `/login` without a session).
- **app E2E**: `apps/app/e2e/smoke.spec.ts` + `playwright.config.ts`, script `test:e2e`.
  `webServer` starts the built API (test env, `_test` database, E2E port) and the built app
  (`node .output/server/index.mjs`, E2E port, `NUXT_PUBLIC_API_URL` at the E2E API),
  `reuseExistingServer: false`. The smoke test: sign up a fresh address → `/` shows it → sign
  out → log in → `/` shows it again. Every ticket that changes the app adds to this suite.
- **extension** (WXT + Vue): scoped Sentry `BrowserClient` (never `Sentry.init` — it would
  capture the host page's errors), every outgoing link through `withUtm(…, { source:
  'extension', … })`, bearer auth. Never change manifest permissions casually.

## Deploy

No API host is chosen: the API ships as the Dockerfile's two images, for any container host.
Each release runs the one-shot `migrate` image before the new `api` starts. Production
Postgres must be 18; with jobs, production Redis runs with `--appendonly yes
--maxmemory-policy noeviction` (local Redis in `infra/compose.yaml` uses the same flags).
Web and app on Vercel. Merged ≠ deployed unless the project says so.
