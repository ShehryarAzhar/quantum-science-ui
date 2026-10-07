---
description: Create a spec file and feature branch for a Quantum Science UI feature. Pass a feature name e.g. /create-spec trial-lesson
argument-hint: foundation | subjects | auth | schedule | weekly-classes | trial-lesson | account | <number> <feature name>
allowed-tools: Read, Write, Glob, Grep, Bash(git:*)
---

You are a senior frontend developer spinning up a new feature
for the Quantum Science UI. Always follow the rules in
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

| Feature | `step_number` | `feature_title` | Scope | Spec source in `CLAUDE.md` |
| --- | --- | --- | --- | --- |
| `foundation` | 01 | App Foundation | No pages: the `(marketing)`, `(auth)` and `(app)` layouts, nav and footer, providers (React Query, theme), both axios instances, the `server-only` data-access module, the `/api/…` proxy Route Handler, `proxy.ts`, global `not-found` / `error` | Auth and data access · Design · Conventions that differ from the usual defaults · the "Not pages" paragraph under Implemented vs Stub Pages |
| `subjects` | 02 | Subjects and Landing | `/`, `/subjects`, `/subjects/[id]` | Backend API contract (Subjects) · Money |
| `auth` | 03 | Authentication | `/login`, `/register`, `/forgot-password`, `/password/reset/confirm/[uid]/[token]`, logout | Backend API contract (Auth routes) · Auth and data access |
| `schedule` | 04 | My Schedule | `/schedule` | Backend API contract (My schedule) · Money · Timezones |
| `weekly-classes` | 05 | Weekly Class Booking | `/classes/new`, `/classes/[id]/edit` | Backend API contract (Weekly classes) · Booking rules · Timezones · Money |
| `trial-lesson` | 06 | Trial Lesson | `/trial-lesson` | Backend API contract (Trial lessons) · Booking rules · Timezones |
| `account` | 07 | Account | `/account` | Backend API contract (Auth routes: `users/me`, `set_password`) |

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
- `CLAUDE.md` — project, backend API contract, booking
  rules, money, timezones, auth and data access, design,
  page table
- `AGENTS.md`, then the guides in
  `node_modules/next/dist/docs/` for whatever the feature
  touches (route handlers, server actions, `proxy.ts`,
  forms, `loading` / `error` / `not-found` files). This
  Next.js version differs from what you know; the spec
  must name the APIs as these guides describe them
- `package.json` — installed dependencies
- `components.json` and `app/globals.css` — shadcn setup
  and design tokens
- Whatever exists under `app/`, `components/` and `lib/`,
  and `proxy.ts` — reuse what is already there
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
Base the spec on the feature's sections of `CLAUDE.md`. Do
not invent behaviour that is not there.

`CLAUDE.md` deliberately does not document exact API field
names. Before writing, collect every ambiguity and ask the
user about them together. Always ask about:
- the request and response field names and types of each
  backend endpoint the feature uses, unless an earlier
  spec already records them
- the shape of a backend error the UI has to display,
  where it is not obvious
- any behaviour `CLAUDE.md` leaves open (e.g. where the
  student lands after booking a class)

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
`CLAUDE.md` that this spec implements, in its own words but
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
Each TypeScript type and zod schema, the file in `lib/` it
lives in, and where it is reused. A response is typed once.
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
Client Component, and what it renders. List the shadcn
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
- Prices stay decimal strings; convert to integer cents for
  any client-side sum; format with `Intl.NumberFormat`
- All timezone conversion goes through the one helper
  module in `lib/`; weekday and hour are converted
  together; local times render in Client Components
- shadcn components come from the CLI and are built on
  Base UI (`render` prop, no `@radix-ui/*`); `cn` is
  imported from `@/lib/utils`
- Semantic tokens only; no hex/OKLCH literals or raw
  palette classes in components
- No `src/` directory; imports use the `@/*` alias
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
