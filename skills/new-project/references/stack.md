# Stack

| Area | Choice |
|---|---|
| Package manager | pnpm 12, the current release pinned exactly in `"packageManager"` (the Dockerfile uses the same), workspaces `apps/*`, `packages/*` |
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
| Web | Astro (landing/marketing), static output |
| App | Nuxt 4 (Vue 3) |
| Extension | WXT + Vue |
| Styling | Tailwind 4 |
| Errors | Sentry 11 (`@sentry/nestjs` in the API, preloaded) |
| Analytics | PostHog, EU host (`eu.i.posthog.com`) |
| Email | Resend (`resend` SDK) behind one `MailService`; Better Auth's verification and password-reset emails from day one |
| Jobs (optional) | BullMQ 6 via `@nestjs/bullmq`, `ioredis` as a direct API dependency (an optional peer since BullMQ 6), Redis 8; processors run in the API process |
| Tests | Vitest (+ `unplugin-swc` for Nest decorators), integration against Docker Postgres, one Playwright smoke test in `apps/app` (`make test-e2e`) |
| Quality | ESLint flat config, Prettier, husky hooks |
| Secrets | sops + age (`envs/*.env`), varlock schemas |
| Local | Docker Compose for Postgres (+ Redis with jobs); apps run on the host with `pnpm dev` |
| Deploy | API: Dockerfile with `api` and one-shot `migrate` images, for any container host. Web/app: static builds on Vercel or Cloudflare |

## Pins (do not take "latest")

Last checked 2026-10-19. At generation time, re-check each reason; drop a pin whose reason
no longer holds and say so in the report.

- **Prisma 7.10.x**, latest patch (7.10.1 on 2026-10-19) — npm's `latest` tag points at an
  8.0 release candidate.
- **stripe 22.x** — `@better-auth/stripe` 1.7 peers `stripe` ≤ 22; npm's `latest` is 23.
- **TypeScript 6** — TS 7 ships no compiler API; typescript-eslint (peers `<6.1`),
  `astro check` and the Nest CLI (`~6.0`) refuse it. Revisit at 7.1.
- **Postgres 18** (`postgres:18-alpine`) — `uuidv7()` does not exist before 18.

## Zod both ways (API)

- Global `ZodValidationPipe` (`APP_PIPE`): every body/query/param typed with a
  `createZodDto(schema)` class is parsed by it; failure → 400.
- Global `ZodSerializerInterceptor` (`APP_INTERCEPTOR`) and **`@ZodResponse({ status: 200,
  type: Dto })` on every endpoint** (the endpoint's real status): the return value is parsed
  by the schema, undeclared fields are stripped (an internal column can't leak), and the
  OpenAPI response type — hence Orval's generated types — comes from the same schema.
  Without `status` the response is `default` and Orval types every error as the success DTO.
- Swagger document passed through `cleanupOpenApiDoc` (nestjs-zod).

## Luxon only

Enforced by ESLint in the root config, not by review:

- `no-restricted-imports`: `date-fns`, `dayjs`, `moment`, `moment-timezone` → "Use luxon."
- `no-restricted-syntax`: `.getTime()` calls, `Date.now()`, and any variable named `*_MS`.

Use `DateTime`, `Duration`, `Interval`. Store `timestamptz`; send ISO strings
(`DateTime.utc().toISO()`). In tests, freeze time with `Settings.now = () => fixed.toMillis()`
and restore the saved original afterwards (assigning `() => DateTime.now().toMillis()`
recurses forever).
