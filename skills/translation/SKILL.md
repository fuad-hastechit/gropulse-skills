---
name: translation
description: >
  Add full multilanguage support to a Gropulse Shopify app using i18next + react-i18next.
  Covers DB-persisted language per shop, auto-detection by billing country/customer locale,
  a language switcher UI component, and server-side/PDF label translations.
  Use when the user asks to "add language support", "translate the app UI", "add i18n",
  "add translation", "handle multilanguage", "add a language switcher", or "translate PDFs/server output".
---

# Skill: Multilanguage i18n Setup

## What this does

Two parallel translation systems:

1. **App UI translations** — `i18next` + `react-i18next` for all React components. Language stored per shop in the Prisma `Settings` table, loaded in the root route loader, switched client-side without a page reload.

2. **Server-side/PDF translations** — flat JSON key maps used directly in server functions (PDF rendering, Liquid templates). Language resolved at request time from billing country → customer locale → shop default.

---

## ASSUMPTIONS (state before proceeding)

```
ASSUMPTIONS I'M MAKING:
1. Stack is React Router v7 (flat file routes) + @shopify/shopify-app-react-router + Polaris Web Components
2. Root app layout is app/routes/app.tsx with a loader
3. Prisma Settings model has a `settings Json` column (flexible JSON blob) storing per-shop config
4. Settings are saved/loaded via app/models/settings.server.ts
5. Settings route is at /app/settings with an action that handles tabId-keyed updates
→ Correct me now or I'll proceed with these.
```

---

## Prerequisites

```bash
npm install i18next react-i18next
```

Verify `settings Json` column exists in Prisma schema — no migration needed if it's already a flexible JSON blob.

---

## Step 1 — Translation file structure

### App UI translations: `app/locales/`

Create one JSON file per language. Keys are nested objects accessed via dot-notation:

```
app/locales/
  en.json
  fr.json
  de.json
  es.json
  zh-CN.json
  zh-TW.json
  ja.json
  pt-BR.json
  pt-PT.json
  it.json
  hi.json
  nl.json
  vi.json
  el.json
  fi.json
  index.ts        ← barrel + config
```

**Sample `en.json` structure:**
```json
{
  "common": {
    "save": "Save",
    "cancel": "Cancel",
    "selected_one": "{{count}} selected",
    "selected_other": "{{count}} selected"
  },
  "nav": {
    "documents": "Documents",
    "settings": "Settings"
  },
  "documents": {
    "heading": "Documents",
    "actions": {
      "bulkGenerate": "Bulk Generate",
      "downloadDoc": "Download {{documentNumber}}"
    },
    "selection": {
      "ordersSelected_one": "{{count}} order selected",
      "ordersSelected_other": "{{count}} orders selected"
    }
  }
}
```

**`app/locales/index.ts`:**
```typescript
import en from "./en.json";
import zhCN from "./zh-CN.json";
import zhTW from "./zh-TW.json";
import fr from "./fr.json";
import de from "./de.json";
import ja from "./ja.json";
import es from "./es.json";
import ptBR from "./pt-BR.json";
import ptPT from "./pt-PT.json";
import it from "./it.json";
import hi from "./hi.json";
import nl from "./nl.json";
import vi from "./vi.json";
import el from "./el.json";
import fi from "./fi.json";

export const APP_LANGUAGES = [
  "en", "zh-CN", "zh-TW", "fr", "de", "ja", "es",
  "pt-BR", "pt-PT", "it", "hi", "nl", "vi", "el", "fi",
] as const;

export type AppLanguage = (typeof APP_LANGUAGES)[number];

export const APP_LANGUAGE_NAMES: Record<AppLanguage, string> = {
  en: "English",
  "zh-CN": "简体中文",
  "zh-TW": "繁體中文",
  fr: "Français",
  de: "Deutsch",
  ja: "日本語",
  es: "Español",
  "pt-BR": "Português (Brasil)",
  "pt-PT": "Português (Portugal)",
  it: "Italiano",
  hi: "हिन्दी",
  nl: "Nederlands",
  vi: "Tiếng Việt",
  el: "Ελληνικά",
  fi: "Suomi",
};

export const appTranslations: Record<string, Record<string, unknown>> = {
  en,
  "zh-CN": zhCN,
  "zh-TW": zhTW,
  fr, de, ja, es,
  "pt-BR": ptBR,
  "pt-PT": ptPT,
  it, hi, nl, vi, el, fi,
};
```

### Server-side/PDF translations: `app/config/translations/`

Used for rendered output (PDFs, Liquid). Keys are flat dot-notation strings:

```
app/config/translations/
  en.json
  de.json  fr.json  es.json  it.json
  pt.json  nl.json  ja.json  zh.json  ko.json
  index.ts
```

**Sample `en.json` structure:**
```json
{
  "document.invoice": "Invoice",
  "document.credit_note": "Credit Note",
  "label.from": "From",
  "label.billTo": "Bill To",
  "label.date": "Date",
  "label.dueDate": "Due Date",
  "table.description": "Description",
  "table.qty": "Qty",
  "table.unitPrice": "Unit Price",
  "table.total": "Total",
  "totals.subtotal": "Subtotal",
  "totals.shipping": "Shipping",
  "totals.tax": "Tax",
  "totals.total": "Total"
}
```

**`app/config/translations/index.ts`:**
```typescript
import en from "./en.json";
import de from "./de.json";
import fr from "./fr.json";
import es from "./es.json";
import it from "./it.json";
import pt from "./pt.json";
import nl from "./nl.json";
import ja from "./ja.json";
import zh from "./zh.json";
import ko from "./ko.json";

export type SupportedLanguage = "en" | "de" | "fr" | "es" | "it" | "pt" | "nl" | "ja" | "zh" | "ko";

export const SUPPORTED_LANGUAGES: SupportedLanguage[] = [
  "en", "de", "fr", "es", "it", "pt", "nl", "ja", "zh", "ko",
];

export const translations: Record<SupportedLanguage, Record<string, string>> = {
  en, de, fr, es, it, pt, nl, ja, zh, ko,
};

export const LANGUAGE_NAMES: Record<SupportedLanguage, string> = {
  en: "English", de: "Deutsch", fr: "Français", es: "Español",
  it: "Italiano", pt: "Português", nl: "Nederlands",
  ja: "日本語", zh: "中文", ko: "한국어",
};

export const CJK_LANGUAGES: SupportedLanguage[] = ["ja", "zh", "ko"];

export const LANGUAGE_TO_LOCALE: Record<SupportedLanguage, string> = {
  en: "en-US", de: "de-DE", fr: "fr-FR", es: "es-ES",
  it: "it-IT", pt: "pt-BR", nl: "nl-NL",
  ja: "ja-JP", zh: "zh-CN", ko: "ko-KR",
};
```

**Verify:**
```bash
npm run typecheck 2>&1 | grep -E "error" | head -20
```

---

## Step 2 — i18next initialization

**`app/i18n.ts`:**
```typescript
import i18n from "i18next";
import { initReactI18next } from "react-i18next";
import { appTranslations } from "~/locales";
import type { AppLanguage } from "~/locales";

export function createI18nInstance(language: AppLanguage = "en") {
  const instance = i18n.createInstance();
  instance.use(initReactI18next).init({
    lng: language,
    fallbackLng: "en",
    resources: Object.fromEntries(
      Object.entries(appTranslations).map(([lang, translations]) => [
        lang,
        { translation: translations },
      ]),
    ),
    interpolation: { escapeValue: false },
  });
  return instance;
}
```

All translations are statically bundled — no lazy loading. `createInstance` avoids polluting the global i18n singleton, which matters in SSR/React Router v7.

**Verify:**
```bash
npm run typecheck 2>&1 | grep -E "error" | head -20
```

---

## Step 3 — Root route wiring

**`app/routes/app.tsx`** — add `I18nextProvider` wrapping `<Outlet />`:

```typescript
import { useMemo, useEffect } from "react";
import { Outlet, useLoaderData } from "react-router";
import { I18nextProvider } from "react-i18next";
import type { LoaderFunctionArgs } from "react-router";

import { createI18nInstance } from "~/i18n";
import type { AppLanguage } from "~/locales";
import { getAppLanguage } from "~/models/settings.server";
import { authenticate, prisma } from "~/shopify.server";

export const loader = async ({ request }: LoaderFunctionArgs) => {
  const { session } = await authenticate.admin(request);
  const appLanguage = await getAppLanguage(prisma, session.shop);
  return {
    apiKey: process.env.SHOPIFY_API_KEY || "",
    appLanguage,
    // ... other existing loader fields
  };
};

export default function App() {
  const { apiKey, appLanguage } = useLoaderData<typeof loader>();
  const i18nInstance = useMemo(
    () => createI18nInstance(appLanguage as AppLanguage),
    [appLanguage],
  );

  useEffect(() => {
    document.documentElement.lang = appLanguage;
  }, [appLanguage]);

  return (
    <I18nextProvider i18n={i18nInstance}>
      {/* Your existing app shell here */}
      <Outlet />
    </I18nextProvider>
  );
}
```

**Verify:**
```bash
npm run typecheck 2>&1 | grep -E "error" | head -20
```

---

## Step 4 — Server-side language resolution

**`app/services/i18n.server.ts`:**
```typescript
import {
  translations,
  LANGUAGE_TO_LOCALE,
  SUPPORTED_LANGUAGES,
  type SupportedLanguage,
} from "~/config/translations";

const COUNTRY_TO_LANGUAGE: Record<string, SupportedLanguage> = {
  DE: "de", AT: "de", CH: "de",
  FR: "fr",
  ES: "es", MX: "es", AR: "es", CO: "es", CL: "es", PE: "es",
  IT: "it",
  BR: "pt", PT: "pt",
  NL: "nl", BE: "nl",
  JP: "ja",
  CN: "zh", TW: "zh", HK: "zh",
  KR: "ko",
};

interface ResolveLanguageOptions {
  defaultLanguage: string;
  autoDetectLanguage: boolean;
  customerLocale?: string;
  billingCountry?: string;
}

export function resolveLanguage(options: ResolveLanguageOptions): SupportedLanguage {
  const { defaultLanguage, autoDetectLanguage, customerLocale, billingCountry } = options;

  if (autoDetectLanguage) {
    if (billingCountry) {
      const mapped = COUNTRY_TO_LANGUAGE[billingCountry.toUpperCase()];
      if (mapped) return mapped;
    }
    if (customerLocale) {
      const langCode = customerLocale.split("-")[0].toLowerCase();
      if (isSupportedLanguage(langCode)) return langCode;
    }
  }

  const defaultLang = defaultLanguage.split("-")[0].toLowerCase();
  if (isSupportedLanguage(defaultLang)) return defaultLang;
  return "en";
}

function isSupportedLanguage(code: string): code is SupportedLanguage {
  return SUPPORTED_LANGUAGES.includes(code as SupportedLanguage);
}

export function getLabels(language: SupportedLanguage): Record<string, string> {
  return translations[language] || translations.en;
}

export function formatDateForLocale(
  date: Date | string,
  language: SupportedLanguage,
): string {
  const d = typeof date === "string" ? new Date(date) : date;
  const locale = LANGUAGE_TO_LOCALE[language] || "en-US";
  return d.toLocaleDateString(locale, { month: "short", day: "numeric", year: "numeric" });
}

const COUNTRY_TO_LOCALE: Record<string, string> = {
  DE: "de-DE", AT: "de-AT", CH: "de-CH",
  FR: "fr-FR", ES: "es-ES", IT: "it-IT", PT: "pt-PT",
  NL: "nl-NL", BE: "nl-BE",
  SE: "sv-SE", NO: "nb-NO", DK: "da-DK", FI: "fi-FI",
  US: "en-US", CA: "en-CA", GB: "en-GB", AU: "en-AU", NZ: "en-NZ",
  MX: "es-MX", BR: "pt-BR", AR: "es-AR",
  JP: "ja-JP", CN: "zh-CN", TW: "zh-TW", HK: "zh-HK", KR: "ko-KR",
  IN: "en-IN", SG: "en-SG",
};

export function getNumberLocale(
  billingCountry: string | undefined,
  labelLanguage: SupportedLanguage,
): string {
  if (billingCountry) {
    const mapped = COUNTRY_TO_LOCALE[billingCountry.toUpperCase()];
    if (mapped) return mapped;
  }
  return LANGUAGE_TO_LOCALE[labelLanguage] || "en-US";
}
```

**Verify:**
```bash
npm run typecheck 2>&1 | grep -E "error" | head -20
```

---

## Step 5 — Language switcher component

**`app/components/AppLanguageSelector.tsx`:**
```typescript
import { useFetcher, useRevalidator } from "react-router";
import { useTranslation } from "react-i18next";
import { APP_LANGUAGES, APP_LANGUAGE_NAMES } from "~/locales";
import type { AppLanguage } from "~/locales";

export function AppLanguageSelector() {
  const { i18n } = useTranslation();
  const fetcher = useFetcher();
  const revalidator = useRevalidator();

  const currentLang = i18n.language as AppLanguage;
  const currentLangName = APP_LANGUAGE_NAMES[currentLang] || "English";

  const handleLangChange = (lang: string) => {
    if (lang === currentLang) return;
    i18n.changeLanguage(lang);
    document.documentElement.lang = lang;
    fetcher.submit(
      { tabId: "appLanguage", data: JSON.stringify({ appLanguage: lang }) },
      { method: "post", action: "/app/settings" },
    );
    revalidator.revalidate();
  };

  return (
    <>
      <s-button slot="secondary-actions" commandFor="language-menu" icon="globe">
        {currentLangName}
      </s-button>
      <s-menu id="language-menu">
        {APP_LANGUAGES.map((lang) => (
          <s-button
            key={lang}
            icon={lang === currentLang ? "check" : undefined}
            onClick={() => handleLangChange(lang)}
          >
            {APP_LANGUAGE_NAMES[lang]}
          </s-button>
        ))}
      </s-menu>
    </>
  );
}
```

### Critical placement rules

**Do NOT place `<AppLanguageSelector />` inside `<s-app-nav>`.** The `s-app-nav` web component only renders `<a>` anchor children via its Shadow DOM slot — any non-`<a>` element (including `<s-button>`) is silently dropped.

**Do NOT add it in the root layout (`app.tsx`)** as a standalone sibling — that bypasses the Polaris slot system and loses header-integrated styling.

**Correct pattern:** Add `<AppLanguageSelector />` as the **first child of `<s-page>`** in every route file:

```tsx
// app/routes/app.some-page.tsx
export default function SomePage() {
  const { t } = useTranslation();
  return (
    <s-page heading={t("page.heading")}>
      <AppLanguageSelector />   {/* ← first child, slot="secondary-actions" targets header top-right */}
      <s-section>
        {/* ... rest of page ... */}
      </s-section>
    </s-page>
  );
}
```

Every route with an `<s-page>` wrapper needs its own `<AppLanguageSelector />` import and child element.

---

## Step 6 — Settings persistence

**`app/models/settings.server.ts` — add:**
```typescript
import type { PrismaClient } from "@prisma/client";

export async function getAppLanguage(
  prisma: PrismaClient,
  shopDomain: string,
): Promise<string> {
  const record = await prisma.settings.findUnique({
    where: { shop: shopDomain },
    select: { settings: true },
  });
  if (!record) return "en";
  const saved = record.settings as Record<string, unknown>;
  return (saved.appLanguage as string) || "en";
}
```

**Default settings schema:**
```typescript
export function getDefaultSettingsJson() {
  return {
    appLanguage: "en",
    // ... other fields
  };
}
```

**Settings route action — in `app/routes/app.settings.tsx`:**
```typescript
if (tabId === "appLanguage") {
  const parsed = JSON.parse(data as string);
  await saveSettingsForShop(prisma, session.shop, "appLanguage", parsed.appLanguage);
}
```

`saveSettingsForShop` does a Prisma `upsert` that merges `appLanguage` into the existing JSON blob.

**Verify:**
```bash
npm run typecheck 2>&1 | grep -E "error" | head -20
```

---

## Step 7 — Using translations in components

### Basic usage
```typescript
import { useTranslation } from "react-i18next";

function MyComponent() {
  const { t } = useTranslation();
  return <p>{t("documents.heading")}</p>;
}
```

### Interpolation
```typescript
t("documents.actions.downloadDoc", { documentNumber: "#1001" })
// → "Download #1001"
```

### Pluralization
```typescript
t("documents.selection.ordersSelected", { count: 3 })
// → "3 orders selected"
```

Translation keys must have `_one` / `_other` suffixes:
```json
"ordersSelected_one": "{{count}} order selected",
"ordersSelected_other": "{{count}} orders selected"
```

### JSX with embedded markup — use `<Trans>`
```typescript
import { Trans } from "react-i18next";

<Trans
  i18nKey="deleteModal.confirmMessage"
  values={{ orderName: "#1001" }}
  components={{ 1: <strong /> }}
/>
```

---

## Step 8 — Build and smoke test

```bash
npm run build
npm run dev
```

Manual verification checks:

| Check | How to test |
|-------|-------------|
| App loads in English by default | Open app for a new shop — UI text should be English |
| Language switch updates UI immediately | Switch to French — no page reload, all strings change |
| Language persists after reload | Switch to German, refresh — still German |
| `document.documentElement.lang` updates | Inspect `<html lang="">` after switching |
| Server-rendered PDF uses correct language | Generate a PDF for a French order — labels in French |
| Date formatting matches locale | French order → date shows "15 janv. 2025" not "Jan 15, 2025" |
| Plural forms render | Select 1 order vs 3 orders — singular/plural forms correct |

---

## Porting checklist

When applying to a new project:

- [ ] Reduce `APP_LANGUAGES` to only languages needed (fewer = faster translation effort)
- [ ] Reduce `SUPPORTED_LANGUAGES` (server-side) similarly
- [ ] Replace `s-button`/`s-menu` in `AppLanguageSelector` with your UI framework's dropdown if not using Shopify `s-*` web components
- [ ] Keep `slot="secondary-actions"` on the trigger `<s-button>` — required for `<s-page>` to render it in the header top-right
- [ ] Add `<AppLanguageSelector />` as first child of `<s-page>` in **every** route file
- [ ] Change `action: "/app/settings"` in the fetcher submit to your actual settings route
- [ ] Update `tabId: "appLanguage"` to match your settings action handler's key
- [ ] Add app-specific translation keys to `en.json` first — other language files mirror its structure
- [ ] If `settings` column is typed (not a free JSON blob), add `appLanguage String @default("en")` column instead
- [ ] Remove unused `CJK_LANGUAGES` export if not handling CJK-specific font/layout logic
- [ ] If not needed, skip server-side PDF translation (Step 1 server files + Step 4) entirely
- [ ] Test with a shop whose Shopify admin language is non-English to verify the fallback chain works

---

## Files changed summary

| File | Change |
|------|--------|
| `package.json` | Add `i18next`, `react-i18next` dependencies |
| `app/locales/en.json` + other lang files | New — app UI translation strings |
| `app/locales/index.ts` | New — barrel, `APP_LANGUAGES`, `APP_LANGUAGE_NAMES`, `appTranslations` |
| `app/config/translations/en.json` + other lang files | New — server-side/PDF flat label maps |
| `app/config/translations/index.ts` | New — `SupportedLanguage`, `translations`, `LANGUAGE_TO_LOCALE` |
| `app/i18n.ts` | New — `createI18nInstance` factory |
| `app/routes/app.tsx` | Add `appLanguage` to loader return; wrap output in `<I18nextProvider>` |
| `app/services/i18n.server.ts` | New — `resolveLanguage`, `getLabels`, `formatDateForLocale`, `getNumberLocale` |
| `app/components/AppLanguageSelector.tsx` | New — language switcher component |
| `app/models/settings.server.ts` | Add `getAppLanguage` function |
| `app/routes/app.settings.tsx` | Add `appLanguage` tab handler in action |
