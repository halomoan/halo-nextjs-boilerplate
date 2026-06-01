# Configuration File Templates

Substitute `<locale>` and `<db_url>` with values gathered in Step 2.
Substitute `<secret>` with the generated BETTER_AUTH_SECRET.

---

## .env

```
DATABASE_URL=<db_url>
BETTER_AUTH_SECRET=<secret>
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## drizzle.config.ts

```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./db/schema.ts",
  out: "./db/migrations",
  dialect: "postgresql",
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
```

---

## next.config.ts

```ts
import createNextIntlPlugin from "next-intl/plugin";
import type { NextConfig } from "next";

const withNextIntl = createNextIntlPlugin("./src/i18n/request.ts");

const nextConfig: NextConfig = {
  serverExternalPackages: [
    "better-auth",
    "@better-auth/kysely-adapter",
    "kysely",
    "pg",
    "drizzle-orm",
  ],
  turbopack: {
    ignoreIssue: [
      {
        path: /@better-auth\/kysely-adapter/,
        title: /Export .* doesn't exist in target module/,
      },
    ],
  },
};

export default withNextIntl(nextConfig);
```

> `serverExternalPackages` prevents better-auth from being bundled by Next.js's
> server compiler, which causes "useRef" errors in SSR. Including `pg` and
> `drizzle-orm` avoids native-binding issues during build.
>
> `turbopack.ignoreIssue` suppresses a build-time error from better-auth's
> transitive dependency `@better-auth/kysely-adapter` which references a kysely
> export path that moved in newer kysely versions. This is a third-party bug;
> ignoring it is safe because the drizzle adapter does not use kysely.

---

## src/middleware.ts

```ts
import createMiddleware from "next-intl/middleware";
import { routing } from "./i18n/routing";

export default createMiddleware(routing);

export const config = {
  matcher: ["/((?!api|_next|_vercel|.*\\..*).*)"],
};
```

---

## src/i18n/routing.ts

```ts
import { defineRouting } from "next-intl/routing";

export const routing = defineRouting({
  locales: ["en", "<locale>"],
  defaultLocale: "en",
});
```

---

## src/i18n/request.ts

```ts
import { getRequestConfig } from "next-intl/server";
import { routing } from "./routing";

export default getRequestConfig(async ({ requestLocale }) => {
  const locale = (await requestLocale) ?? routing.defaultLocale;
  return {
    locale,
    messages: (await import(`../../messages/${locale}.json`)).default,
  };
});
```

---

## messages/en.json (initial shell)

```json
{}
```

## messages/\<locale\>.json (initial shell)

```json
{}
```

---

## vitest.config.ts

```ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    setupFiles: ["./vitest.setup.ts"],
    globals: true,
    exclude: ["**/node_modules/**", "**/e2e/**"],
  },
  resolve: {
    alias: { "@": path.resolve(__dirname, "./src") },
  },
});
```

---

## vitest.setup.ts

```ts
import "@testing-library/jest-dom";
```

---

## playwright.config.ts

```ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./e2e",
  use: {
    baseURL: "http://localhost:3000",
    trace: "on-first-retry",
  },
  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
  ],
  webServer: {
    command: "pnpm dev",
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI,
  },
});
```

---

## .storybook/main.ts

```ts
import type { StorybookConfig } from "@storybook/nextjs";

const config: StorybookConfig = {
  stories: ["../src/**/*.stories.@(js|jsx|ts|tsx)"],
  addons: ["@storybook/addon-essentials"],
  framework: {
    name: "@storybook/nextjs",
    options: {},
  },
  docs: {
    autodocs: "tag",
  },
};

export default config;
```

---

## .storybook/preview.ts

```ts
import type { Preview } from "@storybook/react";
import "../src/app/globals.css";

const preview: Preview = {
  parameters: {
    controls: { matchers: { color: /(background|color)$/i, date: /Date$/i } },
  },
};

export default preview;
```

---

## .prettierrc

```json
{
  "plugins": ["prettier-plugin-tailwindcss"],
  "semi": true,
  "singleQuote": false,
  "trailingComma": "es5",
  "printWidth": 100
}
```

---

## src/lib/logger.ts

```ts
import {
  configure,
  getLogger,
  getConsoleSink,
  type LogLevel,
} from "@logtape/logtape";

await configure({
  sinks: {
    console: getConsoleSink(),
  },
  loggers: [
    {
      category: ["app"],
      sinks: ["console"],
      lowestLevel: (process.env.LOG_LEVEL as LogLevel) ?? "debug",
    },
  ],
  reset: true,
});

export const logger = getLogger(["app"]);
```

> `reset: true` allows the logger to be re-configured in tests without throwing.

---

## db/index.ts

```ts
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./schema";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

export const db = drizzle(pool, { schema });
```

---

## db/schema.ts

```ts
import { pgTable, text, timestamp, uuid } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: text("email").notNull().unique(),
  name: text("name").notNull(),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
});
```

---

## src/lib/auth.ts

```ts
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { db } from "../../db";
import * as schema from "../../db/schema";

export const auth = betterAuth({
  database: drizzleAdapter(db, {
    provider: "pg",
    schema: { users: schema.users },
  }),
  emailAndPassword: { enabled: true },
  trustedOrigins: [process.env.NEXT_PUBLIC_APP_URL ?? "http://localhost:3000"],
});
```

---

## src/app/api/auth/[...all]/route.ts

```ts
import { auth } from "@/lib/auth";
import { toNextJsHandler } from "better-auth/next-js";

export const { GET, POST } = toNextJsHandler(auth);
```

---

## src/test-utils/intl.tsx

A single file shared by both unit tests and Storybook decorators so
`NextIntlClientProvider` setup is never duplicated.

```tsx
import { NextIntlClientProvider } from "next-intl";
import { render, type RenderOptions } from "@testing-library/react";
import type { ReactNode } from "react";
import type { Decorator } from "@storybook/react";

interface IntlWrapperProps {
  locale?: string;
  messages: Record<string, unknown>;
  children: ReactNode;
}

function IntlWrapper({ locale = "en", messages, children }: IntlWrapperProps) {
  return (
    <NextIntlClientProvider locale={locale} messages={messages}>
      {children}
    </NextIntlClientProvider>
  );
}

/** Use in RTL tests: renderWithIntl(<MyForm />, enMessages) */
export function renderWithIntl(
  ui: React.ReactElement,
  messages: Record<string, unknown>,
  locale = "en",
  options?: RenderOptions
) {
  return render(
    <IntlWrapper locale={locale} messages={messages}>
      {ui}
    </IntlWrapper>,
    options
  );
}

/** Use in Storybook: decorators: [withIntl(messages)] */
export function withIntl(
  messages: Record<string, unknown>,
  locale = "en"
): Decorator {
  // eslint-disable-next-line react/display-name
  const decorator: Decorator = (Story) => (
    <IntlWrapper locale={locale} messages={messages}>
      <Story />
    </IntlWrapper>
  );
  return decorator;
}
```

---

## tsconfig.json — add Storybook to exclude

`create-next-app` generates a `tsconfig.json`. Merge this into the `exclude`
array so `next build`'s TypeScript checker doesn't try to resolve `@storybook/*`
types, which causes "Cannot find module" errors:

```json
{
  "exclude": ["node_modules", "**/*.stories.tsx", "**/*.stories.ts"]
}
```

Open the generated `tsconfig.json` and add these two glob patterns to the
existing `exclude` array (or create the array if absent).

---

## src/lib/auth-client.ts

```ts
import { createAuthClient } from "better-auth/react";

export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_APP_URL ?? "http://localhost:3000",
});
```

> Import this only inside client components (or via `next/dynamic` with
> `ssr: false`). Importing it at module level in a server component causes
> "Cannot read properties of null (reading 'useRef')" during build.
