# CeSeA Agenc.ia — website

Editorial, motion-driven marketing site for a Spanish agency, built on Next.js 15 and TypeScript. Scroll choreography done by hand, a typographic system that carries the brand, and a backend that treats every inbound lead as untrusted input.

**Live:** https://ceseagencia.com

![Homepage hero](assets/01-home-hero.webp)

---

## What it is

CeSeA Agenc.ia is a marketing agency working around one idea: reaching people through the generational transition, with human craft driving the AI in the loop rather than the other way round. The site had to sell that positioning before a single line of copy was read, so the whole thing is built as one continuous editorial piece — a scroll-scrubbed video hero, chapters that read like a magazine spread, and case work presented the way a streaming service presents its catalogue.

I designed and built it end to end: information architecture, the design system, every animated section, the SEO and structured-data layer, and the server side that turns a form submission into a notified, stored, deduplicated lead. It ships as a statically generated Next.js app — dozens of pre-rendered routes — with a small set of server routes for anything that touches a secret.

The brand voice is Spanish and unapologetically editorial, so the engineering had to stay invisible: no layout shift, no janky scrolljacking, motion that steps aside the moment a visitor asks it to.

## Screenshots

<table>
  <tr>
    <td width="50%"><img src="assets/02-home-services.webp" alt="Services scroll track"><br><sub><b>Home · services.</b> A horizontal scroll-track that pins and pans through the nine service lines.</sub></td>
    <td width="50%"><img src="assets/05-sector-salud-dental.webp" alt="Dental sector landing"><br><sub><b>Sector landing.</b> Dark editorial hero with the flip-cards that frame each vertical as <i>reto / abordaje</i>.</sub></td>
  </tr>
  <tr>
    <td width="50%"><img src="assets/06-casos-grid.webp" alt="Case index grid"><br><sub><b>Case index.</b> Poster grid, one tile per vertical, art-directed rather than templated.</sub></td>
    <td width="50%"><img src="assets/07-caso-detalle.webp" alt="Case study detail"><br><sub><b>Case study.</b> A reusable template: challenge, approach timeline, process visual, results.</sub></td>
  </tr>
  <tr>
    <td width="50%"><img src="assets/04-sectores.webp" alt="Sectors index"><br><sub><b>Sectors.</b> Type-led index built on the display serif.</sub></td>
    <td width="50%"><img src="assets/08-metodologia.webp" alt="Methodology page"><br><sub><b>Methodology.</b> Three phases, marked up as <code>HowTo</code> structured data.</sub></td>
  </tr>
  <tr>
    <td width="50%"><img src="assets/03-home-diferenciadores.webp" alt="Differentiators section"><br><sub><b>Home · differentiators.</b> Colour-blocked cards over the cream base.</sub></td>
    <td width="50%"><img src="assets/09-servicio-seo.webp" alt="Service detail page"><br><sub><b>Service detail.</b> Deliverables laid out as hover-to-expand cards.</sub></td>
  </tr>
</table>

## Stack

| Layer | Choice |
|---|---|
| Framework | Next.js 15 (App Router), React 19 |
| Language | TypeScript |
| Styling | Tailwind CSS v4 — tokens declared in a `@theme` block, no `tailwind.config` |
| Motion | Framer Motion, Lenis (smooth scroll) |
| Type | Lora (display), Montserrat (body), Poppins (labels), self-hosted via `next/font` |
| Forms | React Hook Form + Zod |
| Backend | Next.js route handlers, Nodemailer (SMTP), Supabase, Notion API |
| Delivery | Standalone build, containerised, self-hosted behind a reverse proxy |

## Architecture

**Rendering & routing.** Everything the public sees is statically generated. Services, sectors and case studies are dynamic segments resolved at build time through `generateStaticParams`, so a new case study is a data entry, not a new page. The only server code that runs per request is the handful of route handlers behind the forms — the surface that needs a secret is kept deliberately small.

**Motion.** The hero is a scroll-scrubbed frame sequence pinned across several viewport heights; scroll position drives playback, with a video fallback. Below it, a velocity-tied marquee, full-bleed scroll chapters, word-by-word reveals and the flip-cards all hang off Lenis and Framer Motion. Two rules kept it honest: no animation may cause layout shift, and everything checks `prefers-reduced-motion` and switches off cleanly.

**Design system.** Three typefaces, a fixed palette, and spacing/colour tokens declared once in a Tailwind v4 `@theme` block. Sections compose from the same primitives, which is why a new sector landing or case study looks designed rather than assembled.

**SEO & structured data.** This is a marketing agency, so the site has to practise what it sells. It emits a full JSON-LD graph — `Organization` (with each office modelled as a `LocalBusiness`, geo and opening hours), `WebSite` with a `SearchAction`, `Service` catalogues, `FAQPage`, `HowTo`, `BreadcrumbList`, `Article` and `Review` — cross-linked by stable `@id`s. Sitemap and robots are generated in code; metadata, canonicals and Open Graph are set per route. Images are served as AVIF/WebP through `next/image`.

## Lead capture, handled like untrusted input

The forms are the one place a stranger can reach the backend, so they're built defensively.

- **Method-locked and schema-validated.** Endpoints accept `POST` only and parse the body through Zod before anything else touches it — explicit types, bounded string lengths, and a consent field that must be exactly `true`. Anything off-shape is a 400.
- **Rate limited.** Per-IP limiting (client form and internal endpoint on separate budgets) returns `429` past the threshold, with the IP taken from the forwarded headers the proxy sets.
- **Output encoding.** Every dynamic value is HTML-escaped, and the rendered email body is then normalised to numeric entities so accented Spanish survives whatever mail client opens it. The JSON-LD serialiser escapes `<` so structured data can't break out of its `<script>` tag.
- **Secrets stay server-side.** SMTP, Supabase and Notion credentials live only in environment variables. The Supabase service-role key never leaves the server. Nothing sensitive is shipped to the client bundle.
- **Fails soft.** Notification email, confirmation email and datastore writes run under `Promise.allSettled`; if one degrades, the visitor still gets a success response and the lead is never dropped.

## Hardening backlog

Written down because a security review that only lists what's already done isn't a review:

- Security response headers at the edge — CSP, HSTS, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`. The app-layer input/output hardening above is in place; the header layer is the next pass.
- Move rate-limit state to a shared store (Redis) so the limits hold across more than one instance.

## Performance & accessibility

Static delivery keeps TTFB low; fonts are self-hosted and preloaded to avoid a third-party round-trip; images are responsive AVIF/WebP. Motion is opt-out via `prefers-reduced-motion`, focus states are visible throughout, and structure stays semantic so the page reads without JavaScript.

---

Designed and built by **Daniel Brosed** — developer and security auditor.
[danielbrosed.com](https://danielbrosed.com) · [LinkedIn](https://www.linkedin.com/in/danielbrosed/)

The site and its content are © CeSeA Agenc.ia. This repository documents the build as a portfolio case study; the screenshots are of the production site and the application source is not included. See [`LICENSE`](LICENSE).
