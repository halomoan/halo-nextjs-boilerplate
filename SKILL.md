---
name: nextjs-boilerplate
description: >
  Scaffold a complete, opinionated Next.js 15 boilerplate from scratch with
  TypeScript strict mode, DrizzleORM + PostgreSQL, better-auth, next-intl
  (i18n), React Hook Form + Zod, Tailwind v4, Vitest, Playwright, and
  Storybook — all pre-configured and verified. Use this skill whenever the
  user wants to create a new web app, start a new project, scaffold or
  bootstrap a Next.js application, or mentions setting up a new codebase with
  any subset of: TypeScript, Drizzle, auth, i18n, forms, testing, Storybook.
  Also trigger on "new project", "create app", "scaffold", "boilerplate",
  "initialize project", "set up a web app", even without specific tech names.
---

# Next.js Boilerplate

Walk the user from nothing to a running, fully configured Next.js project
with a working example form, unit tests, and a Storybook story. Explain each
step briefly as you go — the user should understand what is being built.

All shell commands are POSIX-compatible. On Windows run them through the Bash
tool (Git Bash / WSL). Do not translate to PowerShell unless a command fails.

This skill is linear. Work through steps in order — later steps depend on
earlier ones (e.g. migrations need a configured `DATABASE_URL`).

---

## 1. Pre-flight

Detect what is installed before asking the user to choose anything.

Run in parallel:

```bash
node --version
pnpm --version
git --version
psql --version
```

Interpret results:

- **Node.js** must be >= 18. If not, stop and tell the user to install Node 18+.
- **pnpm** must be present. If not: `npm install -g pnpm`, then re-check.
- **Git** must be present.
- **psql** — note if available; useful for migration verification but not required.

Report findings in one sentence, e.g. "Found Node 22, pnpm 10, Git 2.47. psql not found (optional)."

---

## 2. Gather project info

Ask the user via `AskUserQuestion` for three things:

1. **Project name** — used as the directory name and `package.json` `name`.
2. **Database URL** — PostgreSQL connection string (can be a placeholder like
   `postgresql://postgres:password@localhost:5432/mydb`).
3. **Secondary locale code** — the locale to add alongside English (e.g. `id`,
   `fr`, `de`). Default: `id`.

Store these as `<name>`, `<db_url>`, `<locale>`.

---

## 3. Confirm dependency list

Before installing anything, show the user this full dependency list and ask:
"Does this list look right? Let me know if you want to add or remove anything."

**create-next-app scaffold**
```
pnpm create next-app@latest <name> \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*" \
  --no-turbopack \
  --yes
```
> `--no-turbopack` prevents better-auth SSR errors during development. You can
> re-enable it later once better-auth ships a Turbopack-compatible release.
> `--yes` skips interactive prompts so the command runs non-interactively.

**Production dependencies**
```
pnpm add \
  drizzle-orm pg \
  better-auth \
  next-intl \
  react-hook-form @hookform/resolvers zod \
  @logtape/logtape
```

**Dev dependencies**
```
pnpm add -D \
  drizzle-kit @types/pg \
  prettier prettier-plugin-tailwindcss \
  vitest @vitejs/plugin-react jsdom \
  @testing-library/react @testing-library/jest-dom @testing-library/user-event \
  @playwright/test \
  "@storybook/nextjs@^8" "@storybook/addon-essentials@^8" "@storybook/react@^8" "storybook@^8"
```
> Storybook must be pinned to v8 — v10 ships with peer dep conflicts that break
> installs. `@storybook/react` must be listed explicitly for type imports in
> stories and test utilities.
>
> `@storybook/nextjs@8` declares peer support only up to Next.js 15. pnpm will
> show a peer dep warning with Next.js 16 but will install it anyway and it works
> correctly. Do NOT switch to `@storybook/react-vite` to avoid this warning — it
> introduces a Vite version conflict with Vitest 4.

Only proceed once the user confirms.

---

## 4. Scaffold with create-next-app

```bash
pnpm create next-app@latest <name> \
  --typescript --tailwind --eslint --app --src-dir \
  --import-alias "@/*" --no-turbopack --yes
```

Run this **synchronously** (not in background) with a timeout of at least
5 minutes — it downloads and installs Next.js and its deps. Wait for it to
finish before proceeding.

After it finishes, verify `node_modules` was populated:

```bash
test -d <name>/node_modules && ls <name>/node_modules | head -5
```

If `node_modules` is missing, run `pnpm install` inside `<name>` before continuing.

`cd` into `<name>` for all subsequent steps.

---

## 5. Install additional dependencies

```bash
pnpm add drizzle-orm pg better-auth next-intl \
  react-hook-form @hookform/resolvers zod @logtape/logtape

pnpm add -D drizzle-kit @types/pg \
  prettier prettier-plugin-tailwindcss \
  vitest @vitejs/plugin-react jsdom \
  @testing-library/react @testing-library/jest-dom @testing-library/user-event \
  @playwright/test \
  @storybook/nextjs @storybook/addon-essentials storybook
```

---

## 6. Create configuration files

**Now read `references/configs.md`** — it contains the exact file contents for
every config. Create each file listed there, substituting `<locale>` and
`<db_url>` as appropriate. The files to create are:

**Design System Note:** If the project has a `DESIGN.md` file (document defining
color tokens, typography, spacing, component patterns, etc.), reference it when
setting up Tailwind configuration and creating CSS variables. The design tokens
should align with any existing brand system defined in DESIGN.md.

- `.env` (DATABASE_URL + BETTER_AUTH_SECRET)
- `drizzle.config.ts`
- `next.config.ts` (next-intl plugin + serverExternalPackages)
- `src/middleware.ts` (next-intl routing)
- `src/i18n/request.ts` (next-intl request config)
- `messages/en.json` (empty shell for now)
- `messages/<locale>.json` (empty shell for now)
- `vitest.config.ts` + `vitest.setup.ts`
- `playwright.config.ts`
- `.storybook/main.ts` + `.storybook/preview.ts`
- `.prettierrc`
- `src/lib/logger.ts`
- `tsconfig.json` — merge the Storybook `exclude` entries (see configs.md)

Generate `BETTER_AUTH_SECRET`:
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

Write the output into `.env` as `BETTER_AUTH_SECRET=<value>`.

Also update `package.json` `scripts` to add:
```json
"test": "vitest",
"test:e2e": "playwright test",
"storybook": "storybook dev -p 6006",
"build-storybook": "storybook build",
"db:generate": "drizzle-kit generate",
"db:migrate": "drizzle-kit migrate",
"db:studio": "drizzle-kit studio"
```

---

## 7. Create the database schema and run migrations

Create `db/schema.ts` and `db/index.ts` — see `references/configs.md` for the
exact contents.

Then generate and run the first migration:

```bash
pnpm db:generate
pnpm db:migrate
```

If migration fails, diagnose from the error message:

- **"database does not exist"** — create the database first:
  ```bash
  # If psql is available:
  psql -c "CREATE DATABASE <dbname>;" postgres
  # Or via createdb:
  createdb <dbname>
  ```
  Extract `<dbname>` from the `DATABASE_URL` in `.env` (the last path segment).
  Then re-run `pnpm db:migrate`.

- **"connection refused"** — Postgres is not running. Ask the user to start it
  (e.g. `pg_ctl start`, start the service, or `docker start <container>`).

- **"password authentication failed"** or **"role does not exist"** — wrong
  credentials in `DATABASE_URL`. Ask the user to correct `.env` and retry.

Surface the exact error text; do not guess or silently retry with different flags.

---

## 8. Set up better-auth

Create `src/lib/auth.ts` and `src/app/api/auth/[...all]/route.ts` — see
`references/configs.md`.

**SSR note:** better-auth uses React hooks internally. To prevent
"Cannot read properties of null (reading 'useRef')" errors in SSR:

- Add `export const dynamic = "force-dynamic"` to any page that imports auth
  client-side
- The `serverExternalPackages` setting in `next.config.ts` (from Step 6)
  handles the server-side case

---

## 9. Build the example form

**Now read `references/example-form.md`** — it contains the full implementation
of the sign-up form, its unit test, and its Storybook story.

**Design System Reference:** If the project has a `DESIGN.md` file, consult its
sections on:
- **Components** (section 7.2 "Inputs & Forms") for form field styling and layout rules
- **Typography** (section 3) for label and input text sizes
- **Color** (section 2) for input states (focus, error, disabled) and validation colors
- **Spacing & Layout** (section 4) for form field gaps and padding
- **Accessibility** (section 11) for focus states and form error announcements

Create these files:

- `src/test-utils/intl.tsx` — shared next-intl wrapper for tests and stories
- `src/lib/validations/auth.ts` — Zod schema
- `src/components/ui/sign-up-form/SignUpForm.tsx` — form component
- `src/components/ui/sign-up-form/SignUpForm.test.tsx` — Vitest + RTL tests
- `src/components/ui/sign-up-form/SignUpForm.stories.tsx` — Storybook story
- `messages/en.json` — with SignUpForm translation keys
- `messages/<locale>.json` — same keys, placeholder translations

---

## 10. Verify

Run these checks in order:

```bash
pnpm lint           # ESLint — fix any errors before continuing
pnpm test           # Vitest — all tests should pass
pnpm build          # Production build — must succeed
```

If any step fails, diagnose and fix before reporting success. Do not paper over
type errors or test failures.

After a clean build, start the dev server and confirm it responds:

```bash
pnpm dev &
sleep 5
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000
```

---

## 11. Install project-level skills

Copy frontend-design and security-guidance skills to the project's `.claude/skills/`
directory so they're available whenever the user works in this project.

```bash
mkdir -p .claude/skills
```

Then locate and copy each skill from the user's global skill directory:

- **frontend-design** → `~/.claude/skills/frontend-design/` → `<project>/.claude/skills/frontend-design/`
- **security-guidance** → `~/.claude/skills/security-guidance/` → `<project>/.claude/skills/security-guidance/`

Use `cp -r` to copy the entire skill directory (SKILL.md + bundled resources).

If either skill is not found in the user's global skills directory, note it and
continue — the project will still work, and the user can manually copy these
skills later if needed.

---

## 12. Report to user

Tell the user:

1. The app is running at http://localhost:3000
2. Where `.env` is (absolute path) and which vars are set vs blank
3. The project structure (run `find src db messages -type f | sort`)
4. Project-level skills installed: frontend-design and security-guidance are now
   available in `.claude/skills/` — they'll load automatically when working in
   this project
5. **Design System:** If the project includes a `DESIGN.md` file, emphasize that
   it's the reference for all component patterns, colors, typography, spacing,
   and accessibility rules. All new components and pages should follow the
   patterns defined there.
6. Next steps: "Run `pnpm db:studio` to browse data, `pnpm storybook` to view
   the component explorer, `pnpm test:e2e` to run Playwright, and refer to
   `DESIGN.md` when building new components or pages."

Keep it short — they want to start building.
