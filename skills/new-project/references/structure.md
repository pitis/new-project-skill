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
├── infra/compose.yaml  local Postgres 18 (+ Redis)
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
  `esbuild`, `@sentry/cli` → `true`; `@scarf/scarf` → `false` (install telemetry from
  swagger-ui).
- `turbo.json`: `build` outputs `dist/**`, `generated/**`, `.output/**`, `.nuxt/**`;
  `test` and `test:int` uncached and `dependsOn: ["^build"]`; `typecheck` depends on
  `^build` **and its own `build`** (Nuxt/Astro typecheck against generated types).
- Prettier: `singleQuote`, `trailingComma: 'all'`, `printWidth: 100`, `arrowParens:
  'always'`, `endOfLine: 'lf'`, plugin `prettier-plugin-organize-imports`. `.prettierignore`
  mirrors `.gitignore` plus `docs/vault/`, `envs/`, migrations, generated client, lockfile.
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
  host's env settings). Needs bash ≥ 4 (`mapfile`).

## Makefile

`up` / `down` / `logs` / `ps` / `psql` (Compose project `-p <slug>`), `migrate`,
`migrate-dev name=…`, `seed`, `env-*`, `worktree`, `worktree-setup`, `update`,
`test-int`, `check`, `wipe` (asks first).

**Ports:** derive the Postgres host port from the slug (e.g. `54000 + cksum(slug) % 1000`)
so several projects run side by side; containers named `<slug>_postgres`.

**`test-int`:** refuse unless `TEST_DB` ends in `_test`; start Postgres; create the test
database if missing; `migrate deploy`; empty every table except `_prisma_migrations`
(`infra/test/reset-test-db.sql`, a `TRUNCATE … CASCADE` over `pg_tables`, run with
`ON_ERROR_STOP=1`); run the API's `test:int` with test env vars.

## packages/db

- Schema is a **folder**: `prisma/schema/schema.prisma` (generator + datasource),
  `auth.prisma` (User, Session, Account, Verification), `billing.prisma` (Stripe
  `Subscription`, only with Stripe). Generator `prisma-client-js`, `output =
  "../../generated/client"`. Datasource has **no `url`**.
- `prisma.config.ts`: `schema: 'prisma/schema'`, migrations path, seed, `datasource.url`
  from `DATABASE_URL`, optional `shadowDatabaseUrl` from `SHADOW_DATABASE_URL`.
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
├── scripts/openapi-export.ts   NestFactory.create(AppModule, { preview: true }) → openapi.json
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
  `openapi:export`.
- `.swcrc` + `nest-cli.json` (SWC builder) exclude spec files. `vitest.config.ts`: `swc.vite({
  swcrc: false, inputSourceMap: false, … decorators })`, `hookTimeout: 30000`,
  `server.deps.external: [/\/generated\/client\//]` (see traps).
- Integration spec: `Test.createTestingModule({ imports: [AppModule], controllers:
  [TestOnlyController] })` → `createNestApplication({ bodyParser: false })` →
  `configureApp(app)` → `init()`; assertions listed in SKILL.md §3.4. Prove each guard by
  removing it once and watching its test fail.
- Dockerfile: `node:24-alpine`, corepack pnpm 12.10.1, `pnpm install --frozen-lockfile
  --filter @<slug>/api...`, build, `pnpm deploy`; targets `api` and one-shot `migrate`
  (`migrate deploy && seed`).

## packages/api-client

Orval: input `openapi.json` (written by the API's export), `mode: 'split'`, output
`src/generated/`, `client: 'vue-query'`, `httpClient: 'fetch'`, mutator `src/fetcher.ts`
exporting `apiFetch` that throws `ApiError` (non-2xx, with status and body) or
`NetworkError`, and returns the body. Cache keys are always the generated ones.

## Frontends

- **web** (Astro): marketing pages, `Analytics.astro` (PostHog behind the consent banner,
  UTM capture — references/utm.md), links to the app forward UTM params.
- **app** (Nuxt): plugins for vue-query, PostHog (client), Sentry (client, `dataCollection`
  off); `useAttribution()` composable; Better Auth Vue client; ESLint extends the root
  config's `luxonOnly` block.
- **extension** (WXT + Vue): scoped Sentry `BrowserClient` (never `Sentry.init` — it would
  capture the host page's errors), every outgoing link through `withUtm(…, { source:
  'extension', … })`, bearer auth. Never change manifest permissions casually.

## Deploy (Dokploy)

`docker-compose.dokploy.yml`: `postgres` (18, volume) → `migrate` (one-shot) → `api`
(`depends_on` migrate completed). Web and app on Vercel. Merged ≠ deployed unless the
project says so.
