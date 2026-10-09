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
