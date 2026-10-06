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

The project is a fresh scaffold: `app/layout.tsx`, a placeholder `app/page.tsx`, and one UI component (`components/ui/button.tsx`). `next.config.ts` is empty.

## Conventions that differ from the usual defaults

**No `src/` directory.** `app/`, `components/`, and `lib/` sit at the repo root, and the `@/*` path alias maps to the root (`@/components/ui/button`, `@/lib/utils`).

**shadcn uses the `base-vega` style, so primitives come from `@base-ui/react`, not Radix.** Base UI's API differs: composition uses the `render` prop rather than `asChild`, and prop types are `Primitive.Props` (see `components/ui/button.tsx`). Don't add `@radix-ui/*` packages or copy Radix-flavoured shadcn snippets; generate components through the shadcn CLI so they match `components.json`.

**`cn` comes from the `cn` npm package** (shadcn's compiled replacement for `clsx` + `tailwind-merge`), neither of which is installed. `lib/utils.ts` just re-exports it; generated components import from `"cn"` directly, app code uses `@/lib/utils`.

**Tailwind v4 is configured entirely in CSS.** There is no `tailwind.config.*`; `app/globals.css` holds the imports (`tailwindcss`, `tw-animate-css`, `shadcn/tailwind.css`), the `@theme inline` token mapping, and the design tokens as OKLCH CSS variables under `:root` and `.dark`. To add or change a colour/radius, edit the variable there and (for new tokens) map it in `@theme inline`. Use semantic utilities (`bg-primary`, `text-muted-foreground`, `border-border`) rather than raw palette colours.

**Dark mode is class-based** (`@custom-variant dark (&:is(.dark *))`) — it activates when an ancestor has the `.dark` class. Nothing toggles that class yet; there is no theme provider.

**Fonts** are set in `app/layout.tsx` via `next/font/google`: Inter is bound to `--font-sans` (body and headings), Geist Mono to `--font-geist-mono` (`font-mono`). Geist Sans is loaded but its variable is not referenced by the theme.

**Route prop types are generated globals.** `layout.tsx` uses `LayoutProps<"/">` without an import; these helpers (`LayoutProps`, `PageProps`) are emitted into `.next/types` by `next dev` / `next build` / `next typegen`, so a fresh checkout shows type errors on them until one of those has run.

**Icons:** `lucide-react`.
