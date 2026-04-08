# Scale Plan — RefBoard

A plan for evolving the current single-file whiteboard into a maintainable, multi-sport,
subscription-capable PWA. Written for a solo TS/React developer with no backend ops
overhead requirements. Prioritises simplicity and low running costs over enterprise
scalability.

---

## 0. Is It Worth It?

The plan below is technically correct for the stated ambitions — auth, subscriptions,
multi-sport, libraries, PWA, org accounts. But those ambitions together represent a full
product build, not a refactor. For a solo developer, on an exploratory project, on a
donation model, targeting a niche audience, that is likely 6–12 months of part-time work
before a single user saves a single scenario to the cloud. Meanwhile, the current app
already works. It runs on iPad with Apple Pencil. It loads instantly. It has zero
infrastructure costs and zero maintenance burden.

### What you actually need right now

| Problem | Blocked today? | Simplest fix |
|---|---|---|
| Scenarios lost on refresh | Probably yes | `localStorage` — one day, no backend |
| Can't install on iPad | Yes | Add a PWA manifest to `index.html` — one hour |
| Single file hard to maintain | Eventually | React rewrite — but only when the file is actually painful |
| Multi-sport | No users asking yet | Add another environment in the existing config |
| Cloud libraries | No users yet | Wait for demand |
| Subscriptions | No users yet | Wait for demand |

The first two problems can be solved this week without touching the architecture at all.

### What in this plan is genuinely worth doing

**The React/TypeScript rewrite** — yes, eventually. The single file will become a
maintenance problem around ~2500 lines and is already at ~2000. The hot/cold state
model, mode handler objects, and engine separation are good ideas that will pay off. But
this is an engineering quality-of-life improvement, not a user-facing feature.

**Supabase + auth + Stripe** — only once there are users asking to save things
persistently and willing to pay. Building a subscription system before you have
subscribers is the classic mistake.

**The sport config system** — cheap to do as part of the React rewrite; worth doing then.

### The right way to use this document

This plan is a valid target state. You do not need to reach it before the app has
traction. Treat Phases 1–2 (scaffold and modularise) as near-term work worth doing for
maintainability. Treat everything from Phase 3 onwards as contingent on having users who
justify it. The simplest next steps are: ship a PWA manifest, add `localStorage`
persistence, see if anyone uses it. If they do and they're asking to save things across
devices, add Supabase. If they're asking for team accounts, build org support then. Do
not build the org account system before an org asks for it.

---

## 1. Guiding Principles

- **Freely accessible by default.** The core whiteboard requires no account. Auth and
  payment are only touched when saving to a library or accessing premium features.
- **Solo-maintainable.** Every technology choice favours managed services and minimal
  operational surface area. No servers to provision or databases to manage.
- **The engine is not the UI.** Whiteboard logic must be decoupled from layout and
  visual design so that toolbar reorganisations and interaction experiments never require
  touching core rendering or event logic.
- **PWA, not native.** A well-configured PWA covers iPad + Apple Pencil support, home-
  screen installation, and offline use — without the complexity, cost, or App Store
  politics of native app development. Crucially, payments via Stripe apply in the browser;
  Apple cannot require in-app purchase (30% cut) from a PWA.
- **EU/UK data by default.** All infrastructure is configured to keep user data within
  the EU/EEA from day one.

---

## 2. Technology Stack

### Frontend
| Concern | Choice | Rationale |
|---|---|---|
| Language | TypeScript | Your primary stack |
| Framework | React 18 | Your primary stack |
| Build tool | Vite | Fast DX, zero config, PWA plugin available |
| Styling | Tailwind CSS | Utility-first; no component-library lock-in; pairs well with Radix UI primitives or shadcn/ui if desired |
| State management | Zustand | Minimal API, no boilerplate; replaces the current `state` object naturally |
| Routing | React Router v6 | Standard; simple file-based page structure |
| PWA | `vite-plugin-pwa` | Generates service worker + manifest; Workbox handles caching strategies |

### Backend (all-in-one)
| Concern | Choice | Rationale |
|---|---|---|
| Auth | Supabase Auth | Magic-link email + optional Google OAuth; no password management; integrates directly with database |
| Database | Supabase (PostgreSQL) | Fully managed, Row Level Security for user isolation, EU region available |
| File storage | Supabase Storage | For future custom uploads (e.g. custom field SVGs) |
| Serverless functions | Supabase Edge Functions | Deno-based; used only for Stripe webhook handling |
| Hosting | Vercel | Free tier generous; zero-config deploys from GitHub; handles PWA well |

### Payments
| Concern | Choice | Rationale |
|---|---|---|
| Subscriptions | Stripe Billing | Familiar to you; Stripe-hosted checkout page; customer portal for self-service cancellation |
| Donations | Stripe Payment Link | One-off, no subscription logic; shareable link; no code required |

**Running cost at zero users:** ~£0/month (Supabase free tier, Vercel free tier, Stripe
charges only on transactions). Costs only appear as you acquire paying users.

---

## 3. Architecture Overview

```
┌───────────────────────────────────────────────────────┐
│                    React UI Layer                     │
│  Toolbar · Panels · Library · Account · Modals        │
│  (layout, icons, interactions — no engine logic)      │
├───────────────────────────────────────────────────────┤
│                 Whiteboard Engine                     │
│  Canvas rendering · Token management · Mode handlers  │
│  History/undo · Snap · Field coord system             │
│  (sport-agnostic; no React, no Supabase)              │
├───────────────────────────────────────────────────────┤
│                  Sport Config Layer                   │
│  Field SVG · Token types · Dimensions · Snap res.     │
│  (pure data; adding a sport = adding a config file)   │
├───────────────────────────────────────────────────────┤
│                  Data / Auth Layer                    │
│  Supabase client · Scenario CRUD · Library CRUD       │
│  User profile · Subscription status                  │
└───────────────────────────────────────────────────────┘
```

The whiteboard engine has no knowledge of React, Supabase, or sport configs. It exposes
a small API that the React layer calls. This is the key architectural boundary that keeps
things maintainable: UI experiments never touch rendering logic, and engine refactors
never touch component structure.

---

## 4. Project File Structure

```
/
├── public/
│   ├── field-ifaf.svg
│   ├── field-flag.svg
│   ├── field-assoc.svg          ← future
│   ├── field-rugby.svg          ← future
│   ├── media/                   (icons, logo)
│   ├── fonts/
│   └── grass-tile.png
│
└── src/
    ├── engine/                  ← sport-agnostic whiteboard logic
    │   ├── canvas.ts            drawCone, redrawCanvas, drawShape
    │   ├── tokens.ts            placeToken, removeToken, repositionToken
    │   ├── coords.ts            measureField, fracToPx, pxToFrac
    │   ├── snap.ts              snapFrac
    │   ├── history.ts           pushHistory, undo, redo
    │   └── modes/               one file per interaction mode
    │       ├── types.ts         ModeHandler interface
    │       ├── select.ts
    │       ├── place.ts
    │       ├── draw.ts
    │       ├── erase.ts
    │       └── cone.ts
    │
    ├── sports/                  ← sport config data (no logic)
    │   ├── types.ts             SportConfig, Environment, TokenType interfaces
    │   ├── americanFootball.ts
    │   ├── associationFootball.ts   ← future
    │   └── rugby.ts                 ← future
    │
    ├── stores/                  ← Zustand state (replaces current state object)
    │   ├── boardStore.ts        tokens, strokes, cones, mode, history
    │   └── uiStore.ts           panel visibility, active sport, active environment
    │
    ├── components/
    │   ├── Board/               main whiteboard component (owns canvas refs)
    │   ├── Toolbar/             top bar, mode buttons, environment switcher
    │   ├── DrawPanel/           draw sub-panel (colour, width, tool)
    │   ├── LibraryPanel/        scenario browser, save/load
    │   ├── AccountPanel/        auth, subscription, profile
    │   └── ui/                  shared primitives (Button, Modal, etc.)
    │
    ├── lib/
    │   ├── supabase.ts          Supabase client singleton
    │   ├── stripe.ts            Stripe helpers (create checkout session, etc.)
    │   └── flags.ts             feature flags (see §9)
    │
    ├── routes/                  React Router pages
    │   ├── index.tsx            landing / redirect to /board
    │   ├── board.tsx            main whiteboard
    │   ├── library.tsx          library management
    │   └── account.tsx          subscription + profile
    │
    └── types/                   shared TypeScript interfaces
        ├── Scenario.ts
        ├── Token.ts
        ├── Stroke.ts
        └── Cone.ts
```

---

## 5. Sport Config System

Each sport is a TypeScript object implementing `SportConfig`. Adding a sport requires no
engine changes — only a new config file and a new field SVG.

```typescript
// src/sports/types.ts

interface TokenType {
  id: string;           // 'player-a' | 'player-b' | 'official' | 'ball' | 'flag' | 'ltg'
  label: string;        // display name
  colour: string;
  editableNumber: boolean;
  coneEligible: boolean;        // whether this token type can have a vision cone
  teamIndex: number | null;     // 0 = team A, 1 = team B, null = neutral (officials, ball)
  labels?: string[];            // fixed labels for officials (R, U, D, …)
}

interface Environment {
  id: string;               // 'ifaf' | 'flag' | '11v11' | '7s' etc.
  name: string;
  fieldSvg: string;
  fieldWidth: number;       // in `unit` below
  fieldHeight: number;      // in `unit` below
  unit: 'yards' | 'metres'; // American football = yards; assoc. football, rugby = metres
  fieldBoundary: { left: number; top: number; right: number; bottom: number }; // fractions
  snapResolution: number;   // snap increment, in `unit`
}

interface SportConfig {
  id: string;           // 'american-football' | 'association-football' | 'rugby'
  name: string;
  tokenTypes: TokenType[];
  environments: Environment[];
  defaultEnvironment: string;
}
```

The existing IFAF and Flag environments become two entries under `americanFootball`.
`fieldDimensions.widthYards` / `snapResolutionYards` are replaced by `fieldWidth` /
`fieldHeight` / `snapResolution` + `unit` so that metre-based sports are represented
correctly without storing metres in a field named "yards".

---

## 6. Rendering Architecture & State Tiers

Before mode handlers can be designed, the ownership model for rendering must be
explicitly defined. This is the most important architectural decision in the migration —
getting it wrong in Phase 1 means a rewrite in Phase 2.

### Ownership boundaries

| What | Owned by | How |
|---|---|---|
| Token divs | React | Absolutely-positioned components, rendered from Zustand cold state |
| Token position during active drag | Engine (ref) | Direct `el.style.left/top` mutation, bypassing React |
| Canvas drawing (strokes, cones) | Engine (imperative) | `redrawCanvas()` called via `useEffect` or directly from mode handlers via refs |
| Committed tokens, strokes, cones | Zustand (cold state) | Triggers React re-renders; written only on interaction commit |

React and the engine must never fight over the same DOM node. The contract is: React
renders the initial position of a token from Zustand; the engine moves it imperatively
during drag; on pointer up the engine commits the final position back to Zustand and
React reconciles (the pixel position already matches, so no visible jump occurs).

Token labels remain `contenteditable` spans inside React-rendered divs — this is
preserved exactly as today.

### Two-tier state

Routing `pointermove` through Zustand would trigger 60 React re-renders per second.
Instead, state is split by whether it needs to cause a React re-render:

**Hot state** — `useRef` or module-level mutable variable. Never triggers re-renders.
Used for: active drag position, stroke points being captured, ghost shape preview,
`penDetected`, `dragging`, `coneStep`.

**Cold state** — Zustand store. Triggers re-renders. Used for: committed tokens,
committed strokes, committed cones, history snapshots, current mode, draw settings.

Canvas `redrawCanvas()` reads from both tiers simultaneously:

```typescript
// In Board component
const committedStrokes = useBoardStore(s => s.strokes);
const activeStrokeRef = useRef<Stroke | null>(null);  // hot

// During pointermove — no Zustand involved, no re-render:
activeStrokeRef.current?.points.push(pt);
redrawCanvas(committedStrokes, activeStrokeRef.current);

// On pointerup — commit to cold state, triggers re-render:
useBoardStore.getState().commitStroke(activeStrokeRef.current);
activeStrokeRef.current = null;
```

Token drag follows the same pattern:

```typescript
const dragRef = useRef<{ tokenEl: HTMLElement; xFrac: number; yFrac: number } | null>(null);

// pointermove: dragRef.current.tokenEl.style.left = `${px}px`  ← direct DOM, no React
// pointerup:   boardStore.getState().updateToken(id, xFrac, yFrac)  ← React re-renders
```

This is structurally identical to the current app's behaviour. The migration makes it
explicit and typed rather than discovering it by accident.

The hot/cold split must be defined before any Zustand store is wired up — it determines
what goes in the store and what stays in refs.

### Mode Handler Architecture

With the state tiers established, mode handlers are objects that receive both tiers:

```typescript
// src/engine/modes/types.ts
// No React import — the engine has no framework dependency.

/** Framework-agnostic ref container. Structurally identical to React.MutableRefObject<T>. */
interface Ref<T> { current: T }

interface BoardRefs {
  canvasRefs: CanvasRefs;
  activeStrokeRef: Ref<Stroke | null>;
  dragRef: Ref<DragState | null>;
  ghostShapeRef: Ref<Stroke | null>;
  fieldRectRef: Ref<DOMRect>;   // ref, not value — always current even after resize
}

interface ModeHandler {
  onPointerDown(e: PointerEvent, refs: BoardRefs): void;
  onPointerMove(e: PointerEvent, refs: BoardRefs): void;
  onPointerUp(e: PointerEvent, refs: BoardRefs): void;
  onActivate?(refs: BoardRefs): void;
  onDeactivate?(refs: BoardRefs): void;
}
```

`fieldRectRef` is a ref rather than a `DOMRect` value. Mode handlers read
`refs.fieldRectRef.current` on every event, so coordinates remain correct even if the
user resizes the window mid-interaction. The `Board` component updates `fieldRectRef`
in its `ResizeObserver` callback.

Mode handlers read cold state via `useBoardStore.getState()` (outside React's render
cycle, no subscription) and write to cold state via the same. They mutate hot refs
freely. They never call React hooks — keeping them framework-agnostic and testable
without a React renderer.

**Mode in two places.** The current mode lives as both a hot ref and cold Zustand state.
The hot ref (`currentModeRef: Ref<ModeHandler>`) is what pointer event handlers use —
swapping it has zero rendering cost. The cold Zustand state (`boardStore.mode: ModeId`)
is what the toolbar reads to show which button is active. When `setMode()` is called, it
updates both atomically. They must never diverge.

The `Board` component holds `currentModeRef`. Switching mode = swapping the handler and
updating `boardStore.mode`. Experimenting with a mode = creating an alternate handler
and toggling via feature flag (see §12). Known interaction subtleties are encapsulated
per mode:

- **Apple Pencil jitter on contact** — `DRAG_THRESHOLD` (10px) stays in `select.ts`
- **iOS double-tap window suppression** — `touchstart preventDefault()` pattern stays
  in the place/cone handlers, documented in code comments
- **Pencil hover filtering** — `penDetected` hot ref stays in `draw.ts`
- **Safari first-drag-after-env-switch** — currently unresolved; isolated to `select.ts`
  and `coords.ts`, making it easier to instrument and fix without regression elsewhere

---

## 7. Database Schema

Hosted in Supabase, EU region (Frankfurt). Row Level Security (RLS) ensures every query
is automatically scoped to the authenticated user.

```sql
-- Managed by Supabase Auth; extended with a public profile view
create table public.profiles (
  id            uuid primary key references auth.users on delete cascade,
  subscription  text not null default 'free',  -- 'free' | 'premium'
  stripe_id     text unique                    -- Stripe customer ID
);

create table public.libraries (
  id         uuid primary key default gen_random_uuid(),
  user_id    uuid not null references public.profiles on delete cascade,
  name       text not null,
  sport_id   text not null,
  created_at timestamptz default now()
);

create table public.scenarios (
  id         uuid primary key default gen_random_uuid(),
  library_id uuid not null references public.libraries on delete cascade,
  user_id    uuid not null references public.profiles on delete cascade,
  name       text not null,
  tags       text[] default '{}',
  sport_id   text not null check (sport_id in ('american-football', 'association-football', 'rugby')),
  data       jsonb not null,    -- { schemaVersion, tokens, strokes, cones, environment, sport }
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Enforce free-tier limits in the database, not just the application layer.
-- Application checks are convenient but bypassable; this is not.
create or replace function check_scenario_limit()
returns trigger language plpgsql as $$
declare
  sub text;
  cnt int;
begin
  select subscription into sub from public.profiles where id = NEW.user_id;
  if sub = 'premium' then return NEW; end if;
  select count(*) into cnt from public.scenarios where user_id = NEW.user_id;
  if cnt >= 20 then
    raise exception 'Free tier limit reached (20 scenarios)';
  end if;
  return NEW;
end;
$$;
create trigger enforce_scenario_limit
  before insert on public.scenarios
  for each row execute function check_scenario_limit();

-- Same pattern for libraries (limit 3 on free tier).
```

**Scenario data versioning.** The `data` jsonb must always include `schemaVersion: number`
starting at `1`. The current codebase already has one implicit format change (the `type`
field on strokes, with `|| 'pencil'` fallback). A version field costs nothing to add now
and makes future migrations deterministic: load the scenario, check `schemaVersion`,
apply upgrade transforms in sequence, save back. Without it, format changes require
guessing the version from data shape or touching every saved row.

**`sport_id` constraint.** Both `libraries` and `scenarios` use a `CHECK` constraint on
`sport_id` rather than a free-text field. A typo during development cannot create
orphaned data. The constraint is updated via migration when a new sport is added.

**Library sport-scoping.** A `libraries.sport_id` column scopes each library to one sport.
This is intentional for initial release (it simplifies the UI and search), but officials
who cover multiple sports will need separate libraries. This decision should be revisited
once real usage patterns are known; the schema does not prevent adding cross-sport
libraries later.

Deleting a user cascades to all their data, satisfying GDPR right-to-erasure without
custom logic.

---

## 8. Auth

- **Supabase Auth** via `@supabase/supabase-js` v2 (the current package). The deprecated
  `@supabase/auth-helpers-react` package must not be used — it is unmaintained. For
  pre-built auth UI, use `@supabase/auth-ui-react` against the same client.
- **Google OAuth is enabled from day one**, not deferred. The primary target audience
  (officials at the FA, BAFRA, RFU) commonly uses corporate email systems with security
  gateways (Proofpoint, Mimecast, Microsoft Defender) that auto-click links in emails —
  consuming magic links before the user can. Magic link remains available as a fallback,
  but Google OAuth is the primary sign-in path for this audience.
- Auth is surfaced only when the user attempts to save a scenario or access the library
- Unauthenticated users get the full whiteboard; a soft prompt appears on first save
  attempt

---

## 9. Subscriptions & Payments

### Subscription flow

1. Unauthenticated or free-tier user hits a limit (e.g. tries to save a 4th library)
2. Upgrade prompt shown → user clicks "Go Premium"
3. App calls a Supabase Edge Function: `POST /functions/v1/create-checkout`
4. Edge Function creates a Stripe Checkout session (subscription, monthly or annual),
   passing `success_url=/account?upgraded=1` and `cancel_url=/account`
5. User completes payment on Stripe-hosted page; redirected back to `/account?upgraded=1`
6. `/account` detects `?upgraded=1` query param and enters a polling loop: re-fetch
   subscription status every 2 seconds for up to 30 seconds
7. Stripe fires `customer.subscription.updated` webhook (may arrive seconds to minutes
   after redirect)
8. Supabase Edge Function `POST /functions/v1/stripe-webhook` receives it, verifies
   signature, sets `profiles.subscription = 'premium'`
9. Polling loop detects the change; "Premium" status shown; limits lifted

Without step 6, the first thing every new paying user sees is their account still showing
"Free" — a broken first impression. The polling loop bridges the webhook delivery gap.

### Tiers

| Feature | Free | Premium |
|---|---|---|
| Full whiteboard (no account needed) | ✓ | ✓ |
| Save scenarios | Up to 20 | Unlimited |
| Libraries | Up to 3 | Unlimited |
| Tags & search | ✓ | ✓ |
| Sharing / collaboration | — | Future |
| All sports | ✓ | ✓ |

### Donations

A Stripe Payment Link is created once in the Stripe dashboard (no code). Link is placed
in the app footer and/or account page. No webhook handling required for donations.

### App Store note

As a PWA, the app is accessed via the browser. Apple's IAP requirement does not apply
to web apps accessed in Safari. This is a deliberate and meaningful reason to stay PWA
rather than packaging as a native app. If this ever changes (Apple's rules are not
static), Stripe + PWA remains the right architecture to avoid the 30% cut on what is
already a low-margin product.

---

## 10. PWA & Tablet Support

- `vite-plugin-pwa` generates `manifest.json` and a Workbox service worker automatically
  from Vite config
- **Caching strategy:** app shell (JS, CSS, fonts, icons) precached at build time;
  field SVGs precached; API calls (Supabase) use network-first with cache fallback
- **Offline behaviour:** full whiteboard works offline; library sync requires connectivity;
  a clear offline indicator is shown when sync is unavailable
- **iPad install:** user adds to home screen from Safari (manual step — iOS does not
  support install prompts, unlike Android/Chrome). Launches full-screen with no browser
  chrome. Functionally equivalent to a native app for this use case. A first-visit nudge
  in the UI ("tap Share → Add to Home Screen for the best experience") compensates for
  the missing prompt.
- **Apple Pencil:** already handled correctly via `PointerEvents` API. The existing
  patterns (jitter threshold, `penDetected` hover filtering, `touchstart preventDefault`
  for iOS double-tap window) must be preserved verbatim when porting to React
- **Responsive layout:** Tailwind breakpoints; toolbar collapses or reflows for narrow
  screens; panels slide in from the side on mobile/tablet

---

## 11. GDPR & Data Privacy

- **Data residency:** Supabase project in `eu-central-1` (Frankfurt) — user data never
  leaves the EU/EEA
- **Minimal collection:** email address + scenario data only; no analytics by default
- **Stripe:** handles all payment data; PCI-compliant; out of scope for personal data
  under GDPR (Stripe is a data processor, not a controller, for payment data)
- **Right to erasure:** cascade-delete on `profiles` table removes all user data
  (libraries, scenarios) in one operation; expose via "Delete account" in account page
- **Privacy policy:** `privacy.html` must be updated to describe: what is stored (email,
  scenarios), where it is stored (EU, Supabase), how to request deletion, and Stripe's
  role in payment processing
- **Analytics:** avoid until there is a clear need; if added later, use a privacy-first
  tool (e.g. Plausible, self-hosted or EU-hosted) with no cookie consent requirement
- **Cookie:** Supabase Auth uses `localStorage` by default (not a cookie); no cookie
  consent banner needed unless analytics are added

---

## 12. Feature Flags for Experimentation

A lightweight flags system lets you try alternate interaction behaviours (new toolbar
layout, revised cone creation flow, etc.) without branching the codebase.

The API must be designed for remote flags from day one. `flag(key)` is always
synchronous; async loading happens once at app init and writes into a module-level
cache. Call sites never change when the loading mechanism is upgraded.

```typescript
// src/lib/flags.ts

export type FlagKey =
  | 'coneEdgeVariantDefault'   // use edge variant by default instead of midline
  | 'newToolbarLayout'         // experimental toolbar reorganisation
  | 'drawPanelInline';         // embed draw options in toolbar vs sub-panel

const DEFAULTS: Record<FlagKey, boolean> = {
  coneEdgeVariantDefault: false,
  newToolbarLayout: false,
  drawPanelInline: false,
};

// Module-level cache — populated once at init, synchronously readable thereafter
let cache: Record<FlagKey, boolean> = { ...DEFAULTS };

/**
 * Called once in main.tsx before ReactDOM.render().
 * In dev: applies localStorage overrides (toggle via browser console).
 * In prod: fetches per-user overrides from Supabase when a userId is provided.
 * Falls back to DEFAULTS silently on any error.
 */
export async function loadFlags(userId?: string): Promise<void> {
  cache = { ...DEFAULTS };

  if (import.meta.env.DEV) {
    for (const key of Object.keys(DEFAULTS) as FlagKey[]) {
      const v = localStorage.getItem(`flag:${key}`);
      if (v !== null) cache[key] = v === 'true';
    }
    return;
  }

  if (userId) {
    try {
      const { data } = await supabase
        .from('feature_flags')
        .select('key, value')
        .eq('user_id', userId);
      data?.forEach(row => {
        if (row.key in DEFAULTS) cache[row.key as FlagKey] = row.value;
      });
    } catch {
      // Network failure — DEFAULTS remain in cache, app continues normally
    }
  }
}

/** Synchronous read — safe to call anywhere, including mode handlers and render. */
export const flag = (key: FlagKey): boolean => cache[key];
```

`loadFlags()` is called in `main.tsx` before rendering, and again after login (user
may have per-account flag overrides). The brief async gap before first render is covered
by the existing app-init loading state. All call sites remain `flag('key')` — they
never need to change.

---

## 13. Migration Path

Migration is incremental. At each phase the app is fully functional and deployable.
No big-bang rewrite.

### Phase 1 — Scaffold (visible change: none)
- Create Vite + React + TS project
- **Define the hot/cold state split before writing any store** — enumerate every field
  in the current `state` object and assign each to hot (ref) or cold (Zustand). This
  document is the contract for Phase 2 and must exist before any store is wired up.
- **Define the rendering ownership model** — token divs are React components rendered
  from cold state; canvas layers are imperative via refs. Document this explicitly in a
  `src/engine/README.md` so the boundary is never ambiguous.
- Port engine functions from `index.html` into `src/engine/` as typed modules. Logic
  is unchanged; only types are added and DOM references become parameters rather than
  globals.
- Single `Board` component mounts canvas refs and renders tokens as absolutely-
  positioned divs from a minimal local state (not yet Zustand). Token drag uses direct
  `el.style` mutation during drag; commits position on pointer up — matching current
  behaviour exactly.
- Add `flags.ts` with `loadFlags()` / `flag()` using the load-once pattern from §12.
  Local constants only at this stage.
- Add PWA manifest and basic service worker (app shell only, no API caching yet).
- **Parity gate:** verify on desktop and iPad that all existing interactions work
  identically to `index.html` — including Apple Pencil, drag threshold, double-tap
  detection, and cone creation — before proceeding to Phase 2.
- Deploy to Vercel; confirm PWA installs to iPad home screen.

### Phase 2 — Modularise (visible change: none)
- Using the hot/cold split defined in Phase 1, introduce Zustand stores:
  - `boardStore`: committed tokens, strokes, cones, history, current mode, draw settings
  - `uiStore`: panel visibility, active sport/environment
  - Hot state (drag, active stroke, ghost shape, `penDetected`) stays in refs owned by
    `Board` — it does not move to Zustand
- Implement mode handler objects (select, place, draw, erase, cone), each receiving
  both the canvas/DOM refs and reading/writing cold state via `boardStore.getState()`
- Extract sport configs for IFAF/Flag environments into `src/sports/`
- **Parity gate:** re-run full interaction test (desktop + iPad) before Phase 3. The
  Zustand introduction is the highest-risk step; catching regressions here is cheaper
  than discovering them after Supabase is wired up.

### Phase 3a — Persistence (no auth yet)
- Create Supabase project (EU region); set up schema, RLS, and DB-level tier limits
- Scenario save/load via Supabase using anonymous/service-role key — no user auth yet
- `schemaVersion: 1` written into all new scenario data from this point forward
- JSON export/import kept as fallback and tested against the new schema
- **Parity gate:** confirm saved scenarios round-trip correctly (save → reload → identical
  board state) before introducing auth

### Phase 3b — Auth & library panel
- Add Supabase Auth: Google OAuth (primary) + magic-link fallback
- Auth prompt on first save attempt; existing anonymous scenarios offered for migration
  into the user's account on first sign-in
- Library panel (list, create folder, name scenario, tag, search)
- RLS tightened so users can only read/write their own rows
- **Parity gate:** verify sign-in / sign-out / session refresh on both desktop Safari
  and Chrome before Phase 4

### Phase 4 — Subscriptions
- Stripe account + product + price configured in dashboard
- Stripe checkout + webhook via two Supabase Edge Functions
- Free-tier limits enforced in library panel
- Account page (subscription status, manage/cancel via Stripe portal, delete account)

### Phase 5 — PWA & tablet polish
- PWA manifest and app-shell caching were added in Phase 1; this phase extends the
  service worker to handle Supabase auth correctly (auth redirect URLs must not be
  intercepted by the service worker) and adds the offline data-caching strategy
- Responsive layout pass (Tailwind breakpoints)
- Apple Pencil regression test on iPad — verify all known interaction patterns
- PWA install prompt / "Add to home screen" nudge on iPad
- Update `privacy.html`

### Phase 6 — Additional sports
- Add `associationFootball` and/or `rugby` sport configs + field SVGs
- Validate engine handles new dimensions and token types without modification
- Environment switcher in UI extended to show sport selector above environment selector

---

## 14. Decisions Deferred

These are worth keeping in mind but do not need to be resolved now:

- **Sharing / collaboration** — sharing a library or scenario with another user (by email
  or public link) is architecturally straightforward (a `shares` table + RLS policy) but
  is deferred until the private library is working well.
- **Offline-first conflict resolution** — if a scenario is edited on iPad offline and
  then synced, a conflict strategy is needed. For now, last-write-wins is acceptable;
  local-first sync (e.g. via Supabase Realtime or a CRDT) is a future concern.
- **Custom field SVGs** — allowing users to upload their own field diagrams would use
  Supabase Storage; deferred until the sport config system is stable.
- **Organisation / team accounts** — an officiating body sharing a library across
  multiple users requires a `teams` table and team-scoped RLS. Not required yet but the
  schema should not make it hard to add.
- **Safari first-drag-after-env-switch bug** — the existing unresolved Safari regression
  is isolated to `select.ts` and `coords.ts` in the new structure, which makes it easier
  to instrument and fix without touching other mode handlers.
