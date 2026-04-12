# DMZ Scuba Live Site

This repository contains the live DMZ Scuba website deployed to Cloudflare Pages, plus the Cloudflare Worker API that supports media management, destination management, event data, contact delivery, and admin authentication.

It is the production-facing repository for the live site, not the primary day-to-day development workspace.

## What This Project Is

DMZ Scuba is a custom multi-page marketing and operations site for a scuba business. It combines:

- A static front end served by Cloudflare Pages
- A Cloudflare Worker API for dynamic/admin features
- A D1 database for site-managed content
- Cloudflare Stream and Cloudflare Images integrations for media handling
- Resend-powered email delivery for contact and interest flows

The current site includes public pages for:

- Home
- Training
- Travel
- Media
- Events
- Contact
- About
- Thank-you / post-submit states
- Destination detail pages
- Interactive training demos

## Live vs Dev

This repo is the live project:

- Workspace: `H:/dmz-scuba-live`
- GitHub repo: `DMZScuba-live`
- Branch: `main`
- Cloudflare Pages project: `dmzscuba-live`

The separate development project lives in:

- `H:/dmz-scuba site`

That split is intentional. Development happens in the dev repo first, and approved work is promoted into this live repo.

## Architecture

### Front End

The front end is a static site built with plain HTML, CSS, and JavaScript rather than a framework. The site favors direct file-based routing and lightweight client-side behavior.

Key characteristics:

- Multi-page site structure under `pages/`
- Shared JS/CSS for reusable behavior
- Page-specific scripts for admin flows and feature-heavy sections
- Cloudflare Pages hosting
- `_redirects` rules for domain canonicalization and Worker API proxying

Notable public feature areas:

- Travel globe and destination system
- Media library with gallery and reel-style browsing
- Events calendar and event detail views
- Training landing pages and interactive demos
- Contact and inquiry capture flows

### Worker API

The Worker lives in:

- `workers/dmz-media-api/src/index.js`

It currently provides:

- Admin authentication with session tokens
- Media CRUD and bulk publish APIs
- Cloudflare Stream direct upload and tus upload helpers
- Cloudflare Images direct upload and delete helpers
- Public and admin destination v2 APIs
- Public and admin events v2 APIs
- Event registration APIs
- Home ticker and other site-managed content endpoints
- Contact form delivery through Resend
- Interest-list and inquiry auto-reply handling
- Lightweight client telemetry intake

The Worker is configured with:

- D1 database binding: `DB`
- Worker config: `workers/dmz-media-api/wrangler.toml`
- Schema: `workers/dmz-media-api/schema.sql`

## Repository Layout

Top-level structure:

- `index.html`
  - Home page
- `pages/`
  - Main public sections and destination/event detail pages
- `js/`
  - Shared and page-specific client logic
- `css/`
  - Shared and page-specific styling
- `assets/`
  - Data files, icons, images, thumbnails, and media-related assets
- `workers/dmz-media-api/`
  - Cloudflare Worker API source, schema, and Wrangler config
- `functions/`
  - Pages Functions directory used alongside Pages routing
- `tests/`
  - Targeted regression test scripts
- `scripts/`
  - Utility/support scripts

Operational helper files in repo root:

- `Deploy Worker.bat`
- `Python Server.bat`
- `Smoke Check.bat`
- `Test Media Logic.bat`
- `RELEASE-CHECKLIST.md`

## Routing and Hosting Notes

Cloudflare Pages routing behavior is partly defined by `_redirects`.

Current important rules:

- Canonical redirect to `https://www.dmzscuba.com`
- Same-origin `/api/*` requests proxied to the Worker deployment
- Friendly `/training` route mapped to the training landing page

That means the front end can call `/api/...` while the actual backend runs in the Worker.

## Local Development / Review

This repo does not use a root framework dev server or a root `package.json`.

Typical local review flow:

1. Serve the static site locally.
2. Open the relevant page in a browser.
3. If needed, deploy the Worker separately so API-backed features match production behavior.

Practical options already in the repo:

- `Python Server.bat`
  - Quick local static file server
- `Smoke Check.bat <url>`
  - Basic smoke verification against a deployed target
- `Test Media Logic.bat`
  - Targeted test helper for media logic changes

## Testing

Testing is lightweight and targeted rather than comprehensive.

Current state:

- One targeted Node test exists in `tests/media-logic.test.cjs`
- Operational smoke checks are documented in `RELEASE-CHECKLIST.md`
- Most regression checking is still manual

This is one of the project's main technical risks: the site is functional, but automated coverage is limited.

## Deployment

### Front End

The front end deploys through the live Cloudflare Pages project tied to this repo.

### Worker

The Worker can be deployed from the repo root with:

```powershell
./Deploy Worker.bat
```

That runs Wrangler deploy for:

- `workers/dmz-media-api`

## Production Integrations

The live site currently depends on several platform services:

- Cloudflare Pages
- Cloudflare Worker
- Cloudflare D1
- Cloudflare Stream
- Cloudflare Images
- Resend

Environment and secret configuration is handled through Wrangler/Cloudflare, not hardcoded in the repo.

Examples of configured variables include:

- `ALLOWED_ORIGINS`
- `RESEND_FROM_EMAIL`
- `RESEND_FROM_NAME`
- `RESEND_TO`
- `ADMIN_USER`
- `ADMIN_PASS`
- `CF_ACCOUNT_ID`
- `CF_STREAM_TOKEN`
- `CF_IMAGES_TOKEN`
- `CF_IMAGES_DELIVERY`
- `CF_IMAGES_VARIANT`

## Current Technical Shape

The live site is beyond prototype stage and supports real content operations, but it is still actively refined.

Strong points:

- Broad feature coverage without a heavyweight stack
- Real admin tooling for media, destinations, events, and managed content
- Practical integrations for uploads, images, email, and registrations
- Production deploy path is already in place
- Public site architecture is understandable and file-based

Current limitations and debt:

- Limited automated test coverage
- Some docs, especially Worker-specific docs, lag behind the current endpoint surface
- A mix of legacy and newer admin patterns still exists in some sections
- Some page logic is dense because behavior has grown incrementally over time

## Why This Project May Be Interesting To Review

From an engineering standpoint, this project shows:

- Custom front-end architecture without framework dependency
- Real-world integration with multiple Cloudflare platform products
- Admin tooling built directly into a static-site architecture
- A practical split between public content, admin flows, and operational deployment concerns
- Ongoing refactoring of live business software under production constraints

## Useful Files For Reviewers

If someone wants to understand the project quickly, the best starting points are:

- `README.md`
- `index.html`
- `pages/travel/index.html`
- `pages/media/index.html`
- `pages/events/index.html`
- `js/events.js`
- `js/media.js`
- `js/travel-admin.js`
- `workers/dmz-media-api/src/index.js`
- `workers/dmz-media-api/README.md`
- `workers/dmz-media-api/schema.sql`
- `RELEASE-CHECKLIST.md`

## Status

This README reflects the live repo state as reviewed on April 12, 2026.
