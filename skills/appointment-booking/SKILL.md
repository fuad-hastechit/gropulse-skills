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

## ASSUMPTIONS (state before proceeding)

```
ASSUMPTIONS I'M MAKING:
1. Stack is React Router v7 (flat file routes) + @shopify/shopify-app-react-router + Polaris Web Components
2. Root app layout is app/routes/app.tsx with a loader that returns apiKey + shopDomain at minimum
3. Shop model exposes: shopOwner, email, planName, installedAt, ianaTimezone
4. The booking API is already deployed at https://booking.gropulse.com (no backend work needed)
5. BOOKING_API_URL env var is optional — widget falls back to https://booking.gropulse.com
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
| `bookingApiUrl` | `process.env.BOOKING_API_URL` | Falls back to `https://booking.gropulse.com` |
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
- Uses raw `<button>` + inline styles (not `<s-button>`) because the button sits outside Polaris App Bridge context and Polaris Web Components may not render correctly in fixed-position overlays inside the Shopify Admin iframe.
- `zIndex: 9998` keeps it below Shopify Admin modals (z-index 9999+).
- Button hidden on `/app/growth-call` to avoid visual redundancy.
- **Do NOT add `/app/growth-call` to `<s-app-nav>`** — hidden page, access only via the floating button.

**Verify:**
```bash
npm run typecheck 2>&1 | grep -E "error" | head -20
```

---

## Step 4 — Create the growth-call route

Create `app/routes/app.growth-call.tsx`. Flat file route → maps to `/app/growth-call`, child of `app.tsx` layout.

**Key constraints:**
- No loader — reads parent data via `useRouteLoaderData("routes/app")`
- Import `BookingWidget` and its CSS here (not in `app.tsx`)
- Polaris Web Components only for layout (`s-page`, `s-section`, `s-stack`, `s-grid`, `s-paragraph`, `s-icon`)
- Inline `BookingWidget` with `trigger="button"` (not floating/fixed)
- **Page content (BENEFITS, headings, copy) is identical across all Gropulse apps — copy verbatim**
- Only `appName` in the context object changes per app (it's passed to the AI, not displayed)

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
              appName: "GroPulse Redirect Manager",
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

**Change `appName` to match the specific app** — it's injected into the AI system prompt only, not shown on the page.

**Verify:**
```bash
npm run typecheck 2>&1 | grep -E "error" | head -20
npm run build 2>&1 | tail -20
```

---

## Step 5 — Optional env var

Add `BOOKING_API_URL` to `.env.example`:

```bash
BOOKING_API_URL=
# Leave blank — falls back to https://booking.gropulse.com
# Set to http://localhost:8787 only when running the booking API locally
```

---

## Step 6 — Build and smoke test

```bash
npm run build
npm run dev
```

Manual checks:
1. Open any app page → floating "Book Free Strategy Call" button visible bottom-left
2. Hover button → slight lift + opacity change
3. Click button → navigates to `/app/growth-call`
4. On growth-call page → floating button is hidden
5. "Book Free Strategy Call" button in page → clicking opens the booking chat widget
6. Chat progresses → AI asks context questions → slots appear → booking can be confirmed

---

## Widget API reference

### `BookingWidget` props

```ts
interface BookingWidgetProps {
  apiBaseUrl: string;          // "https://booking.gropulse.com"
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
  appName: string;          // REQUIRED — injected into AI system prompt (change per app)
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

1. **Change `appName`** in the `context` object — injected into AI system prompt only
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
| Floating button shows on growth-call page | `pathname` check failing | Verify `location.pathname === "/app/growth-call"` — check for trailing slash |
| Widget styles broken | CSS not imported | Confirm `import "@gropulse/booking-widget/styles.css"` in `app.growth-call.tsx` |
| `useRouteLoaderData` returns `undefined` | Route ID string wrong | Must be exactly `"routes/app"` — matches `app/routes/app.tsx` |

---

## Files changed summary

| File | Change |
|------|--------|
| `package.json` | Add `@gropulse/booking-widget` dependency |
| `app/routes/app.tsx` | Add 6 loader fields; add `useLocation`/`useNavigate`; add floating CTA button |
| `app/routes/app.growth-call.tsx` | New file — growth strategy call landing page |
| `.env.example` | Add `BOOKING_API_URL=` (optional) |
