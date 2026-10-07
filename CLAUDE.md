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

The project is a fresh scaffold: `app/layout.tsx`, a placeholder `app/page.tsx`, and one UI component (`components/ui/button.tsx`). `next.config.ts` is empty. No page from the [route plan](#implemented-vs-stub-pages) is built yet.

## Conventions that differ from the usual defaults

**No `src/` directory.** `app/`, `components/`, and `lib/` sit at the repo root, and the `@/*` path alias maps to the root (`@/components/ui/button`, `@/lib/utils`).

**shadcn uses the `base-vega` style, so primitives come from `@base-ui/react`, not Radix.** Base UI's API differs: composition uses the `render` prop rather than `asChild`, and prop types are `Primitive.Props` (see `components/ui/button.tsx`). Don't add `@radix-ui/*` packages or copy Radix-flavoured shadcn snippets; generate components through the shadcn CLI so they match `components.json`.

**`cn` comes from the `cn` npm package** (shadcn's compiled replacement for `clsx` + `tailwind-merge`), neither of which is installed. `lib/utils.ts` just re-exports it; generated components import from `"cn"` directly, app code uses `@/lib/utils`.

**Tailwind v4 is configured entirely in CSS.** There is no `tailwind.config.*`; `app/globals.css` holds the imports (`tailwindcss`, `tw-animate-css`, `shadcn/tailwind.css`), the `@theme inline` token mapping, and the design tokens as OKLCH CSS variables under `:root` and `.dark`. To add or change a colour/radius, edit the variable there and (for new tokens) map it in `@theme inline`. Use semantic utilities (`bg-primary`, `text-muted-foreground`, `border-border`) rather than raw palette colours.

**Dark mode is class-based** (`@custom-variant dark (&:is(.dark *))`) — it activates when an ancestor has the `.dark` class. Nothing toggles that class yet; there is no theme provider.

**Fonts** are set in `app/layout.tsx` via `next/font/google`: Inter is bound to `--font-sans` (body and headings), Geist Mono to `--font-geist-mono` (`font-mono`). Geist Sans is loaded but its variable is not referenced by the theme.

**Route prop types are generated globals.** `layout.tsx` uses `LayoutProps<"/">` without an import; these helpers (`LayoutProps`, `PageProps`) are emitted into `.next/types` by `next dev` / `next build` / `next typegen`, so a fresh checkout shows type errors on them until one of those has run.

**Icons:** `lucide-react`.

## Project

This is the frontend for **Quantum Science**, a science tutoring web application. It is frontend only: a separate Django REST Framework backend provides the API and this app consumes it over HTTP. There is no database and no business logic here; the only server-side code is the thin auth/proxy layer described under [Auth and data access](#auth-and-data-access).

Students use this app to:

1. Browse subjects and their prices.
2. Book weekly recurring classes.
3. Book a free trial lesson.
4. View their schedule and total weekly cost.

Teacher and admin work (managing subjects, viewing all students' bookings) happens in the Django admin site. **Do not build any teacher or admin pages.**

Libraries already installed for this: `axios` (HTTP), `@tanstack/react-query` (client-side fetching, caching, and mutations), `react-hook-form` + `zod` + `@hookform/resolvers` (forms and validation).

## Environment

| Variable  | Where it is read | Purpose |
| --------- | ---------------- | ------- |
| `API_URL` | Server only      | Base URL of the Django API, no trailing slash (e.g. `http://localhost:8000`) |

Copy `.env.example` to `.env.local`. `.gitignore` ignores `.env*` with an explicit `!.env.example` exception, so only the example file is committed.

`API_URL` is deliberately **not** `NEXT_PUBLIC_`: the browser never talks to Django directly (see below), so the URL stays out of the client bundle and can change without a rebuild. Don't introduce a `NEXT_PUBLIC_` copy of it.

## Backend API contract

All paths are relative to `API_URL`. Django requires the **trailing slash** on every path.

**Auth is Djoser + SimpleJWT. The `Authorization` prefix is `JWT`, not `Bearer`:** `Authorization: JWT <access_token>`.

Auth routes:

- `POST /auth/users/` — register (`username`, `email`, `password`, `first_name`, `last_name`)
- `GET` / `PATCH /auth/users/me/` — current user
- `POST /auth/jwt/create/` — log in with `username` + `password`; returns `access` and `refresh`
- `POST /auth/jwt/refresh/`, `POST /auth/jwt/verify/`
- `POST /auth/users/set_password/`, `/auth/users/reset_password/`, `/auth/users/reset_password_confirm/`

There is no email-activation step: registration is followed immediately by login.

Resources:

- **Subjects** (read-only, **public**, no auth needed): `GET /subjects/`, `GET /subjects/{id}/`. Each subject has two USD prices, one for a 40-minute class and one for a 60-minute class.
- **Weekly classes** (full CRUD, scoped to the logged-in student): `/classes/`, `/classes/{id}/`. A booking has a subject, a day of the week, a start time, and a duration of 40 or 60 minutes.
- **Trial lessons** (full CRUD, scoped to the logged-in student): `/trial-lessons/`, `/trial-lessons/{id}/`. A trial lesson has a subject and a specific date and time.
- **My schedule**: `GET /schedule/` — the student's weekly classes and trial lesson, plus their total cost per week.

Exact field names are not documented here; confirm them against the live API before typing a response, then define the type and zod schema once in `lib/` and reuse it.

### Money

Prices arrive as **decimal strings** (`"25.00"`). Never run them through `parseFloat`/`Number` for arithmetic. Prefer the backend's own total from `/schedule/`; if a client-side sum is unavoidable (e.g. a live preview while booking), convert to integer cents first. Format for display with `Intl.NumberFormat` (USD).

### Booking rules

The backend enforces these. The UI should guide users toward valid choices, but **the backend is the source of truth: always display its validation errors clearly** (field errors next to the field, `non_field_errors`/`detail` in a form-level alert) and never assume a client-side check is enough.

- Classes and trial lessons start only on the full hour (4:00, 5:00, 6:00…). Time pickers offer full hours only.
- A timeslot (weekday + hour) holds only one booking **across all students**. Taken slots are rejected. There is no availability endpoint, so the UI can only pre-disable slots the student already holds; a clash with another student surfaces as a backend error on submit.
- A student can book several weekly classes of the same subject.
- Trial lessons are 60 minutes and free. Each student gets **one** trial lesson. It can be edited or deleted only while it is not completed and its date and time have not passed; after that it is locked but still visible (read-only).
- A weekly class can't take a weekday + hour occupied by an upcoming trial lesson, and a trial lesson can't take a time occupied by a weekly class or another trial lesson.
- Weekly cost = the sum of each weekly class's price for its duration. Trial lessons add nothing.

### Timezones

The backend stores times in UTC; the UI shows and collects them in the **student's local timezone** and converts at the boundary. Keep all conversion in one helper module in `lib/` rather than scattering `Date` maths through components. Things to get right:

- A weekly slot is a weekday + hour, so converting can move it to the **previous or next weekday**. Convert the pair together, never the hour alone.
- "Full hour" is the backend's rule, in UTC. For students in a half-hour-offset zone (e.g. UTC+5:30) valid slots display as `:30`; offer the backend-valid slots converted to local time, not local full hours.
- A weekly slot fixed in UTC shifts by an hour locally across a DST change. Show the timezone next to times so this is not a surprise.
- Timezone-dependent output differs between server and browser. Render local times in Client Components (or after mount) to avoid hydration mismatches.

## Auth and data access

Tokens live in **httpOnly cookies** set by this app's server, never in `localStorage` or anywhere page scripts can read them. The browser never calls Django directly.

- **Login / register / logout** run on the server (Server Actions or Route Handlers): they call Djoser, then set or clear the `access` and `refresh` cookies with `httpOnly`, `secure` (in production), `sameSite: "lax"`, `path: "/"`, and an expiry matching the token lifetime.
- **Server Components** read the access cookie through a `server-only` data-access module and call Django with the `JWT` header. Do auth checks there, close to the data — not in layouts, which don't re-render on navigation.
- **Client Components** call a same-origin Route Handler proxy under `/api/…`, which attaches the `JWT` header from the cookie and forwards to Django. Pass Django's status code and error body through untouched so forms can show its validation errors.
- **Refreshing**: Server Components cannot write cookies, so an expired access token is refreshed (`/auth/jwt/refresh/`) in `proxy.ts` or a Route Handler, not during render. If refresh fails, clear both cookies and send the user to `/login`.
- **Route protection**: `proxy.ts` at the repo root (Next 16's name for middleware) does the optimistic check — no cookie → redirect `(app)` routes to `/login`; has cookie → redirect guest-only routes to `/schedule`. It only reads the cookie; it is not the security boundary. Django is.

Keep one configured axios instance per side (server → Django, client → `/api`) in `lib/`; don't call `axios` or `fetch` ad hoc from components.

## Design

The app must be aesthetically beautiful, polished, and modern — not a generic template. Every page follows these guidelines so the product stays consistent.

**Identity.** A science tutoring brand: friendly and trustworthy, curious rather than corporate. The violet `--primary` already in `app/globals.css` is the brand colour; use it for primary actions and key highlights, not as a wash over everything. Pair it with calm neutral surfaces and at most one supporting accent.

**Tokens, not hardcoded colours.** Build on the tokens in `app/globals.css` and the shadcn/Base UI components. When something new is needed (a success colour, a per-subject accent, a brand gradient stop), add a CSS variable under both `:root` and `.dark`, map it in `@theme inline`, and use the resulting utility. No hex/OKLCH literals or raw palette classes (`bg-purple-500`) in components.

**Typography and spacing.** Inter for everything, Geist Mono only for genuinely tabular or code-like content. Use a clear type scale with confident headings, comfortable line length (about 60–75 characters for prose), and `tabular-nums` for prices and times. Spacing is generous: let sections breathe, and prefer whitespace over borders and boxes to separate things.

**Responsive, mobile first.** Write the small-screen layout first and add breakpoints upward. No horizontal scrolling; touch targets at least 44px; the weekly schedule must be genuinely usable on a phone (e.g. a per-day list), not a shrunken desktop grid.

**Accessible.** WCAG AA contrast in both themes; visible focus states (keep the `focus-visible` rings from the UI components); full keyboard navigation; every input has a real `<label>`; errors are tied to their field with `aria-invalid`/`aria-describedby`; icon-only buttons have an accessible name; never convey state by colour alone.

**Every data view has loading, empty, and error states.** Loading uses skeletons shaped like the content (`loading.tsx` or `<Suspense>`), not a lone spinner. Empty states explain what belongs there and offer the next action ("No classes yet — book your first"). Error states say what went wrong in plain language and offer a retry (`error.tsx` for route-level failures).

**Motion.** Subtle and purposeful only: short transitions (roughly 150–250ms) that explain a state change. Nothing decorative or looping, and respect `prefers-reduced-motion`.

**Light and dark.** Both themes are first-class; check every screen in both. Theme switching is planned via `next-themes` (not yet installed) toggling the `.dark` class, defaulting to the system setting.

## Testing

**Do not write tests for this frontend and do not add a test runner.** Verify work with:

```
npm run lint
npx tsc --noEmit
npm run build
```

## Implemented vs Stub Pages

Routes live in three route groups, each with its own layout: `(marketing)` (public), `(auth)` (guest only), and `(app)` (logged-in students). **Keep this table updated as pages are built** — flip the status in the same change that implements the page.

| Path | Group | Purpose | Backend endpoints | Status |
| ---- | ----- | ------- | ----------------- | ------ |
| `/` | `(marketing)` | Landing: pitch, subject and price preview, calls to action for the free trial | `GET /subjects/` | Stub |
| `/subjects` | `(marketing)` | Browse all subjects with 40- and 60-minute prices | `GET /subjects/` | Stub |
| `/subjects/[id]` | `(marketing)` | Subject detail with "book weekly class" and "book trial" actions | `GET /subjects/{id}/` | Stub |
| `/login` | `(auth)` | Sign in | `POST /auth/jwt/create/`, `GET /auth/users/me/` | Stub |
| `/register` | `(auth)` | Create an account, then sign in automatically | `POST /auth/users/`, `POST /auth/jwt/create/` | Stub |
| `/forgot-password` | `(auth)` | Request a password-reset email | `POST /auth/users/reset_password/` | Stub |
| `/password/reset/confirm/[uid]/[token]` | `(auth)` | Set a new password from the emailed link | `POST /auth/users/reset_password_confirm/` | Stub |
| `/schedule` | `(app)` | Home after login: weekly timetable, trial lesson, total weekly cost | `GET /schedule/` | Stub |
| `/classes/new` | `(app)` | Book a weekly class (subject, weekday, full hour, 40 or 60 minutes) | `GET /subjects/`, `POST /classes/` | Stub |
| `/classes/[id]/edit` | `(app)` | Change or cancel a weekly class | `GET` / `PATCH` / `DELETE /classes/{id}/` | Stub |
| `/trial-lesson` | `(app)` | Book the one free trial; if it exists, show it with edit/delete while unlocked, read-only once locked | `GET` / `POST /trial-lessons/`, `GET` / `PATCH` / `DELETE /trial-lessons/{id}/` | Stub |
| `/account` | `(app)` | Edit profile and change password | `GET` / `PATCH /auth/users/me/`, `POST /auth/users/set_password/` | Stub |

Not pages, but part of the plan: `proxy.ts` (route protection and token refresh), the `/api/…` proxy Route Handler, and global `not-found` / `error` UI. `POST /auth/jwt/refresh/` and `/auth/jwt/verify/` are used by the auth layer, not by a page.

The reset-confirm path must match Djoser's `PASSWORD_RESET_CONFIRM_URL` in the Django settings; if the backend uses a different pattern, change the route here to match rather than the other way round.
