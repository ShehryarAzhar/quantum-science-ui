---
description: Create a spec file and feature branch for a Science Nest UI feature. Pass a feature name e.g. /create-spec trial-lesson
argument-hint: foundation | subjects | auth | schedule | weekly-classes | trial-lesson | account | content-pages | <number> <feature name>
allowed-tools: Read, Write, Glob, Grep, Bash(git:*)
---

You are a senior frontend developer spinning up a new feature
for the Science Nest UI. Always follow the rules in
CLAUDE.md and AGENTS.md.

User input: $ARGUMENTS

## Step 1 — Check working directory is clean
Run `git status` and check for uncommitted, unstaged, or
untracked files. If any exist, stop immediately and tell
the user to commit or stash changes before proceeding.
DO NOT CONTINUE until the working directory is clean.

## Step 2 — Parse the arguments
If $ARGUMENTS is one of the known features, take the values
from this table:

| Feature | `step_number` | `feature_title` | Scope | Spec source (file: sections) |
| --- | --- | --- | --- | --- |
| `foundation` | 01 | App Foundation | No pages: the `(marketing)`, `(auth)` and `(app)` layouts, nav and footer, providers (React Query, theme), both axios instances, the `server-only` data-access module, the `/api/…` proxy Route Handler, `proxy.ts`, global `not-found` / `error`, the shared "not a student account" notice, the SEO base (root metadata, `robots.ts`, `sitemap.ts`, site share image, JSON-LD component, `noindex` on `(auth)` and `(app)`) | `docs/auth.md`: Auth and data access · `docs/design.md`: Design · `docs/seo.md`: SEO (What is indexed, Metadata, URLs, Rendering and speed, Structured data) · `CLAUDE.md`: Environment · Conventions that differ from the usual defaults · the "Not pages" paragraph under Implemented vs Stub Pages |
| `subjects` | 02 | Subjects and Landing | `/`, `/subjects`, `/subjects/[slug]`, each showing a subject's levels and prices | `docs/api-contract.md`: Backend API contract (Subjects) · Money · `docs/seo.md`: SEO (all of it) |
| `auth` | 03 | Authentication | `/login`, `/register` (with phone number), `/forgot-password`, `/reset-password/[uid]/[token]` (open to everyone), logout | `docs/api-contract.md`: Backend API contract (Auth routes) · Phone numbers · `docs/auth.md`: Auth and data access (Login / register / logout, Route protection, Password changes, Forgot password) |
| `schedule` | 04 | My Schedule | `/schedule` | `docs/api-contract.md`: Backend API contract (My schedule, No student profile) · Money · Timezones · `docs/auth.md`: Auth and data access (Not a student account) |
| `weekly-classes` | 05 | Weekly Class Booking | `/classes/new`, `/classes/[id]/edit`, both with the level choice | `docs/api-contract.md`: Backend API contract (Subjects, Weekly classes, No student profile) · Booking rules · Timezones · Money · `docs/auth.md`: Auth and data access (Not a student account) |
| `trial-lesson` | 06 | Trial Lesson | `/trial-lesson`, with the level choice | `docs/api-contract.md`: Backend API contract (Subjects, Trial lessons, No student profile) · Booking rules · Timezones · `docs/auth.md`: Auth and data access (Not a student account) |
| `account` | 07 | Account | `/account`: edit profile, read-only username and phone number, change password, delete account | `docs/api-contract.md`: Backend API contract (Auth routes: `users/me`, `set_password`) · Phone numbers (Display) · `docs/auth.md`: Auth and data access (Password changes, Delete account) |
| `content-pages` | 08 | Content Pages | `/how-it-works`, `/faq`, `/about`, and their links in the nav and footer | `docs/seo.md`: SEO (all of it) · `docs/design.md`: Design |

Otherwise expect `<number> <feature name>` (e.g. `8 contact
page`) and extract:

1. `step_number` — zero-padded to 2 digits: 8 → 08, 11 → 11
2. `feature_title` — human readable title in Title Case
3. `feature_slug` — lowercase kebab-case, only a-z, 0-9
   and -, maximum 40 characters

For known features `feature_slug` is the feature name.
In both cases `branch_name` is `feature/<feature_slug>`.

If you cannot infer these from $ARGUMENTS, ask the user
to clarify before proceeding.

## Step 3 — Check branch name is not taken
Run `git branch -a` to list existing branches.
If `branch_name` is already taken, append a number:
`feature/trial-lesson-01`, `feature/trial-lesson-02` etc.

## Step 4 — Switch to main and pull latest
Run:
```
git checkout main
git pull origin main
```

## Step 5 — Create and switch to the feature branch
Run:
```
git checkout -b <branch_name>
```

## Step 6 — Research the codebase
Read these before writing the spec:
- `CLAUDE.md` — project, conventions, environment, feature
  workflow, page table. It imports the five files below;
  read each one, since reading `CLAUDE.md` alone does not
  show their content
- `docs/api-contract.md` — backend API contract, money,
  booking rules, timezones, phone numbers
- `docs/auth.md` — auth and data access
- `docs/code-organization.md` — code organization
- `docs/design.md` — design
- `docs/seo.md` — SEO
- `AGENTS.md`, then the guides in
  `node_modules/next/dist/docs/` for whatever the feature
  touches (route handlers, server actions, `proxy.ts`,
  forms, `loading` / `error` / `not-found` files). This
  Next.js version differs from what you know; the spec
  must name the APIs as these guides describe them
- `package.json` — installed dependencies
- `components.json` and `app/globals.css` — shadcn setup
  and design tokens
- Whatever exists under `app/`, `components/`, `lib/` and
  `content/`, and `proxy.ts` — reuse what is already there
- All files in `.claude/specs/` — avoid duplicating
  existing specs and pick up any rule an earlier spec
  deferred to this feature

Research this repository only. Do not read the backend
repository, even if it is available on disk.

Check the "Implemented vs Stub Pages" section of
`CLAUDE.md`. If every page of the requested feature is
already marked Implemented, warn the user and stop. For
`foundation`, which has no pages, do the same if `proxy.ts`,
the `/api/…` Route Handler and all three route-group
layouts already exist.

## Step 7 — Write the spec
Base the spec on the feature's sections of `CLAUDE.md` and
the `docs/` files it imports. Do not invent behaviour that is not there.

`docs/api-contract.md` documents only some API field names (e.g.
`slug`, `description`, `price_40_min`, `price_60_min`,
`levels`, `level`, `day`, `day_display`, `starts_at`,
`weekly_classes`, `trial_lesson`, `weekly_cost`); the rest
(e.g. the time and duration fields of a weekly class) are
deliberately left out.
Before writing, collect every ambiguity and ask the user
about them together. Always ask about:
- the request and response field names and types of each
  backend endpoint the feature uses, unless
  `docs/api-contract.md` or an earlier spec already records them
- the shape of a backend error the UI has to display,
  where it is not obvious
- any behaviour `CLAUDE.md` and the `docs/` files leave
  open (e.g. where the student lands after booking a
  class)
- for a feature with public pages, every fact the copy
  needs that you cannot know (the tutor's name,
  qualifications and experience, exam boards covered,
  anything else the site would claim). Never invent one
- for `schedule`, whether `price` and `weekly_cost` are
  numbers like the subject prices

Never guess a field name. Write each answer into the spec
and mark it "confirmed by user".

Generate a spec document with this exact structure:

---
# Spec: <feature_title>

## Overview
One paragraph describing what this feature does and why
it is being built at this point.

## Depends on
Which features and specs must already be implemented.
If none: state "No dependencies".

## Requirements
Numbered list of every rule from the feature's sections of
`CLAUDE.md` and the `docs/` files that this spec implements, in its own words but
without changing the meaning.

## Deferred rules
Rules of this feature that cannot be implemented yet
because they depend on something that does not exist (e.g.
the schedule's link to the class edit page before weekly
classes exist), and which spec must pick them up. Also list
any rule an earlier spec deferred to this one.
If none: state "No deferred rules".

## Pages and routes
Every new page:
- `/path` — route group — purpose — access level
  (public / guest only / logged-in student) — which of
  `loading.tsx`, `error.tsx`, `not-found.tsx` it gets

If no new pages: state "No new pages".

## API contract
Each backend endpoint the feature uses: method, path (with
trailing slash), whether it needs auth, request fields,
response fields, and the error responses the UI handles.
Field names are as confirmed by the user.
If none: state "No backend calls".

## Types and schemas
Each TypeScript type, the file in `lib/` it lives in, and
where it is reused. A response is typed once. Each zod
schema, by its export name: all of them live in
`app/validationSchemas.ts`, never in a component, page or
other file.
If none: state "No new types".

## Data access
For each piece of data, how it is loaded: in a Server
Component through the `server-only` data-access module, or
in a Client Component with React Query through the
`/api/…` proxy. List every Server Action and Route Handler
added, every query key, and what each mutation invalidates
or revalidates.

## Components
Each new component, its file, whether it is a Server or
Client Component, and what it renders. Place each file by
the "Code organization" section of
`docs/code-organization.md` and say
which routes use it: one route → that route's folder; a
route and its nested routes → the parent route folder;
unrelated routes → `app/components/`. List any existing
component that must move because its usage changes. Each
`page.tsx` only fetches data and lays out components. List
the shadcn
components to add with `npx shadcn@latest add <component>`.

## Forms and validation
Each form: its fields and labels, the zod rule for each
field, and how backend errors are shown (field errors next
to the field, `non_field_errors` / `detail` in a form-level
alert).
If none: state "No forms".

## States
For every data view, the loading state (what the skeleton
looks like), the empty state (message and next action) and
the error state (message and retry).

## Time and money
Which values go through the timezone helper in `lib/` and
in which direction, and how each price is formatted or
summed.
If none: state "Not applicable".

## SEO
For each public `(marketing)` page, following the "SEO"
section of `docs/seo.md`:
- its URL and canonical URL
- its main search phrase (no other page may share it)
- its title and meta description, written out in full
- its `h1` and heading outline
- the JSON-LD types it renders and where their values come
  from
- its share image
- whether it is in the sitemap, and when it is `noindex`
- how it is rendered (prerendered, revalidation interval)

For a page outside `(marketing)`: state that it is
`noindex`.
If the feature has no pages: state "Not applicable".

## Content
Each file under `content/` that is added or changed, what
it holds, and the copy itself for new public pages. List
any fact still needed from the user.
If none: state "No content changes".

## Design and accessibility
The mobile layout first, then what changes at each
breakpoint. Any token added to `app/globals.css`. Labels,
focus order, `aria-*` attributes and keyboard behaviour
that this feature needs. Anything that differs between
light and dark.

## Files to change
Every existing file that will be modified, including
`CLAUDE.md` for the page table.

## Files to create
Every new file that will be created.

## New dependencies
Any new packages, added with `npm install`.
If none: state "No new dependencies".

## Rules for implementation
Specific constraints Claude must follow. Always include:
- Read the relevant guide in `node_modules/next/dist/docs/`
  before writing code
- The browser never calls Django; tokens live only in
  httpOnly cookies; `API_URL` is never `NEXT_PUBLIC_`
- The auth header is `Authorization: JWT <token>` and every
  Django path ends with a trailing slash
- One configured axios instance per side in `lib/`; no ad
  hoc `axios` or `fetch` in components
- The backend is the source of truth: always display its
  validation errors, and pass its status code and error
  body through the proxy untouched
- Prices arrive as numbers; convert to whole cents
  (`Math.round(price * 100)`) before any arithmetic and
  sum in cents; format with `Intl.NumberFormat`, never
  print the raw number
- All timezone conversion goes through the one helper
  module in `lib/`; weekday and hour are converted
  together; local times render in Client Components
- Schedule data is sorted by local day and time after
  converting, never left in the order the API returns
- A weekly class and a trial lesson each need a `level`
  that is one of the chosen subject's levels; level names
  come from the API, never hardcoded
- A 403 on reading `/schedule/`, `/classes/` or
  `/trial-lessons/` renders the shared "not a student
  account" notice; a 403 on editing or deleting a locked
  trial lesson shows the backend's message
- A password change invalidates every earlier token:
  follow "Password changes" under Auth and data access in
  `docs/auth.md`
- A phone field uses the ReUI phone input and the one
  phone schema, as "Phone numbers" in
  `docs/api-contract.md` describes: default country `US`, strict validation with
  the max metadata, E.164 sent to the backend; no phone
  input built on Radix or `cmdk`
- shadcn components come from the CLI and are built on
  Base UI (`render` prop, no `@radix-ui/*`); `cn` is
  imported from `@/lib/utils`
- Semantic tokens only; no hex/OKLCH literals or raw
  palette classes in components
- No `src/` directory; imports use the `@/*` alias
- Every zod schema lives in `app/validationSchemas.ts`
- Components are placed by where they are used (route
  folder, parent route folder, or `app/components/`);
  shadcn primitives stay in `components/ui/`
- `page.tsx` fetches data and lays out components; UI
  blocks, form logic and interactive state live in small,
  focused components
- Public pages are Server Components, prerendered and
  revalidated, with their full content in the HTML; they
  read no cookies and ship minimal client JavaScript
- Every public page has a unique title, meta description
  and canonical URL, exactly one `h1`, headings in order,
  semantic elements and descriptive link text
- Marketing copy lives in `content/`, not in components; a
  subject's description comes from the API and is rendered
  as plain text
- JSON-LD describes only what is true and on the page: no
  invented reviews, ratings, numbers or results
- Pages outside `(marketing)` are `noindex`
- No teacher or admin pages
- No tests and no test runner
- Update the "Implemented vs Stub Pages" table in
  `CLAUDE.md` in the same change

Add any rule specific to this feature.

## Verification
There are no tests. State the commands (`npm run lint`,
`npx tsc --noEmit`, `npm run build`) and the manual checks
this feature needs in the browser: both themes, phone
width, keyboard only, logged out and logged in.

## Definition of done
A specific testable checklist. Each item must be
verifiable by an action in the browser or a command.
Always include:
- [ ] `npm run lint`, `npx tsc --noEmit` and
      `npm run build` pass
- [ ] Each page behaves as specified for a logged-out and
      a logged-in visitor
- [ ] Every data view shows its loading, empty and error
      state
- [ ] Each page works at 360px wide with no horizontal
      scrolling
- [ ] Each page is checked in light and dark
- [ ] Each public page's source, viewed with JavaScript
      off, contains its full content, one `h1`, and a
      unique title, meta description and canonical URL
- [ ] Each public page's JSON-LD passes Google's Rich
      Results Test, and the page is in `/sitemap.xml` or
      `noindex` as the spec says
- [ ] Each page outside `(marketing)` is `noindex`
- [ ] The feature's pages are marked Implemented in
      `CLAUDE.md`
---

## Step 8 — Save the spec
Save to: `.claude/specs/<step_number>-<feature_slug>.md`

## Step 9 — Report to the user
Print a short summary in this exact format:
```
Branch:    <branch_name>
Spec file: .claude/specs/<step_number>-<feature_slug>.md
Title:     <feature_title>
```

Then tell the user:
"Review the spec at `.claude/specs/<step_number>-<feature_slug>.md`
then enter Plan Mode with Shift+Tab twice to begin implementation.
Once it is implemented, verify with `npm run lint`,
`npx tsc --noEmit` and `npm run build`."

Do not print the full spec in chat unless explicitly asked.
