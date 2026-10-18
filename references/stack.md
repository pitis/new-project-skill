# Stack

| Area | Choice |
|---|---|
| Package manager | **pnpm 12.10.1** (`"packageManager": "pnpm@12.10.1"`), workspaces `apps/*`, `packages/*` |
| Monorepo | Turborepo: `build`, `dev`, `lint`, `typecheck`, `test`, `test:int` |
| Runtime | Node 24 (`engines` + `.nvmrc`), ESM everywhere (`"type": "module"`, `.js` import specifiers) |
| Language | TypeScript `~6.0` |
| API | NestJS 12, Express 5, SWC build (`nest build`), helmet, `@nestjs/throttler`, Swagger |
| Validation | **nestjs-zod, both directions** (below). Zod 4. No class-validator |
| Database | Postgres 18, Prisma **7.10** + `@prisma/adapter-pg`, ids `@default(dbgenerated("uuidv7()")) @db.Uuid` |
| Auth | Better Auth (Prisma adapter, `bearer()` plugin) |
| Payments | Stripe via `@better-auth/stripe`, one webhook at `/api/auth/stripe/webhook` |
| API client | Orval → `packages/api-client`, `client: 'vue-query'`, `httpClient: 'fetch'`, one `apiFetch` mutator |
| Dates | **Luxon only**, everywhere, tests included |
| Web | Astro (landing/marketing), on Vercel |
| App | Nuxt 4 (Vue 3) |
| Extension | WXT + Vue |
| Styling | Tailwind 4 |
| Errors | Sentry 11 (`@sentry/nestjs` in the API, preloaded) |
| Analytics | PostHog, EU host (`eu.i.posthog.com`) |
| Email | Resend |
| Tests | Vitest (+ `unplugin-swc` for Nest decorators), integration against Docker Postgres, Playwright E2E |
| Quality | ESLint flat config, Prettier, husky hooks |
| Secrets | sops + age (`envs/*.env`), varlock schemas |
| Local | Docker Compose for Postgres (+ Redis); apps run on the host with `pnpm dev` |
| Deploy | API: Dockerfile + Dokploy compose (postgres → migrate → api). Web/app: Vercel |

## Pins (do not take "latest")

- **Prisma 7.10.0** — npm's `latest` tag points at an 8.0 release candidate.
- **TypeScript 6** — TS 7 ships no compiler API; typescript-eslint, `astro check` and the
  Nest CLI refuse it. Revisit at 7.1.
- **Postgres 18** (`postgres:18-alpine`) — `uuidv7()` does not exist before 18.

## Zod both ways (API)

- Global `ZodValidationPipe` (`APP_PIPE`): every body/query/param typed with a
  `createZodDto(schema)` class is parsed by it; failure → 400.
- Global `ZodSerializerInterceptor` (`APP_INTERCEPTOR`) and **`@ZodResponse({ type: Dto })`
  on every endpoint**: the return value is parsed by the schema, undeclared fields are
  stripped (an internal column can't leak), and the OpenAPI response type — hence Orval's
  generated types — comes from the same schema.
- Swagger document passed through `cleanupOpenApiDoc` (nestjs-zod).

## Luxon only

Enforced by ESLint in the root config, not by review:

- `no-restricted-imports`: `date-fns`, `dayjs`, `moment`, `moment-timezone` → "Use luxon."
- `no-restricted-syntax`: `.getTime()` calls, `Date.now()`, and any variable named `*_MS`.

Use `DateTime`, `Duration`, `Interval`. Store `timestamptz`; send ISO strings
(`DateTime.utc().toISO()`). In tests, freeze time with `Settings.now = () => fixed.toMillis()`
and restore the saved original afterwards (assigning `() => DateTime.now().toMillis()`
recurses forever).
