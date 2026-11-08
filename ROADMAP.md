# Roadmap

A web app containing a collection of tools that help sellers target small businesses. The home page is a tool selector. Each tool opens to its own homepage where the seller does their work.

This roadmap currently covers the foundation, the shell, backend integration, and **Tool 1: Zone Map & Business Discovery**. Later tools and the hardening/launch phases will be added once Tool 1 is validated.

## Stack

| Concern | Choice |
|---|---|
| Build | Vite |
| UI | React + TypeScript |
| Package manager | pnpm |
| Routing | TanStack Router (file-based) |
| Server state | TanStack Query |
| Styling / components | Tailwind CSS + shadcn/ui |
| Forms / validation | React Hook Form + Zod |
| Map rendering | MapLibre GL JS via `react-map-gl` (deck.gl `H3HexagonLayer` is the fallback if cell counts get large) |
| H3 | `h3-js` |
| Auth / RBAC | Existing backend using better-auth (frontend uses the better-auth client) |
| Testing | Vitest + Testing Library, Playwright for key flows |
| Hosting | Netlify (static SPA, deploy previews per PR) |

## Goals

- Ship Tool 1 end to end, then validate it with real sellers before building more tools.
- Make adding a new tool cheap: a new folder plus one registry entry.
- Keep verification to a single command (`pnpm check`) so AI-assisted changes are easy to validate.

## Non-goals (for now)

- Tools 2-5
- Building auth or RBAC (already exists in the backend)
- Public marketing pages or SEO
- Native mobile apps
- Hardening and launch phases (to be added later)

## Estimates

One developer, AI-assisted ("vibe coded"), frontend scope only. Backend endpoints are treated as dependencies (see Phase 3 contracts).

| Phase | Hours |
|---|---|
| 0. Foundation (includes Netlify setup) | 3-8 |
| 1. App shell and tool selector | 4-10 |
| 2. Backend integration | 4-10 |
| 3. Tool 1: Zone Map & Business Discovery | 60-130 |
| **Total** | **~70-160, plus 25-30% buffer** |

---

## Phase 0: Foundation

**Goal:** a clean, verifiable project skeleton.

- [x] Scaffold with Vite + React + TypeScript
- [x] pnpm as package manager
- [ ] TanStack Router with file-based routing (router plugin before the React plugin in `vite.config.ts`)
- [ ] TanStack Query with the `QueryClient` in router context
- [ ] Tailwind CSS (v4 via `@tailwindcss/vite`)
- [ ] shadcn/ui initialized with base components
- [ ] Path alias (`@/`) in `tsconfig` and Vite
- [ ] Lint/format (Biome or ESLint + Prettier)
- [ ] Vitest + Testing Library with one passing test
- [ ] `pnpm check` script (typecheck, lint, tests)
- [ ] `CLAUDE.md` with stack, folder conventions, and commands
- [ ] CI running `pnpm check` on every PR
- [ ] Netlify site connected to the Git repo
- [ ] `netlify.toml` with build command `pnpm build`, publish directory `dist`, and a pinned Node version
- [ ] SPA fallback redirect (`/*` → `/index.html`, status 200) so deep links to TanStack Router routes work on refresh
- [ ] Deploy Previews enabled for every PR; production deploys from `main`
- [ ] Environment variables set per context (production, deploy preview, branch deploy); only public values (`VITE_*`) in the client
- [ ] Confirm Netlify uses pnpm (lockfile detection or `packageManager` field in `package.json`)

**Done when:** a fresh clone runs `pnpm install && pnpm dev`, `pnpm check` passes in CI, and a PR produces a working Netlify deploy preview where refreshing a nested route does not 404.

## Phase 1: App shell and tool selector

**Goal:** the navigation skeleton Tool 1 plugs into.

- [ ] Root layout (header, user menu, main content area)
- [ ] Tool registry: typed list of tools (id, name, description, icon, route, required role)
- [ ] Home page: grid of tool cards generated from the registry
- [ ] Per-tool layout route (`/tools/$toolId`) with shared header and "back to all tools"
- [ ] 404 and error boundary pages
- [ ] Shared loading and empty-state components

**Suggested structure**

```
src/
  routes/
    __root.tsx
    index.tsx                  # tool selector
    tools/
      route.tsx                # shared tool layout
      zone-map/index.tsx       # Tool 1 homepage
  tools/
    registry.ts
    zone-map/                  # components, hooks, schemas, tests for Tool 1
  components/                  # shared UI (including shadcn/ui)
  lib/                         # API client, auth client, utilities
```

**Done when:** Tool 1 appears on the home page as a card and opens its (placeholder) homepage.

## Phase 2: Backend integration

**Goal:** connect to the existing backend. Auth and RBAC are not rebuilt here.

- [ ] better-auth client configured (base URL, session handling)
- [ ] Sign in / sign out screens wired to the backend
- [ ] Protected routes via router `beforeLoad` (redirect when no session)
- [ ] Role helper (`useCan('permission')`) to show/hide UI by role; the backend remains the source of truth for enforcement
- [ ] Typed API client with Zod-validated responses and consistent error handling
- [ ] Environment config (API base URL, map tile key; public values only in the client)
- [ ] API origin strategy that works on Netlify production and Deploy Previews: either a Netlify proxy rewrite (e.g. `/api/*` → backend, keeping requests same-origin so session cookies work) or CORS with credentials plus better-auth `trustedOrigins` covering the production domain and preview URLs
- [ ] Verify sign-in, session persistence, and sign-out on a deploy preview, not just locally

**Done when:** a signed-out user is redirected to sign in, and a signed-in user can load the home page with their role available in the UI.

---

## Phase 3: Tool 1 — Zone Map & Business Discovery

**Goal:** sellers define H3-based zones inside service areas, run an AI agent to discover small businesses in a zone or cell, review and merge the results, and export them.

### Core concepts

- **Service area:** a named group of zones.
- **Zone:** a named set of H3 cells belonging to exactly one service area.
- **Business:** a discovered small business with extracted details and a status.
- **Search run:** one agent execution for a zone/cell and a `business_type` string.
- **Review item:** a merge decision the user must make (conflict or newly available info).

### Proposed data model (to confirm with backend)

| Entity | Key fields |
|---|---|
| ServiceArea | id, name |
| Zone | id, serviceAreaId, name, h3Resolution, cells (string[]) |
| Business | id, zoneId, h3Cell, name, businessType, address, lat/lng, phone, hours, menu, owners, contacts, status, verifiedBy, verifiedAt, sources |
| SearchRun | id, zoneId, cells, businessType, state, startedAt, finishedAt |
| ReviewItem | id, searchRunId, existingBusinessId, incoming, fieldDiffs, resolution |

### Status rules

| Status | Meaning | Set by |
|---|---|---|
| `verified` | Human reviewed | Human only |
| `complete` | No missing info, still needs verification | Derived from required fields |
| `incomplete` | Missing info, still needs verification | Derived from required fields |

Pin colors must not be the only signal (add icon or shape variants for accessibility).

### Dedupe and merge rules

For each incoming business from an agent run:

1. **No match found** → add as a new business.
2. **Match found, every field identical** → silently omit the incoming result.
3. **Match found, any field conflicts** → create a review item; the user chooses which value wins.
4. **Match found, existing entry was missing info that the incoming result provides** → create a review item so the user can accept or reject the new info.

Matching strategy (to confirm): normalized name + address, phone number, and geographic proximity.

### Milestones

#### 3.1 Spec and contracts (3-6 h)

- [ ] One-page spec for Tool 1 (user, flows, success criteria)
- [ ] Confirm the data model and API contract with the backend (zones, service areas, businesses, search runs, review items, export)
- [ ] Define "required fields" for `complete` vs `incomplete`
- [ ] Decide H3 resolution(s) and whether a cell may belong to multiple zones
- [ ] Decide how the agent job reports progress (polling vs SSE) and how it is cancelled

#### 3.2 Map foundation (4-8 h)

- [ ] Tool route with full-height map layout and side panel
- [ ] MapLibre map with chosen tile provider
- [ ] H3 grid overlay at the current zoom (cells derived from the viewport)
- [ ] Hover and click on cells; multi-select cells (click, shift-click, box select)

#### 3.3 Service areas and zones (12-24 h)

- [ ] Service area list with create, rename, delete
- [ ] Create a zone from selected cells, or by drawing a polygon converted with `polygonToCells`
- [ ] Name a zone and assign it to a service area
- [ ] Display zones with fills, outlines (`cellsToMultiPolygon`), and labels, colored per zone
- [ ] Edit a zone: add/remove cells, rename, move to another service area, delete
- [ ] Prevent or resolve cell overlap between zones (per the decision in 3.1)
- [ ] Zone/service area filter and search in the side panel
- [ ] Persist through the backend; role-gate create/edit/delete

#### 3.4 Business pins and detail view (8-16 h)

- [ ] Render business pins with clustering at low zoom
- [ ] Color-coded (plus icon/shape) pins by status
- [ ] Show/hide pins: global toggle, per-status toggles, per-zone toggle
- [ ] Business detail panel: all extracted fields, sources, status
- [ ] Manual edit of a business and a "Mark verified" action (role-gated, records who and when)
- [ ] Table/list view synced with the map selection

#### 3.5 Agent search (10-22 h)

- [ ] Select a zone or one or more cells as the search target
- [ ] `business_type` input with recent/suggested values
- [ ] Start a search run; show progress and state (queued, running, done, failed, cancelled)
- [ ] Cancel a run; handle errors and partial results
- [ ] Filter returned businesses to the selected cells (`latLngToCell` point-in-cell check)
- [ ] Extracted fields displayed: business hours, menu, owners, address, phone, contact information
- [ ] Search history per zone

#### 3.6 Dedupe and review queue (14-30 h)

- [ ] Apply matching and the four dedupe rules (backend or frontend, per the contract in 3.1)
- [ ] Review queue UI: lists items needing a decision, grouped by search run
- [ ] Conflict view: side-by-side existing vs. incoming, choose a winner per field (and a "keep all existing / take all incoming" shortcut)
- [ ] Missing-info view: show new values that fill gaps on an existing entry, accept or reject
- [ ] Recompute status after each decision
- [ ] Unresolved items persist between sessions; badge/count in the UI
- [ ] Unit tests for matching and merge logic (this is the highest-risk logic in the tool)

#### 3.7 Export (3-8 h)

- [ ] Export from zone, service area, or current filter
- [ ] Filter by status before export
- [ ] Formats: CSV and GeoJSON (JSON/XLSX optional)
- [ ] Role-gated; consider logging exports

#### 3.8 Tests and polish for Tool 1 (6-14 h)

- [ ] Unit tests for H3 helpers, status derivation, and merge logic
- [ ] Playwright: create zone, run a search (mocked), resolve a conflict, export
- [ ] Loading, empty, and error states across the tool
- [ ] Performance check with large zones (thousands of cells) and many pins
- [ ] Keyboard and screen-reader basics for panels and review queue
- [ ] Review with at least one real seller

**Done when:** a seller can create a service area and zone, run a search for a business type, review conflicts and new info, mark businesses verified, and export the results.

---

## Open decisions

- [ ] **Where the agent runs:** assumed to be the backend (the frontend never holds LLM keys). Does the endpoint exist, or does it also need to be built? Backend work is not in the estimates above.
- [ ] **Required fields** for `complete` vs `incomplete`
- [ ] **Incoming result missing data the existing entry has:** treat as "no new information" and omit, or surface to the user? (Rule 2 says omit only if all info is identical, so this edge needs an explicit call.)
- [ ] **Verified businesses:** are they ever changed by a merge? If the user accepts a change, does the status revert to `complete`/`incomplete` and need re-verification?
- [ ] **H3 resolution** and whether a cell can belong to more than one zone
- [ ] **Map tile provider** (MapTiler, Protomaps, Stadia, etc.) and cost limits
- [ ] **Export formats** and who is allowed to export
- [ ] **Roles/permissions** for creating zones, running searches, verifying, and exporting
- [ ] **API origin strategy on Netlify:** proxy rewrite (same-origin) vs. cross-origin with CORS and cookie settings. This also decides how better-auth `trustedOrigins` handles deploy preview URLs.
- [ ] **Netlify plan limits** (build minutes, bandwidth, team seats) vs. expected usage

## Risks

| Risk | Mitigation |
|---|---|
| LLM-extracted data is wrong or hallucinated | Store sources per field, never auto-mark `verified`, keep humans in the review loop |
| Dedupe false positives/negatives | Conservative matching, unit tests with real-world messy data, make merges reversible |
| Agent runs are slow or expensive | Cap zone size per run, show progress and cancel, rate limit per user |
| Data sourcing/terms-of-service issues | Confirm allowed data sources with the backend owner before building on them |
| Large zones make the map sluggish | Viewport-based cell rendering, deck.gl fallback, pin clustering |
| API drift vs. AI knowledge of TanStack, react-map-gl, h3-js | Check current docs when generated code errors |
| Auth cookies/CORS break on Netlify Deploy Previews (each has a different origin) | Prefer a Netlify proxy rewrite to keep requests same-origin; otherwise allow preview origins in better-auth `trustedOrigins`; test sign-in on a preview early |
| SPA deep links 404 on refresh | SPA fallback redirect in `netlify.toml`, verified on every deploy preview |
| Vague requirements cause rework | Finish milestone 3.1 before building |

## Definition of done (for any task)

- `pnpm check` passes (typecheck, lint, tests)
- New behavior has a test or is covered by an existing one
- Loading, empty, and error states handled
- Reviewed by a human, not only generated
- `CLAUDE.md` updated if conventions changed
