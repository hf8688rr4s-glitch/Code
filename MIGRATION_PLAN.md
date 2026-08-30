# Atelier Digital — Framer → React/Next.js Migration Plan

**Source:** Framer project "Atelier Digital Site WEB" (`UYqf9dEhEfs6kJrwE199`)
**Target:** Standalone, production-ready React/Next.js codebase, pushable to GitHub and importable into Lovable
**Goal:** Maximum visual fidelity to the existing Framer site
**Status of this document:** Planning only — no code has been written, no changes were made to the Framer project (read-only audit).

---

## 1. Executive Summary

Atelier Digital is a 7-page, French/English marketing site for a Quebec-based video production & digital marketing studio. It is built entirely from Framer-native primitives (no unusual custom code beyond the default boilerplate override file), backed by 3 CMS collections (Projects, Blog, Project Category), using a single shared layout template (fixed nav + footer), a small design-token system, and a component library of 22 reusable components. Interactions are simple and state-driven (Framer variants + `SET_VARIANT`), not physics-heavy — everything is realistically portable to **Framer Motion** in React with high fidelity.

**Complexity verdict:** Low-to-medium. Nothing here requires exotic reverse-engineering — the CMS content, design tokens, typography, and image/video assets are extractable programmatically; the interactions are all reproducible with well-known React patterns. The only genuinely "recreate from scratch" items are Framer-marketplace components (Smooth Scroll, Video player, Tab, hamburger icon, arrow icon, Buy Badges) since their internal implementation isn't inspectable — but their *observed behavior* is simple enough to rebuild directly.

---

## 2. Site Map & Page Inventory

| Path | Purpose | Notes |
|---|---|---|
| `/` (Home) | Hero/reel showcase of 5 latest projects (scroll-driven video reveals) + manifesto/services/CTA | 216 descendants (largest static page) |
| `/projects` | Filterable project grid (All / Branded Content / Fashion Film / Music Video / Commercial) | Category filter via query params |
| `/projects/:slug` | CMS-driven project detail (client/type/year/industry, gallery, video) | Dynamic route, 5 CMS items |
| `/about` | Manifesto, "method" statement, client logo grid, team cards, CTA | 261 descendants (most complex page) |
| `/services` | 6 numbered service entries (Production vidéo, Marketing digital, SEO & GEO, Conception web, Gestion réseaux sociaux, Photographie) | Large type-driven layout |
| `/contact` | Statement + contact form (Nom / Email / Numero de téléphone) | Static form, no visible backend wiring found |
| `/404` | Error page | `noIndex` |

All pages share one **Layout Template** (nav + footer), confirmed via screenshots and structural serialization — every page renders:
- Fixed/floating **Navigation** bar (logo left, 4 links + hamburger toggle right)
- Page-specific content
- Shared black **footer/"About" band** (Logo, 3 social icons — LinkedIn/Vimeo/Instagram, "Québec, Canada" credit)

**Breakpoints (confirmed from live tree):** Desktop `1200px`, Tablet `810px`, Phone `390px` — maps directly to a Tailwind config with custom breakpoints `sm: 390px / md: 810px / lg: 1200px`, or Tailwind's default `md`/`lg` if you prefer standard naming (document the mapping either way).

---

## 3. Design System → Tailwind/CSS Tokens

### 3.1 Color tokens (5, no dark-mode variants defined)
| Token name | Value | Suggested Tailwind var |
|---|---|---|
| Semi white | `rgb(235,235,235)` | `--color-semi-white` |
| Black | `rgb(17,18,18)` | `--color-black` |
| Black 35% | `rgba(17,18,18,0.35)` | `--color-black/35` |
| Gray 30% | `rgba(235,235,235,0.3)` | `--color-gray/30` |
| Black 45% | `rgba(17,18,18,0.45)` | `--color-black/45` |

→ Directly extractable, 1:1 portable to `tailwind.config` `theme.extend.colors` or CSS custom properties.

### 3.2 Typography
- **Font:** Barlow Condensed (single family across the whole site) — available on Google Fonts, drop-in via `next/font/google`.
- ~10 unique text style presets (20 incl. hover/replica duplicates), mostly **uppercase**, weights **300 / 600 / 800**.
- Key presets to recreate as Tailwind text-style utilities or CSS classes:
  - `Body` — 18px / 600
  - `Body light` — (used for links/nav, lighter weight variant)
  - `Heading 1` — responsive 60 / 48 / 38px, 600, uppercase
  - `Heading 2 big` — responsive 100 / 64 / 51px, 600, uppercase (used for big CTA/manifesto statements — confirmed in screenshots: "FAÇONNONS QUELQUE CHOSE.")
  - `Heading 1 big` — responsive 90 / 72 / 58px, 600
  - `Heading 3 normal` / `Heading 3 small` — used in labels, cards, logo
  - `Services description` / `Services label` — responsive 52/42/34px and 800 weight, with OpenType stylistic sets `blwf, cv03, cv04, cv09, cv11` (Barlow Condensed stylistic alternates — confirm exact glyph substitutions and replicate via `font-feature-settings` in CSS, since Tailwind doesn't expose these by default)
- **Fidelity flag:** the responsive per-breakpoint font sizes (e.g. 100/64/51px) must become explicit `clamp()` or breakpoint-specific classes — don't rely on `vw`-based fluid scaling since the source uses discrete breakpoint values, not fluid interpolation.

### 3.3 Inconsistency to resolve during migration
Button Form's **Error** state uses a hardcoded `rgb(255,0,0)` and switches font to **Geist Mono** instead of the standard `Body` preset/color tokens. Decision needed: **replicate exactly** (true fidelity) vs. **normalize** to the design system (cleaner but diverges from source). Recommend flagging to the user before implementation; default recommendation is to replicate exactly per the "maximum visual fidelity" goal, then note it as tech debt.

---

## 4. CMS Content Migration

Three collections were fully captured (schemas + all item data).

| Collection | Items | Fields | Has detail page? |
|---|---|---|---|
| **Projects** | 5 | Title, Project Category (ref), Slug, Video file, Video cover, Image 1–4, Content (richtext), Client, Type, Year, Industry, Director | Yes → `/projects/:slug` |
| **Project category** | 4 (Branded Content, Fashion Film, Music Video, Commercial) | Name | Reference collection only |
| **Blog** | 6 | Cover, Title, Content (richtext), Slug | **No detail page found** in the live project — blog items currently render as cards only (no `/blog/:slug` route exists). Confirm with the user whether individual article pages are wanted in the new site, or whether cards/excerpts-only is intentional. |

**Recommended migration path:** Convert both collections to **local content** rather than a live headless CMS, since Lovable projects work best with self-contained data:
- Option A (simplest, recommended for Lovable): **static JSON/TS data files** (`data/projects.ts`, `data/blog.ts`, `data/categories.ts`) — all 5+6+4 items already captured in full during this audit, ready to transcribe verbatim.
- Option B: MDX per item for richtext bodies (Content fields) if the user wants article-style editing later.
- Option C: real headless CMS (Sanity/Contentful) if ongoing non-developer content editing is required — bigger lift, only recommended if the user explicitly wants CMS-editable content post-migration.

Project data captured (titles/clients for reference): AFTERGLOW (Vertigo, Fashion film), SECOND WIND (Nova Group, Music video), STATIC (9 Dollar, Music video), NIGHT SHIFT (Auriga Motors, Commercial), Vela drink (Vela, Commercial). Blog titles: "Doughnuts in the Lead Role", "Rome in Three Days", "Concrete, Bass and a Single Camera", "Back to Analog", "Welcome Aboard", "The Trophy We Never Entered".

All CMS image/video asset URLs (hosted on `framerusercontent.com`) were captured during the audit. `framerusercontent.com` URLs are stable CDN links and **can be referenced directly** in the new codebase without re-uploading — recommend downloading/mirroring them into the new repo's `/public` (or a Lovable-compatible asset host) for long-term independence from Framer's hosting, rather than depending on Framer's CDN indefinitely.

---

## 5. Component Library → React Mapping

All 22 custom components were inspected (5 in full structural depth, the remainder to variant/prop-schema level, which is sufficient given their simplicity). General pattern: **every component is a Framer "variant" component** (Framer's equivalent of a state machine / props-driven design component), which maps cleanly onto a React component with a `variant` prop and Tailwind conditional classes, or `class-variance-authority` (cva) for state variants — a very natural fit for Lovable/shadcn-style codebases.

| Framer component | Variants | React equivalent | Fidelity notes |
|---|---|---|---|
| **Navigation** | Open/Close | `<Navbar />` with `useState` open/close, blurred backdrop | Auto-close on hover-leave (3s) and on-tap — replicate with `setTimeout`/`onMouseLeave` |
| **Button** | primary / secondary / primary-no-arrow (+hover replicas) | `<Button variant="primary\|secondary\|noArrow" />` | Arrow icon + decorative line as sub-elements; bind to `title/href/newTab/smoothScroll` props |
| **Button Form** | Default/Loading/Disabled/Success/Error (+hover/pressed) | `<SubmitButton status="..." />` | Spinner = conic-gradient + CSS `animation: spin` (replaces Framer's `loopEffect`); **Error state font/color inconsistency — see §3.3** |
| **Tab** | Default/Active | `<Tab active={bool} />` | Simple padding/fill toggle — trivial |
| **Video reveal** | 3-variant scroll/appear reveal | `<VideoReveal />` using Framer Motion `whileInView`/`animate` | 0.5s delay, spring-duration 1.5s transition, collapse via height 0 / -400px offset — map directly to Framer Motion `transition={{ type: "spring", duration: 1.5 }}` |
| **Labels** (client/type/industry/year, ×4) | 5 variants each | `<Label field="client\|type\|industry\|year" />` | Heading + a "collection list" of 5 items at offsets 0–4 bound to the Projects collection — **this is Framer's CMS-repeater pattern for showing related/paginated project fields**; in React this becomes a simple `.map()` over the projects array with a `heading` prop, not 5 separate hardcoded variants |
| **Title Wrapper** | 5 variants | `<TitleWrapper />` | Same CMS-repeater pattern as Labels — 5 variants exist only because Framer needs one canvas variant per CMS-bound index; collapse to a single React component in a `.map()` |
| **Logo** | 1 | `<Logo />` | Static text "ATelier Digital" styled with `Heading 3 normal` |
| **Nav links** | Default (×2 near-duplicates) | `<NavLink href title smoothScroll newTab />` | Links: About, Journal (no link), Contact, Projects |
| **social_media_link** | 1 (×2 dupes) | `<SocialLink icon="linkedin\|vimeo\|instagram" href />` | Icon set = 3 icons total, confirmed |
| **Blog Card** | 1 (×2 dupes) | `<BlogCard author category title number cover numberVisible />` | Image + "Black layer" overlay + bottom title row |
| **detail-card** | 1 | `<DetailCard heading content />` | Simple heading+body pair (used for Client/Type/Year/Industry/Director rows on project detail page) |
| **service_name** | Variant 1 / mobile | `<ServiceRow name description />` | Used in `/services` numbered list |
| **card-team** | horizontal | `<TeamCard image name position />` | About page team grid (confirmed in screenshot: Ryan Gharbi / Hamza Trabelsi) |
| **Client logo card** | 1 | `<ClientLogo logo />` | About page logo grid |
| **Link projects** | Default (×2 dupes) | `<ProjectsLink />` | Simple nav-style link |
| **Project link** | 5 variants | `<ProjectLink />` | Same CMS-repeater pattern as Labels/Title Wrapper — collapse to one component in a `.map()` over Projects, linking to `/projects/:slug` |
| **section-call-to-action** | 1 | `<CTASection />` | "Façonnons quelque chose." + button — appears on every page footer-adjacent |
| **Buy Badges** (local wrapper) | Desktop | *(exclude from migration — see §6)* | Framer-marketplace-template promotional badges ("Get for free" / "More Templates" linking to framer.link and konrad.design) — **not real site content**; flag to user for removal |

**Key structural insight:** Several "5-variant" components (Labels ×4, Title Wrapper, Project link) are not really 5 different visual states — they're Framer's mechanism for repeating a CMS-bound element 5 times with different collection offsets. In React these collapse dramatically: instead of porting 5 near-identical variants each, implement **one component + `.map()`**, which is simpler and more maintainable than the Framer source.

---

## 6. External / Marketplace Components (require re-implementation)

These are Framer marketplace/library components whose internals can't be inspected (opaque to the plugin API) — only their *usage and observed behavior* is known. They must be rebuilt as plain React/CSS equivalents:

| Component | Observed usage | React replacement strategy |
|---|---|---|
| Smooth Scroll | Applied to in-page nav links (`smoothScroll` boolean prop) | Native CSS `scroll-behavior: smooth` + anchor links, or `react-scroll` |
| Video (marketplace player) | Project detail page hero video, home page reveals | Native HTML5 `<video>` with custom controls to match the minimal player UI seen in the "Afterglow" screenshot (play/pause, time, fullscreen, more-options icons) |
| Tab (external variant used elsewhere beyond local `Tab` component) | Category filters on `/projects` and `/services` | Plain controlled tab/filter buttons with `useState` |
| Hamburger menu icon | Nav toggle icon (☰) | Any icon library (Lucide/Heroicons) `Menu`/`X` icon, or custom SVG |
| Arrow icon | Inside Button/Project link components | Same — Lucide `ArrowUpRight` or custom SVG matches screenshots closely |
| Buy Badges (external) | Template promo badges | **Exclude entirely** — confirm with user, these are not site content |

None of these require Framer-specific runtime dependencies — they're all standard, well-solved UI patterns.

---

## 7. Code Overrides

Only one code file exists in the project: **`Examples.tsx`**, containing Framer's **default starter boilerplate** (`withRandomColor`, `withHover`, `withRotate` — generic Framer Motion HOC examples). It is not applied to any inspected component (`codeOverride` attribute was absent everywhere checked) and appears to be unused scaffolding left over from project creation.

**Recommendation:** Exclude from migration — no functional logic to port.

---

## 8. Animation & Interaction Inventory

| Interaction | Where | React/Motion approach |
|---|---|---|
| Nav open/close (`SET_VARIANT`) | Navigation component | `useState` + Framer Motion `AnimatePresence`/`motion.div` variants |
| Auto-close nav on hover-leave (3s delay) | Navigation | `setTimeout` cleared on re-hover |
| Scroll/appear video reveal (`onAppear`, 0.5s delay, spring 1.5s) | Home page project showcase | Framer Motion `whileInView` + `viewport={{ once: true }}` + `transition={{ delay: 0.5, type: "spring", duration: 1.5 }}` |
| Button/Tab hover & pressed replicas | Button, Button Form, Tab | Tailwind `hover:`/`active:` classes, or Motion `whileHover`/`whileTap` |
| Button Form status machine (Default→Loading→Success/Error) | Contact form submit | `useState<'idle'\|'loading'\|'success'\|'error'>` driving conditional render, spinner via CSS `@keyframes spin` on a `conic-gradient` masked circle (exact visual match achievable in pure CSS) |
| Nav background blur | Navigation | `backdrop-filter: blur(...)` |
| Project category filtering | `/projects`, `/services` | Client-side `useState` filter, optionally synced to query params via `useSearchParams` (matches Framer's query-param filter behavior) |

Nothing here requires Framer's proprietary runtime — **Framer Motion** (already the idiomatic choice for Next.js) reproduces every effect observed with high fidelity.

---

## 9. Asset Extraction Plan

- All CMS-bound images/videos are hosted on `framerusercontent.com` with stable long-lived URLs — captured in full for all 5 Projects items (video + video cover + Image 1–4) and all 6 Blog items (cover).
- Non-CMS static images (About page team photos, client logo grid ×12, hero imagery per page) are embedded directly as image fills on frame nodes — same CDN host, extractable by URL.
- Icon set: **3 icons total** (`vimeo`, `instagram`, `linkedin`) from the "Social Media icons" set — trivial to replace with any icon library or inline SVG matching the footer.
- **Recommended action before implementation:** bulk-download all referenced `framerusercontent.com` URLs into the new repo's `/public/images` and `/public/videos`, rewriting references to local paths — removes any dependency on Framer's hosting surviving indefinitely, and is required for a truly "standalone" repo per the user's goal.

---

## 10. Recommended Next.js Project Structure

```
atelier-digital/
├── app/
│   ├── layout.tsx                 # <html>, fonts, global nav+footer shell
│   ├── page.tsx                   # Home
│   ├── about/page.tsx
│   ├── services/page.tsx
│   ├── contact/page.tsx
│   ├── projects/
│   │   ├── page.tsx               # /projects list + category filter
│   │   └── [slug]/page.tsx        # /projects/:slug detail
│   └── not-found.tsx              # /404
├── components/
│   ├── layout/
│   │   ├── Navbar.tsx
│   │   └── Footer.tsx
│   ├── ui/                        # Button, Tab, SubmitButton, Label, TitleWrapper, DetailCard, TeamCard, ClientLogo, BlogCard, ProjectLink, CTASection, SocialLink, NavLink, Logo, ServiceRow
│   └── video/VideoReveal.tsx
├── data/
│   ├── projects.ts                # 5 items, typed
│   ├── blog.ts                    # 6 items, typed
│   └── categories.ts              # 4 items
├── lib/
│   └── types.ts                   # Project, BlogPost, Category types
├── public/
│   ├── images/                    # downloaded framerusercontent.com assets
│   └── videos/
├── styles/
│   └── globals.css                # design tokens as CSS vars, font-feature-settings for stylistic sets
├── tailwind.config.ts              # colors, breakpoints (390/810/1200), text presets as fontSize scale
└── package.json                    # next, react, framer-motion, tailwindcss
```

This structure is directly compatible with Lovable's Vite/Next.js import conventions and is clean to push to GitHub as-is.

---

## 11. Open Questions for the User (before implementation begins)

1. **Blog detail pages:** no `/blog/:slug` route exists in the source — should the new site add individual blog article pages, or keep blog as cards-only (matching current behavior)?
2. **Buy Badges component:** appears to be leftover Framer-template promotional content (links to `framer.link` / `konrad.design`), not real site content — confirm it should be dropped.
3. **Button Form Error state inconsistency** (§3.3): replicate the hardcoded red/Geist Mono exactly, or normalize to the site's design tokens?
4. **Contact form backend:** no server wiring was found in the Framer source (fields only) — do you want the new form to actually submit somewhere (email service, API route), or should it remain presentational until you wire it up?
5. **CMS strategy:** static data files (fastest, recommended for Lovable) vs. a real headless CMS for future non-developer content edits?

---

## 12. Suggested Implementation Order (once this plan is approved)

1. Scaffold Next.js + Tailwind + Framer Motion project, wire up fonts and design tokens.
2. Build layout shell (Navbar + Footer) and confirm pixel-match against the Home/About screenshots captured during this audit.
3. Build shared `ui/` component library (Button, Tab, Label, Card variants) as isolated, storybook-able components.
4. Migrate CMS data to typed `data/*.ts` files; download and localize all image/video assets.
5. Build out the 7 pages in order of complexity: 404 → Contact → Services → Projects list → Project detail → Home → About.
6. Layer in animations/interactions (nav, video reveal, form states, filters) last, once static layout fidelity is confirmed.
7. Cross-check every page against the captured screenshots at all 3 breakpoints before calling migration complete.

---

*This plan is based on a complete, read-only audit of the live Framer project (all pages, all 22 components, all 3 CMS collections and their items, all style tokens/typography, the one code file, and full-page screenshots at Desktop width). No changes were made to the Framer project during this audit.*
