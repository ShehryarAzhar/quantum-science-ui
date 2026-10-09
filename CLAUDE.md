# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## Commands

npm is the package manager (`package-lock.json`).

- `npm run dev` — dev server at http://localhost:3000
- `npm run build` — production build (also the only full type check wired into a script)
- `npm run lint` — ESLint over the repo (flat config, `eslint.config.mjs`); lint one file with `npx eslint app/page.tsx`
- `npx tsc --noEmit` — type check without building
- `npx shadcn@latest add <component>` — add a UI component into `components/ui/`

There is no test runner configured.

## Stack

Next.js 16.3 (App Router) · React 19.2 · TypeScript (strict) · Tailwind CSS v4 · shadcn/ui on Base UI.

The project is a fresh scaffold: `app/layout.tsx`, a placeholder `app/page.tsx`, and one UI component (`components/ui/button.tsx`). `next.config.ts` is empty. The metadata in `app/layout.tsx` is still the scaffold's placeholder; the `foundation` feature replaces it (see [SEO](docs/seo.md)). No page from the [route plan](#implemented-vs-stub-pages) is built yet.

## Conventions that differ from the usual defaults

**No `src/` directory.** `app/`, `components/`, and `lib/` sit at the repo root, and the `@/*` path alias maps to the root (`@/components/ui/button`, `@/lib/utils`). The site's copy will live in `content/`, also at the root (`@/content/faq`); see [SEO](docs/seo.md).

**shadcn uses the `base-vega` style, so primitives come from `@base-ui/react`, not Radix.** Base UI's API differs: composition uses the `render` prop rather than `asChild`, and prop types are `Primitive.Props` (see `components/ui/button.tsx`). Don't add `@radix-ui/*` packages or copy Radix-flavoured shadcn snippets; generate components through the shadcn CLI so they match `components.json`. A component from a third-party shadcn registry is acceptable only when its `base-vega` build depends on these Base UI primitives; the one planned is ReUI's phone input (see [Phone numbers](docs/api-contract.md#phone-numbers)).

**`cn` comes from the `cn` npm package** (shadcn's compiled replacement for `clsx` + `tailwind-merge`), neither of which is installed. `lib/utils.ts` just re-exports it; generated components import from `"cn"` directly, app code uses `@/lib/utils`.

**Tailwind v4 is configured entirely in CSS.** There is no `tailwind.config.*`; `app/globals.css` holds the imports (`tailwindcss`, `tw-animate-css`, `shadcn/tailwind.css`), the `@theme inline` token mapping, and the design tokens as OKLCH CSS variables under `:root` and `.dark`. To add or change a colour/radius, edit the variable there and (for new tokens) map it in `@theme inline`. Use semantic utilities (`bg-primary`, `text-muted-foreground`, `border-border`) rather than raw palette colours.

**Dark mode is class-based** (`@custom-variant dark (&:is(.dark *))`) — it activates when an ancestor has the `.dark` class. Nothing toggles that class yet; there is no theme provider.

**Fonts** are set in `app/layout.tsx` via `next/font/google`: Inter is bound to `--font-sans` (body and headings), Geist Mono to `--font-geist-mono` (`font-mono`). Geist Sans is loaded but its variable is not referenced by the theme.

**Route prop types are generated globals.** `layout.tsx` uses `LayoutProps<"/">` without an import; these helpers (`LayoutProps`, `PageProps`) are emitted into `.next/types` by `next dev` / `next build` / `next typegen`, so a fresh checkout shows type errors on them until one of those has run.

**Icons:** `lucide-react`.

## Project

This is the frontend for **Science Nest**, a science tutoring web application. It is frontend only: a separate Django REST Framework backend provides the API and this app consumes it over HTTP. There is no database and no business logic here; the only server-side code is the thin auth/proxy layer described under [Auth and data access](docs/auth.md).

Students use this app to:

1. Browse subjects, the levels they are taught at, and their prices.
2. Book weekly recurring classes, each for a subject at a chosen level.
3. Book a free trial lesson for a subject at a chosen level.
4. View their schedule and total weekly cost.

Teacher and admin work (managing subjects, viewing all students' bookings) happens in the Django admin site. **Do not build any teacher or admin pages.**

Libraries already installed for this: `axios` (HTTP), `@tanstack/react-query` (client-side fetching, caching, and mutations), `react-hook-form` + `zod` + `@hookform/resolvers` (forms and validation). Planned but not installed: `react-phone-number-input` and `libphonenumber-js`, added in the `auth` feature (see [Phone numbers](docs/api-contract.md#phone-numbers)).

## Environment

| Variable  | Where it is read | Purpose |
| --------- | ---------------- | ------- |
| `API_URL` | Server only      | Base URL of the Django API, no trailing slash (e.g. `http://localhost:8000`) |
| `SITE_URL` | Server only     | This site's canonical origin, no trailing slash (e.g. `http://localhost:3000`, `https://www.example.com` in production). Used for `metadataBase`, canonical URLs, the sitemap, `robots.txt` and structured data |
| `ALLOW_INDEXING` | Server only | `true` only on the production deployment. Anything else keeps the whole site `noindex`, so previews and staging are never indexed |
| `GOOGLE_SITE_VERIFICATION` | Server only | Optional. Google Search Console verification token |
| `BING_SITE_VERIFICATION` | Server only | Optional. Bing Webmaster Tools verification token |

Copy `.env.example` to `.env.local`. `.gitignore` ignores `.env*` with an explicit `!.env.example` exception, so only the example file is committed.

`API_URL` is deliberately **not** `NEXT_PUBLIC_`: the browser never talks to Django directly (see below), so the URL stays out of the client bundle and can change without a rebuild. Don't introduce a `NEXT_PUBLIC_` copy of it.

## Topic files

The detailed rules live in these files, imported here so they are always loaded:

@docs/api-contract.md
@docs/auth.md
@docs/code-organization.md
@docs/design.md
@docs/seo.md

## Testing

**Do not write tests for this frontend and do not add a test runner.** Verify work with:

```
npm run lint
npx tsc --noEmit
npm run build
```

## Feature workflow

Each feature goes through the same three steps:

1. `/create-spec <feature>` — needs a clean working tree. It branches `feature/<feature>` off an up-to-date `main` and writes the feature's spec file, asking about anything this file leaves open (API field names in particular). Build features in the order of the table below: later specs pick up rules that earlier ones deferred.
2. Implement from the spec, in Plan Mode. Flip the page statuses in [Implemented vs Stub Pages](#implemented-vs-stub-pages) in the same change.
3. Verify with the commands under [Testing](#testing).

| Feature | Spec file | Scope |
| ------- | --------- | ----- |
| `foundation` | `.claude/specs/01-foundation.md` | Route-group layouts, nav and footer, providers, axios instances, data-access module, `/api/…` proxy, `proxy.ts`, global `not-found` / `error`, the shared "not a student account" notice, the SEO base (root metadata, `robots.ts`, `sitemap.ts`, site share image, JSON-LD component, `noindex` on `(auth)` and `(app)`) |
| `subjects` | `.claude/specs/02-subjects.md` | `/`, `/subjects`, `/subjects/[slug]` |
| `auth` | `.claude/specs/03-auth.md` | `/login`, `/register`, `/forgot-password`, `/reset-password/[uid]/[token]`, logout |
| `schedule` | `.claude/specs/04-schedule.md` | `/schedule` |
| `weekly-classes` | `.claude/specs/05-weekly-classes.md` | `/classes/new`, `/classes/[id]/edit` |
| `trial-lesson` | `.claude/specs/06-trial-lesson.md` | `/trial-lesson` |
| `account` | `.claude/specs/07-account.md` | `/account` |
| `content-pages` | `.claude/specs/08-content-pages.md` | `/how-it-works`, `/faq`, `/about`, and their links in the nav and footer |

A spec is the detailed contract for one feature and records the decisions made on ambiguous behaviour. This file is the source a spec is written from: if a spec and this file contradict each other, stop and ask; do not pick one.

`.claude/commands/create-spec.md` carries its own copy of this table and refers to the sections of this file and of the files it imports by heading name. When a feature, a heading or a rule changes here, update the command in the same change.

## Implemented vs Stub Pages

Routes live in three route groups, each with its own layout: `(marketing)` (public), `(auth)` (guest only), and `(app)` (logged-in students). Only `(marketing)` pages are indexed by search engines; the other two groups are `noindex` (see [SEO](docs/seo.md)). **Keep this table updated as pages are built** — flip the status in the same change that implements the page.

| Path | Group | Purpose | Backend endpoints | Status |
| ---- | ----- | ------- | ----------------- | ------ |
| `/` | `(marketing)` | Landing: pitch, preview of subjects with their levels and prices, calls to action for the free trial | `GET /subjects/` | Stub |
| `/subjects` | `(marketing)` | Browse all subjects with their levels and 40- and 60-minute prices | `GET /subjects/` | Stub |
| `/subjects/[slug]` | `(marketing)` | Subject detail (description, levels, prices) with "book weekly class" and "book trial" actions | `GET /subjects/`, `GET /subjects/{slug}/` | Stub |
| `/how-it-works` | `(marketing)` | How online classes, booking and the free trial work | — | Stub |
| `/faq` | `(marketing)` | Questions students and parents ask, with `FAQPage` structured data | — | Stub |
| `/about` | `(marketing)` | Who the tutor is and their qualifications, from facts the user supplies | — | Stub |
| `/login` | `(auth)` | Sign in | `POST /auth/jwt/create/`, `GET /auth/users/me/` | Stub |
| `/register` | `(auth)` | Create an account (username, email, password, first and last name, phone number with country selector), then sign in automatically | `POST /auth/users/`, `POST /auth/jwt/create/` | Stub |
| `/forgot-password` | `(auth)` | Request a password-reset email; shows the same message whether or not the email exists, and handles the 429 rate limit | `POST /auth/users/reset_password/` | Stub |
| `/reset-password/[uid]/[token]` | `(auth)` | Set a new password from the emailed link, then sign in again at `/login`. Open to everyone, not guest only | `POST /auth/users/reset_password_confirm/` | Stub |
| `/schedule` | `(app)` | Home after login: weekly timetable sorted by local day and time, trial lesson, total weekly cost | `GET /schedule/` | Stub |
| `/classes/new` | `(app)` | Book a weekly class (subject, level, weekday, full hour, 40 or 60 minutes) | `GET /subjects/`, `POST /classes/` | Stub |
| `/classes/[id]/edit` | `(app)` | Change (subject, level, weekday, hour, duration) or cancel a weekly class | `GET /subjects/`, `GET` / `PATCH` / `DELETE /classes/{id}/` | Stub |
| `/trial-lesson` | `(app)` | Book the one free trial (subject, level, future date and time); if it exists, show it with edit/delete while unlocked, read-only once locked | `GET /subjects/`, `GET` / `POST /trial-lessons/`, `GET` / `PATCH` / `DELETE /trial-lessons/{id}/` | Stub |
| `/account` | `(app)` | Edit email, first and last name; show username and the formatted phone number read-only; change password; delete account | `GET` / `PATCH` / `DELETE /auth/users/me/`, `POST /auth/users/set_password/`, `POST /auth/jwt/create/` | Stub |

Not pages, but part of the plan: `proxy.ts` (route protection and token refresh), the `/api/…` proxy Route Handler, global `not-found` / `error` UI, the shared "not a student account" notice, and the SEO files (`robots.ts`, `sitemap.ts`, the share images, the JSON-LD component). `POST /auth/jwt/refresh/` is used by the auth layer, not by a page; `/auth/jwt/verify/` is not needed.

The reset path matches the backend's Djoser `PASSWORD_RESET_CONFIRM_URL`, `reset-password/{uid}/{token}`; if the backend ever changes that pattern, change the route here to match rather than the other way round.
