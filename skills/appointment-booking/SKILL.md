---
name: appointment-booking
description: >
  Add a free growth strategy call booking feature to a Gropulse Shopify app.
  Installs @gropulse/booking-widget, wires shop context from the app loader,
  adds a dedicated booking page, and adds a floating CTA button in the app shell.
  Use when the user asks to "add appointment booking", "add the booking widget",
  "add free strategy call", or "integrate @gropulse/booking-widget".
---

# Skill: Add Appointment Booking (Free Growth Strategy Call)

## What this does

Adds a "Book a Free Growth Strategy Call" feature to a Gropulse embedded Shopify app:

1. **Installs** `@gropulse/booking-widget` from npm
2. **Exposes** shop context fields in the root app loader (`bookingApiUrl`, `ownerName`, `email`, `plan`, `installedSince`, `timezone`)
3. **Creates** a dedicated `/app/growth-call` page (Polaris Web Components, inline `BookingWidget`)
4. **Adds** a fixed floating CTA button in the app shell that hides on the growth-call page itself

The widget talks to `https://booking.gropulse.com` — a deployed AI chat + Cal.com booking API. No backend work is needed in the app.

---

## Two distinct "Book Free Strategy Call" buttons

There are **two separate buttons** with the same label but completely different roles, appearances, and locations:

### Button 1 — Floating CTA (app shell, every page)

- **Location:** Fixed position, bottom-left corner of every app page (`position: "fixed", bottom: "24px", left: "24px"`)
- **Defined in:** `app/routes/app.tsx` inside `AppShell`, after `<Outlet />`
- **Visible on:** Every page **except** `/app/growth-call` (hidden via `isGrowthCallPage` check)
- **Action:** Navigates to `/app/growth-call` via `useNavigate` from `react-router` — `onClick={() => navigate("/app/growth-call")}`. NOT `<Link>`.
- **Renders as:** Raw HTML `<button>` with inline styles. Not `<Link>`, not `<s-button>` — Polaris Web Components don't render reliably in fixed-position overlays inside the Shopify Admin iframe
- **Layout:** `display: "flex"`, `alignItems: "center"`, `gap: "10px"` (icon left, text right)
- **Design:**
  - Background: `#0d1f3c` (dark navy)
  - Text color: `#ffffff`
  - Font size: `15px`
  - Font weight: `500` (medium — inherits system/browser font stack, no explicit `fontFamily`)
  - Letter spacing: `0.01em`
  - White-space: `nowrap`
  - Padding: `16px 18px`
  - Border: `none`
  - Border radius: `14px`
  - Cursor: `pointer`
  - Box shadow: `0 2px 12px rgba(0,0,0,0.35)`
  - Transition: `opacity 0.15s ease, transform 0.1s ease`
  - Hover: opacity → `0.88` + `translateY(-1px)` lift; mouse-leave restores opacity `1` + `translateY(0)` (cast `e.currentTarget` as `HTMLButtonElement`)
  - z-index: `9998` (below Shopify Admin modals at 9999+)
- **Icon:** Calendar SVG, `width="16" height="16" viewBox="0 0 20 20"`, `fill="none"`, `aria-hidden="true"`, `style={{ flexShrink: 0 }}`. Exact paths:
  ```svg
  <rect x="3" y="4" width="14" height="14" rx="2.5" stroke="white" strokeWidth="1.6"/>
  <path d="M3 8h14" stroke="white" strokeWidth="1.6"/>
  <path d="M7 2v3M13 2v3" stroke="white" strokeWidth="1.6" strokeLinecap="round"/>
  <circle cx="7" cy="12" r="1" fill="white"/>
  <circle cx="10" cy="12" r="1" fill="white"/>
  <circle cx="13" cy="12" r="1" fill="white"/>
  ```
  Shapes: rounded calendar body, horizontal header divider, two tick marks above top edge, three dots in lower grid cells.

### Button 2 — In-page booking trigger (growth-call page only)

- **Location:** Inside the `/app/growth-call` page content, below the benefits list
- **Defined in:** `app/routes/app.growth-call.tsx` as `<BookingWidget trigger="button" buttonLabel="Book Free Strategy Call" />`
- **Visible on:** Only `/app/growth-call`
- **Action:** Opens the booking chat widget inline (AI conversation → slot selection → confirmation)
- **Renders as:** Styled by the `@gropulse/booking-widget` library — not a raw `<button>`, not inline styles
- **Design:** Controlled entirely by the widget library's built-in styles (imported via `"@gropulse/booking-widget/styles.css"`)

**Do NOT confuse these two.** They share the same label but serve different purposes:
- Floating button = navigation shortcut to the booking page (visible everywhere)
- In-page button = actual booking action that launches the AI chat (visible only on the booking page)

---

## ASSUMPTIONS (state before proceeding)

```
ASSUMPTIONS I'M MAKING:
1. Stack is React Router v7 (flat file routes) + @shopify/shopify-app-react-router + Polaris Web Components
2. Root app layout is app/routes/app.tsx with a loader that returns apiKey + shopDomain at minimum
3. Shop model exposes: shopOwner, email, planName, installedAt, ianaTimezone
4. The booking API is already deployed at https://booking.gropulse.com (no backend work needed)
5. BOOKING_API_URL env var is optional — hardcoded default is https://booking.gropulse.com
→ Correct me now or I'll proceed with these.
```

---

## Prerequisites

Before starting, verify:

```bash
# Confirm flat file routing is in use
grep -r "flatRoutes" app/routes.ts

# Confirm app.tsx loader exists and returns shopDomain
grep -n "shopDomain" app/routes/app.tsx

# Confirm Shop model has the required fields
grep -n "shopOwner\|ianaTimezone\|installedAt" prisma/schema.prisma
```

If `shopOwner`, `ianaTimezone`, or `installedAt` are missing from the Prisma schema, stop and ask the user before continuing.

---

## Step 1 — Install the package

```bash
npm install @gropulse/booking-widget
```

**Verify:**
```bash
node -e "require('@gropulse/booking-widget')" 2>/dev/null && echo OK || echo FAIL
ls node_modules/@gropulse/booking-widget/dist/index.d.ts
```

Expected: `OK` and the `.d.ts` file exists.

**Package exports:**
```ts
import { BookingWidget } from "@gropulse/booking-widget";
import "@gropulse/booking-widget/styles.css";
import { useBookingChat } from "@gropulse/booking-widget"; // low-level hook, only if building custom UI
```

---

## Step 2 — Extend the app.tsx loader

The `app/routes/app.tsx` root loader must expose these fields so child routes can read them via `useRouteLoaderData`:

| Field | Source | Notes |
|-------|--------|-------|
| `bookingApiUrl` | `process.env.BOOKING_API_URL` | Default (production) URL is **`https://booking.gropulse.com`** — env var only needed for local dev override |
| `ownerName` | `shop?.shopOwner` | String, empty string fallback |
| `email` | `shop?.email` | String, empty string fallback |
| `plan` | `shop?.planName` | String, `"free"` fallback |
| `installedSince` | `shop?.installedAt` | ISO string, `new Date()` fallback |
| `timezone` | `shop?.ianaTimezone` | Optional string (`undefined` if missing) |

**Edit `app/routes/app.tsx` loader return block:**

```diff
  return {
    apiKey: process.env.SHOPIFY_API_KEY || "",
    shopDomain: session.shop,
    appLanguage: settings?.appLanguage ?? "en",
+   appName: process.env.SHOPIFY_APP_NAME || "GroPulse App",
+   ownerName: shop?.shopOwner ?? "",
+   email: shop?.email ?? "",
+   plan: shop?.planName ?? "free",
+   installedSince: (shop?.installedAt ?? new Date()).toISOString(),
+   timezone: shop?.ianaTimezone ?? undefined,
+   bookingApiUrl: process.env.BOOKING_API_URL ?? "https://booking.gropulse.com",
  };
```

**Verify:**
```bash
npm run typecheck 2>&1 | grep -E "error|warning" | head -20
```

Expected: no new TypeScript errors.

---

## Step 3 — Add floating CTA button to app shell

This is **Button 1** — the navigation button visible on every page except `/app/growth-call`.

In `app/routes/app.tsx`, update the `AppShell` component to:
1. Import `useLocation` and `useNavigate` from `react-router`
2. Hide the button when already on the growth-call page
3. Render a fixed-position custom button that navigates to `/app/growth-call`

**Import line change:**
```diff
-import { Outlet, useLoaderData, useRouteError } from "react-router";
+import { Outlet, useLoaderData, useLocation, useNavigate, useRouteError } from "react-router";
```

**Inside `AppShell` function body, before the return:**
```tsx
const navigate = useNavigate();
const location = useLocation();
const isGrowthCallPage = location.pathname === "/app/growth-call";
```

**Inside the JSX, after `<Outlet />` and before closing `</AppProvider>`:**
```tsx
{!isGrowthCallPage && (
  <div style={{ position: "fixed", bottom: "24px", left: "24px", zIndex: 9998 }}>
    <button
      onClick={() => navigate("/app/growth-call")}
      style={{
        display: "flex",
        alignItems: "center",
        gap: "10px",
        padding: "16px 18px",
        fontSize: "15px",
        fontWeight: 500,
        color: "#ffffff",
        background: "#0d1f3c",
        border: "none",
        borderRadius: "14px",
        cursor: "pointer",
        boxShadow: "0 2px 12px rgba(0,0,0,0.35)",
        letterSpacing: "0.01em",
        whiteSpace: "nowrap",
        transition: "opacity 0.15s ease, transform 0.1s ease",
      }}
      onMouseEnter={(e) => {
        (e.currentTarget as HTMLButtonElement).style.opacity = "0.88";
        (e.currentTarget as HTMLButtonElement).style.transform = "translateY(-1px)";
      }}
      onMouseLeave={(e) => {
        (e.currentTarget as HTMLButtonElement).style.opacity = "1";
        (e.currentTarget as HTMLButtonElement).style.transform = "translateY(0)";
      }}
    >
      <svg width="16" height="16" viewBox="0 0 20 20" fill="none" aria-hidden="true" style={{ flexShrink: 0 }}>
        <rect x="3" y="4" width="14" height="14" rx="2.5" stroke="white" strokeWidth="1.6"/>
        <path d="M3 8h14" stroke="white" strokeWidth="1.6"/>
        <path d="M7 2v3M13 2v3" stroke="white" strokeWidth="1.6" strokeLinecap="round"/>
        <circle cx="7" cy="12" r="1" fill="white"/>
        <circle cx="10" cy="12" r="1" fill="white"/>
        <circle cx="13" cy="12" r="1" fill="white"/>
      </svg>
      Book Free Strategy Call
    </button>
  </div>
)}
```

**Design decisions:**
- Raw `<button>` + `useNavigate` for programmatic navigation. Not `<Link>`, not `<s-button>` — Polaris Web Components don't render reliably in fixed-position overlays inside the Shopify Admin iframe.
- `zIndex: 9998` keeps it below Shopify Admin modals (z-index 9999+).
- Hidden on `/app/growth-call` to avoid visual redundancy with Button 2.
- **Do NOT add `/app/growth-call` to `<s-app-nav>`** — hidden page, access only via this floating button.

**Verify:**
```bash
npm run typecheck 2>&1 | grep -E "error" | head -20
```

---

## Step 4 — Create the growth-call route

Create `app/routes/app.growth-call.tsx`. Flat file route → maps to `/app/growth-call`, child of `app.tsx` layout.

This page contains **Button 2** — the in-page `BookingWidget` trigger that opens the AI booking chat.

**Key constraints:**
- No loader — reads parent data via `useRouteLoaderData("routes/app")`
- Import `BookingWidget` and its CSS here (not in `app.tsx`)
- Polaris Web Components only for layout (`s-page`, `s-section`, `s-stack`, `s-grid`, `s-paragraph`, `s-icon`)
- Inline `BookingWidget` with `trigger="button"` — this renders a styled button via the widget library, NOT a raw `<button>`
- **Page content (BENEFITS, headings, copy, button label) is identical across all Gropulse apps — copy verbatim**
- Only `appName` in the context object changes per app (passed to the AI, not displayed on page)

**Exact page content (copy verbatim, do not alter):**

| Element | Exact text |
|---------|-----------|
| Page heading (`s-page heading`) | `Growth Strategy Call` |
| Subheading (`div` bold) | `Book a free 30-minute Growth Strategy call` |
| Subtitle (`s-paragraph color="subdued"`) | `A working session with a Gropulse Shopify specialist - no sales pitch, no commitment.` |
| Benefit 1 | `Get specific, actionable tactics tailored to your store - we already have your context.` |
| Benefit 2 | `Find the biggest growth lever you're missing - conversion, AOV, retention, or paid acquisition.` |
| Benefit 3 | `Walk away with a 3-step plan you can ship the same week.` |
| Benefit 4 | `Talk to someone who's grown 100+ Shopify stores - not a generic consultant.` |
| Benefit 5 | `Completely free - yours because you're already a Gropulse customer.` |
| Button label (`BookingWidget`) | `Book Free Strategy Call` |

**Complete file:**

```tsx
import { useRouteLoaderData } from "react-router";
import { BookingWidget } from "@gropulse/booking-widget";
import "@gropulse/booking-widget/styles.css";
import type { loader } from "~/routes/app";

const BENEFITS = [
  "Get specific, actionable tactics tailored to your store - we already have your context.",
  "Find the biggest growth lever you're missing - conversion, AOV, retention, or paid acquisition.",
  "Walk away with a 3-step plan you can ship the same week.",
  "Talk to someone who's grown 100+ Shopify stores - not a generic consultant.",
  "Completely free - yours because you're already a Gropulse customer.",
];

export default function GrowthCallPage() {
  const data = useRouteLoaderData<typeof loader>("routes/app")!;

  return (
    <s-page heading="Growth Strategy Call">
      <s-section>
        <s-stack gap="large">
          <s-stack gap="small-100">
            <div style={{ fontSize: "16px", fontWeight: "bold" }}>Book a free 30-minute Growth Strategy call</div>
            <s-paragraph color="subdued">
              A working session with a Gropulse Shopify specialist - no sales pitch, no commitment.
            </s-paragraph>
          </s-stack>

          <s-stack gap="small-200">
            {BENEFITS.map((benefit, i) => (
              <s-grid key={i} gridTemplateColumns="auto 1fr" gap="small-200" alignItems="start">
                <s-icon type="check-circle" tone="success" />
                <s-paragraph>{benefit}</s-paragraph>
              </s-grid>
            ))}
          </s-stack>

          <BookingWidget
            apiBaseUrl={data.bookingApiUrl}
            context={{
              appName: data.appName,
              shopDomain: data.shopDomain,
              ownerName: data.ownerName,
              email: data.email,
              plan: data.plan,
              installedSince: data.installedSince,
              timezone: data.timezone,
            }}
            trigger="button"
            buttonLabel="Book Free Strategy Call"
          />
        </s-stack>
      </s-section>
    </s-page>
  );
}
```

**Verify:**
```bash
npm run typecheck 2>&1 | grep -E "error" | head -20
npm run build 2>&1 | tail -20
```

---

## Step 5 — Optional env var

The booking URL is hardcoded to `https://booking.gropulse.com` by default. No env var is required for production.

Add both vars to `.env.example` to document them:

```bash
SHOPIFY_APP_NAME=
# Set to the app's display name — injected into the booking AI system prompt

BOOKING_API_URL=
# Leave blank for production — hardcoded default is https://booking.gropulse.com
# Set to http://localhost:8787 only when running the booking API locally
```

---

## Step 6 — Build and smoke test

```bash
npm run build
npm run dev
```

Manual checks:
1. Open any app page → **Button 1** (floating, dark navy, bottom-left) visible
2. Hover Button 1 → slight lift + opacity drop to 0.88
3. Click Button 1 → navigates to `/app/growth-call`
4. On `/app/growth-call` → **Button 1 is hidden** (no duplicate)
5. On `/app/growth-call` → **Button 2** (widget-styled, inside page content) is visible below the benefits list
6. Click Button 2 → opens booking chat widget inline
7. Chat progresses → AI asks context questions → slots appear → booking can be confirmed

---

## Widget API reference

### `BookingWidget` props

```ts
interface BookingWidgetProps {
  apiBaseUrl: string;          // always "https://booking.gropulse.com" in production
  context: ShopContext;        // merchant + app data for AI personalization
  trigger?: "button" | "auto"; // default: "button"
  buttonLabel?: string;
  onBookingConfirmed?: (booking: BookingInfo) => void;
  className?: string;
}
```

### `ShopContext` fields

```ts
interface ShopContext {
  appName: string;          // REQUIRED — injected into AI system prompt (set via SHOPIFY_APP_NAME env var)
  shopDomain: string;       // REQUIRED — e.g. "mystore.myshopify.com"
  ownerName: string;        // REQUIRED — merchant name for personalization
  email: string;            // REQUIRED — pre-fills booking attendee email
  plan: string;             // REQUIRED — e.g. "free", "growth", "scale"
  installedSince: string;   // REQUIRED — ISO date string
  timezone?: string;        // OPTIONAL — IANA timezone for slot display
  extraContext?: Record<string, unknown>; // OPTIONAL — app-specific metrics
}
```

### `extraContext` by app type

```ts
// URL Redirects Manager
extraContext: { totalRedirects: number, total404Errors: number, redirectHitsThisMonth: number }

// Reviews app
extraContext: { totalReviews: number, averageRating: number, monthlyOrders: number }

// SEO app
extraContext: { seoScore: number, indexedPages: number, organicTraffic: number }

// Upsell app
extraContext: { upsellRevenue: number, conversionRate: number, activeOffers: number }

// Analytics app
extraContext: { monthlyRevenue: number, returningCustomerRate: number, topChannel: string }
```

---

## Adapting for a different Gropulse app

When adding this to a new app:

1. **Set `SHOPIFY_APP_NAME`** env var — injected into AI system prompt via `data.appName`, no code change needed
2. **Add relevant `extraContext`** fields from the table above
3. **Keep `bookingApiUrl`** pointing to `https://booking.gropulse.com` — same API for all apps
4. **Verify the parent loader** exposes `shopOwner`, `email`, `planName`, `installedAt`, `ianaTimezone`

**Page content is identical across all Gropulse apps.** `BENEFITS`, headings, copy, button label — copy verbatim. Do not change them per app.

---

## Failure modes

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Widget chat shows error | `apiBaseUrl` wrong or API down | Check `https://booking.gropulse.com/health`; verify `BOOKING_API_URL` env var |
| No slots in chat | Cal.com calendar full or timezone mismatch | Check Cal.com admin; verify `timezone` is valid IANA string |
| TypeScript error on `data.bookingApiUrl` | Loader field not added | Re-check Step 2 — all 6 fields must be in loader return |
| Floating button (Button 1) shows on growth-call page | `pathname` check failing | Verify `location.pathname === "/app/growth-call"` — check for trailing slash |
| Both buttons visible on growth-call page | Same as above | Button 1 should be hidden; only Button 2 (widget) shows on that page |
| Widget styles broken | CSS not imported | Confirm `import "@gropulse/booking-widget/styles.css"` in `app.growth-call.tsx` |
| `useRouteLoaderData` returns `undefined` | Route ID string wrong | Must be exactly `"routes/app"` — matches `app/routes/app.tsx` |

---

## Step 7 — Translation (if project uses the translation skill)

**This step is always last. Before doing anything, ask the user:**

```
I can see this project has i18next set up (app/locales/ exists).
Should I translate the appointment booking UI?
This will move all hardcoded strings in the floating button and the growth-call page into your locale files.

Reply YES to proceed, NO to skip.
```

**Only continue if the user explicitly says yes.**

---

### What to translate

There are three locations with hardcoded strings to move into locale files:

**Button 1 — floating CTA label (in `app/routes/app.tsx`):**
- `"Book Free Strategy Call"` (link text)

**Growth-call page (`app/routes/app.growth-call.tsx`):**
- Page heading: `"Growth Strategy Call"`
- Subheading: `"Book a free 30-minute Growth Strategy call"`
- Subtitle: `"A working session with a Gropulse Shopify specialist - no sales pitch, no commitment."`
- All 5 BENEFITS strings
- Button 2 label: `"Book Free Strategy Call"`

---

### Step 7a — Add translation keys to `en.json`

Add a `growthCall` namespace to `app/locales/en.json`:

```json
{
  "growthCall": {
    "pageHeading": "Growth Strategy Call",
    "subheading": "Book a free 30-minute Growth Strategy call",
    "subtitle": "A working session with a Gropulse Shopify specialist - no sales pitch, no commitment.",
    "benefit1": "Get specific, actionable tactics tailored to your store - we already have your context.",
    "benefit2": "Find the biggest growth lever you're missing - conversion, AOV, retention, or paid acquisition.",
    "benefit3": "Walk away with a 3-step plan you can ship the same week.",
    "benefit4": "Talk to someone who's grown 100+ Shopify stores - not a generic consultant.",
    "benefit5": "Completely free - yours because you're already a Gropulse customer.",
    "ctaButton": "Book Free Strategy Call"
  }
}
```

Mirror these keys into every other language file under `app/locales/` — translate each string into the target language.

---

### Step 7b — Update `app/routes/app.tsx` (Button 1)

Button 1 is in `AppShell` — add `useTranslation` and replace the hardcoded label:

```diff
+import { useTranslation } from "react-i18next";

 function AppShell() {
+  const { t } = useTranslation();
   const location = useLocation();
   const isGrowthCallPage = location.pathname === "/app/growth-call";
   // ...

-      Book Free Strategy Call
+      {t("growthCall.ctaButton")}
```

---

### Step 7c — Update `app/routes/app.growth-call.tsx` (page + Button 2)

Replace all hardcoded strings with `t()` calls. The `BENEFITS` array becomes translation keys:

```diff
+import { useTranslation } from "react-i18next";

+const BENEFIT_KEYS = [
+  "growthCall.benefit1",
+  "growthCall.benefit2",
+  "growthCall.benefit3",
+  "growthCall.benefit4",
+  "growthCall.benefit5",
+] as const;

-const BENEFITS = [
-  "Get specific, actionable tactics tailored to your store - we already have your context.",
-  "Find the biggest growth lever you're missing - conversion, AOV, retention, or paid acquisition.",
-  "Walk away with a 3-step plan you can ship the same week.",
-  "Talk to someone who's grown 100+ Shopify stores - not a generic consultant.",
-  "Completely free - yours because you're already a Gropulse customer.",
-];

 export default function GrowthCallPage() {
+  const { t } = useTranslation();
   const data = useRouteLoaderData<typeof loader>("routes/app")!;

   return (
-    <s-page heading="Growth Strategy Call">
+    <s-page heading={t("growthCall.pageHeading")}>
       <s-section>
         <s-stack gap="large">
           <s-stack gap="small-100">
-            <div style={{ fontSize: "16px", fontWeight: "bold" }}>Book a free 30-minute Growth Strategy call</div>
+            <div style={{ fontSize: "16px", fontWeight: "bold" }}>{t("growthCall.subheading")}</div>
             <s-paragraph color="subdued">
-              A working session with a Gropulse Shopify specialist - no sales pitch, no commitment.
+              {t("growthCall.subtitle")}
             </s-paragraph>
           </s-stack>

           <s-stack gap="small-200">
-            {BENEFITS.map((benefit, i) => (
-              <s-grid key={i} gridTemplateColumns="auto 1fr" gap="small-200" alignItems="start">
+            {BENEFIT_KEYS.map((key) => (
+              <s-grid key={key} gridTemplateColumns="auto 1fr" gap="small-200" alignItems="start">
                 <s-icon type="check-circle" tone="success" />
-                <s-paragraph>{benefit}</s-paragraph>
+                <s-paragraph>{t(key)}</s-paragraph>
               </s-grid>
             ))}
           </s-stack>

           <BookingWidget
             // ...
-            buttonLabel="Book Free Strategy Call"
+            buttonLabel={t("growthCall.ctaButton")}
           />
```

---

### Step 7d — Verify

```bash
npm run typecheck 2>&1 | grep -E "error" | head -20
npm run build 2>&1 | tail -20
```

Manual check: switch app language → floating button label and all growth-call page text update immediately with no page reload.

---

## Files changed summary

| File | Change |
|------|--------|
| `package.json` | Add `@gropulse/booking-widget` dependency |
| `app/routes/app.tsx` | Add 6 loader fields; add `useLocation`/`useNavigate`; add floating CTA button (Button 1) |
| `app/routes/app.growth-call.tsx` | New file — growth strategy call landing page with BookingWidget (Button 2) |
| `.env.example` | Add `BOOKING_API_URL=` (optional) |
| `app/locales/en.json` *(if translation skill present)* | Add `growthCall` namespace keys |
| `app/locales/*.json` *(if translation skill present)* | Mirror and translate `growthCall` keys in all language files |
| `app/routes/app.tsx` *(if translation skill present)* | Replace hardcoded Button 1 label with `t("growthCall.ctaButton")` |
| `app/routes/app.growth-call.tsx` *(if translation skill present)* | Replace all hardcoded strings with `t()` calls |
