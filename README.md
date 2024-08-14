1. Install next-intl

https://next-intl-docs.vercel.app/docs/getting-started/app-router

2. Create the `locales.ts` file:

```ts
export const locales = ["en", "es"] as const;
export type Locale = (typeof locales)[number];
```

3. Create the `i18n.ts` file under the `src` directory:

```ts
"server-only";

import { notFound } from "next/navigation";
import { getRequestConfig } from "next-intl/server";
import { type AbstractIntlMessages } from "next-intl";
import { locales, type Locale } from "@/lib/locales";

const messageImports = {
  en: () => import("./messages/en.json"),
  es: () => import("./messages/es.json"),
} as const satisfies Record<Locale, () => Promise<{ default: AbstractIntlMessages }>>;

export function isValidLocale(locale: unknown): locale is Locale {
  return locales.some((l) => l === locale);
}

export default getRequestConfig(async (params) => {
  const baseLocale = new Intl.Locale(params.locale).baseName;
  if (!isValidLocale(baseLocale)) notFound();

  const messages = (await messageImports[baseLocale]()).default;
  return {
    messages,
  };
});
```

4. Create the `middleware.ts` file:

```ts
import { type Locale, locales } from "@/lib/locales";
import createMiddleware from "next-intl/middleware";
import { type NextRequest, type NextResponse } from "next/server";

const nextIntlMiddleware = createMiddleware({
  locales,
  defaultLocale: "en" satisfies Locale,
  localePrefix: "never",
});

export default function (req: NextRequest): NextResponse {
  return nextIntlMiddleware(req);
}

export const config = {
  // match only internationalized pathnames
  matcher: [
    // Match all pathnames except for
    // - … if they start with `/api`, `/_next` or `/_vercel`
    // - … the ones containing a dot (e.g. `favicon.ico`)
    "/((?!api|_next|_vercel|.*\\..*).*)",
  ],
};
```

5. Update the `next.config.js` file:

```js
/**
 * Run `build` or `dev` with `SKIP_ENV_VALIDATION` to skip env validation. This is especially useful
 * for Docker builds.
 */
import withNextIntl from "next-intl/plugin";

await import("./src/env.js");

const nextIntlConfig = withNextIntl();

/** @type {import("next").NextConfig} */
const config = {
  images: {
    remotePatterns: [],
  },
};

export default nextIntlConfig(config);
```

6. Make sure to wrap your whole application with the `[locale]` route (right beneath the `app` directory).

7. Create a `layout.tsx` file under the `[locale]` page:

```tsx
import { type Locale } from "@/lib/locales";
import { getMessages, getTranslations } from "next-intl/server";
import { NextIntlClientProvider } from "next-intl";
import { cn } from "@/lib/cn";
import { type Metadata } from "next";

type Props = {
  children: React.ReactNode;
  params: {
    locale: Locale;
  };
};

const RootLayout: React.FC<Props> = async (props) => {
  const messages = await getMessages();

  return (
    <html lang={props.params.locale}>
      <body className={cn("font-sans bg-background", nunito.variable)}>
        <NextIntlClientProvider messages={messages}>
          {props.children}
        </NextIntlClientProvider>
      </body>
    </html>
  );
};

export async function generateMetadata({
  params: { locale },
}: {
  params: { locale: Locale };
}): Promise<Metadata> {
  const t = await getTranslations({ locale, namespace: "root" });

  return {
    title: t("metadata.title"),
    description: t("metadata.description"),
  };
}

export default RootLayout;
```

8. Create the `language-switcher.tsx` component:

```tsx
"use client";

import { Button } from "@/components/ui/button";
import { DropdownMenu } from "@/components/ui/dropdown-menu";
import { type Locale } from "@/lib/locales";
import { GlobeIcon } from "lucide-react";
import { useLocale } from "next-intl";
import { useRouter } from "next/navigation";
import React from "react";

export const LanguagePicker: React.FC = () => {
  const locale = useLocale() as Locale;
  const router = useRouter();

  function handleLocaleChange(newLocale: Locale): void {
    document.cookie = `NEXT_LOCALE=${newLocale}; path=/; max-age=31536000; SameSite=Lax`;
    router.refresh();
  }

  return (
    <DropdownMenu>
      <DropdownMenu.Trigger asChild>
        <Button type="button" variant="ghost" size="icon">
          <GlobeIcon className="size-5" />
        </Button>
      </DropdownMenu.Trigger>

      <DropdownMenu.Content align="end">
        <DropdownMenu.Label>Language</DropdownMenu.Label>
        <DropdownMenu.Separator />

        <DropdownMenu.CheckboxItem
          checked={locale === "en"}
          onClick={() => {
            handleLocaleChange("en");
          }}
        >
          English
        </DropdownMenu.CheckboxItem>
        <DropdownMenu.CheckboxItem
          checked={locale === "es"}
          onClick={() => {
            handleLocaleChange("es");
          }}
        >
          Spanish
        </DropdownMenu.CheckboxItem>
      </DropdownMenu.Content>
    </DropdownMenu>
  );
};
```

### 9. Bonus: i18n Merger

`scripts/merge-i18n.ts
```ts
import { existsSync, mkdirSync, readdirSync, readFileSync, statSync, writeFileSync } from "fs";
import { join } from "path";

class ScriptError extends Error {
  constructor(message: unknown) {
    // eslint-disable-next-line @typescript-eslint/restrict-template-expressions
    super(`\x1b[31m❌ [Error Processing i18n]: ${message}\x1b[0m`);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

// eslint-disable-next-line no-console
console.time("\x1b[32m[i18n Processing]:\x1b[0m Finished processing i18n in");

const locales = ["en", "es"] as const;

const messages: Record<string, Record<string, unknown>> = locales.reduce(
  (acc: Record<string, Record<string, unknown>>, locale: string) => {
    acc[locale] = {};
    return acc;
  },
  {}
);

locales.forEach((locale: string) => {
  const dirPath = `./src/messages`;
  const filePath = join(dirPath, `${locale}.json`);

  if (!existsSync(dirPath)) {
    mkdirSync(dirPath, { recursive: true });
  }

  if (!existsSync(filePath)) {
    writeFileSync(filePath, JSON.stringify(messages[locale] ?? {}), { flag: "wx" });
  }
});

const processDirectory = (directory: string): void => {
  readdirSync(directory).forEach((file: string) => {
    const absolute = join(directory, file);
    if (statSync(absolute).isDirectory()) {
      if (file === "i18n") {
        locales.forEach((locale: string) => {
          const filePath = join(absolute, `${locale}.json`);
          if (existsSync(filePath)) {
            const content = JSON.parse(readFileSync(filePath, "utf8")) as Record<string, unknown>;
            const keys = Object.keys(content);

            if (keys.length !== 1) {
              throw new ScriptError(
                `Expected one top-level key in ${filePath}, but found ${keys.length}`
              );
            }

            const key = keys[0];
            if (key === undefined) {
              throw new ScriptError(`No key found in ${filePath}`);
            }

            const value = content[key];
            if (value === undefined) {
              throw new ScriptError(`No value found for key ${key} in ${filePath}`);
            }

            const messagesAtLocale = messages[locale];
            if (messagesAtLocale === undefined) {
              throw new ScriptError(`No messages found for locale ${locale}`);
            }

            if (messagesAtLocale[key] !== undefined) {
              throw new ScriptError(`Duplicate key ${key} found in ${filePath}`);
            }

            messagesAtLocale[key] = value;
          }
        });
      } else {
        processDirectory(absolute);
      }
    }
  });
};

processDirectory("./src");

locales.forEach((locale: string) => {
  if (messages[locale] !== undefined) {
    writeFileSync(`./src/messages/${locale}.json`, JSON.stringify(messages[locale]));
    console.log(`\x1b[32m[i18n Processing]:\x1b[0m Successfully processed locale - ${locale}`);
  }
});

// eslint-disable-next-line no-console
console.timeEnd("\x1b[32m[i18n Processing]:\x1b[0m Finished processing i18n in");
```

`package.json`
```json
"scripts": {
  "i18n:merge": "SKIP_ENV_VALIDATION=true tsx ./scripts/merge-i18n.ts",
  "postinstall": "pnpm run i18n:merge"
}
```
