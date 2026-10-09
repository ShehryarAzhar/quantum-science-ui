## SEO

Science Nest should rank well in search, and its pages should be SEO friendly as they are built, not fixed up afterwards. Every public page follows this section.

**Audience.** Students, and their parents, looking for an online science tutor: Physics, Chemistry, Biology and other science subjects, at O Level, A Level, school level (1 to O Level) and university level. Classes are online, so students can be anywhere in the world. The site is in English and prices are in USD.

### What is indexed

- Only the public `(marketing)` pages are indexed. The `(auth)` and `(app)` group layouts set `robots: { index: false, follow: false }` in their metadata.
- `robots.ts` allows everything except `/api/` and points to the sitemap. Don't disallow the private pages there: a crawler that is blocked from a page never sees its `noindex`.
- Pages are indexable only when `ALLOW_INDEXING` is `true`. Otherwise `robots.ts` disallows everything and the root metadata is `noindex`, so a preview or staging deployment never competes with the live site.

### Metadata

Use Next's metadata API, and read its guides in `node_modules/next/dist/docs/` first, as `AGENTS.md` instructs.

- **Root layout**: `metadataBase` from `SITE_URL`; `title: { default, template: "%s | Science Nest" }`; a default description; Open Graph defaults (`siteName`, `type`, `locale`) and Twitter defaults (`summary_large_image`); `verification` from the two verification variables when they are set.
- **Every public page** has its own unique title (the page's main search phrase first, about 60 characters before the template), its own unique meta description (about 140 to 160 characters, saying what the page offers) and `alternates.canonical`. A static page exports `metadata`; a page that depends on data uses `generateMetadata` and shares its fetch with the page through React's `cache`.
- **Share images**: a site-wide `opengraph-image`, and a generated one for each subject page (`ImageResponse` from `next/og`, 1200×630, with alt text).
- **`sitemap.ts`** lists every indexable page with an absolute URL: `/`, `/subjects`, every subject page, `/how-it-works`, `/faq` and `/about`. If the subjects API can't be reached, it still lists the static pages.
- After launch, verify the site in Google Search Console and Bing Webmaster Tools and submit the sitemap there.

### URLs

- One canonical host: the one in `SITE_URL`. The other form (with or without `www`) redirects to it.
- Lowercase, hyphenated, no trailing slash. Canonical URLs carry no query string.
- A subject's URL uses the backend's `slug`: `/subjects/chemistry`. Never derive a slug from the subject's name; a renamed subject must keep its URL.
- An unknown slug calls `notFound()` so the response is a real 404.
- There is no separate page per subject and level. A subject's levels are covered on its own page, each under its own heading, using `levels.ts`.

### Rendering and speed

- Public pages are Server Components, prerendered and revalidated, so the full content is in the HTML without running JavaScript. Which caching API does this (Cache Components with `use cache` and `cacheLife`, or the previous `revalidate` model) is decided in the `foundation` spec from the docs. Prices can change in the Django admin, so subject data revalidates on a short interval instead of being cached indefinitely.
- Public pages fetch subjects without reading cookies: reading cookies makes a page dynamic. Anything in the marketing nav that depends on being signed in is a small Client Component.
- Speed counts for ranking. Use `next/image` with real dimensions and `sizes`, and priority loading only for the largest image above the fold. Load fonts through `next/font`, and only fonts that are used. Keep client JavaScript on public pages minimal: no React Query, form or phone libraries there. Aim for LCP under 2.5 s, INP under 200 ms and CLS under 0.1.

### Structured data

- JSON-LD is rendered on the server as a native `<script type="application/ld+json">`, through one shared component that escapes `<` as the Next guide shows.
- What goes where:
  - `/`: `EducationalOrganization` and `WebSite`. Not `LocalBusiness`: there is no physical address.
  - Subject pages: `Course`, with Science Nest as the provider and one `Offer` per class length (USD, the API's price written with two decimal places, e.g. `20.00`), plus `BreadcrumbList` matching the breadcrumb shown on the page.
  - `/faq`: `FAQPage`, generated from the same content file as the visible questions so the two can't disagree.
  - `/about`: `Person` for the tutor, only from facts the user has supplied.
- **Only describe what is true and visible on the page.** Never invent reviews, ratings, student numbers, pass rates or results.
- Structured data helps search engines understand a page; it doesn't guarantee a rich result. Validate it with Google's Rich Results Test.

### Semantic HTML

- Use the element that fits: `header`, `nav`, `main`, `section`, `article`, `aside`, `footer`; real lists for lists; `time` with `dateTime` for dates and times.
- Exactly one `h1` per page, describing that page. Headings follow a logical order with no skipped levels, and a heading level is never picked for its size.
- Link text describes its destination ("View A Level Chemistry", not "click here"). Every image has meaningful `alt` text; only a purely decorative image has an empty one. `<html>` keeps `lang="en"`.
- Good accessibility helps SEO too: the rules under [Design](design.md) apply to every page.

### Content

Claude writes the site's copy, aimed at what people actually search for. Each page has one main search phrase, and no two pages target the same one:

| Page | Main search phrase |
| ---- | ------------------ |
| `/` | online science tutor |
| `/subjects` | online science classes |
| `/subjects/[slug]` | online {subject} tutor, and the same phrase for each of its levels (e.g. "online physics tutor", "online A Level physics tutor", "O Level chemistry online classes") |
| `/how-it-works` | how online science tutoring works |
| `/faq` | questions about online tutoring, the free trial, pricing |
| `/about` | the tutor by name, and science tutor credentials |

- Use the phrase naturally in the title, the `h1`, the first paragraph and a subheading. No keyword stuffing, no hidden text, and no block of copy repeated across pages.
- Every public page has enough useful, unique content. Between them the pages cover what is taught, the levels, how online classes work, how booking and the free trial work, pricing, and an FAQ answering what students and parents really ask (timezones, class length, what happens in the trial).
- Link pages to each other: the subjects list links to every subject page, each subject page shows a breadcrumb back to it, and the footer links to every public page.
- The tone is clear, friendly and trustworthy, matching the identity under [Design](design.md).
- Make claims only about things the site actually offers. Facts Claude can't know (the tutor's name, qualifications and experience, exam boards covered) are asked for, never invented.

### Where content lives

- **Subject descriptions come from the API** (`description`), written in the Django admin. It is plain text: render it as paragraphs, never as HTML. A good one is 150 to 300 words in a few short paragraphs: what is taught, at which levels, and who it suits. If it is empty, the page shows a generic introduction from `content/` built around the subject's name, levels and prices.
- A subject page's meta description is built from the subject's name, levels and lowest price, so it is always unique and true.
- **All other copy lives in `content/` at the repo root**, one typed TypeScript file per kind, so there is one obvious place to edit each piece of text:
  - `site.ts` — site name, default title and description
  - `home.ts` — landing page copy
  - `faq.ts` — questions and answers
  - `levels.ts` — what each level covers, keyed by level code
  - `howItWorks.ts`, `about.ts` — those pages' copy
- Components on public pages import their text from `@/content/...`; don't hardcode marketing copy in a component.
