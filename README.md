# DMZ Scuba Live Site (`dmz-scuba-live`)

Production website for DMZ Scuba, deployed on Cloudflare Pages with a Cloudflare Worker backend for media, destinations, and form delivery.

This README is the live-site operational source of truth for architecture, routing, deployment, maintenance, and SEO/indexing status.

## 1) Environment

- Live repo path: `H:/dmz-scuba-live`
- Live GitHub repo: `DMZScuba-live`
- Live branch: `main`
- Live Cloudflare Pages project: `dmzscuba-live`
- Production domain: `https://www.dmzscuba.com`
- Separate dev repo (do not mix work unintentionally): `H:/dmz-scuba site`

## 2) Architecture Overview

- Frontend: static multi-page HTML/CSS/JS
- Hosting/CDN: Cloudflare Pages
- API backend: Cloudflare Worker (`workers/dmz-media-api`)
- Database: Cloudflare D1 (`dmz_media`)
- Video stack: Cloudflare Stream
- Image stack: Cloudflare Images
- Email stack: Resend (contact + quiz lead notifications)

## 3) Routing and Request Flow

- Homepage: `/` -> `index.html`
- Training shortcut: `/training` -> `/pages/training/index.html` (configured in `_redirects`)
- API reverse proxy in `_redirects`:
- `/api/*` -> `https://dmz-media-api.zacharylisowski55.workers.dev/api/:splat` (status `200`)
- Pages Function proxy also exists at `functions/api/[[path]].js` and forwards `/api/*` to the same Worker target.

## 4) Repository Structure

- `index.html`: homepage, primary CTA ladder, quiz entry points, footer
- `pages/`:
- `pages/training/`: training landing + course pages + interactive tools
- `pages/training/interactive-tools/color-loss-demo.html`: underwater color-loss demo with camera mode support
- `pages/travel/`: travel globe, destination preview/list, destination detail
- `pages/media/`: media library, reel mode, media admin UI
- `pages/contact/`: inquiry forms
- `pages/about/`: about/brand page
- `pages/thanks/`: post-submit confirmation
- `quiz/index.html`: redirect stub to homepage quiz modal (`/index.html?openQuiz=quick#dive-quiz-title`)
- `js/`:
- `main.js`: shared UI helpers, form submission, telemetry, dropdowns, mobile header behavior
- `quiz.js`: recommendation engine + results + quiz lead capture
- `media.js`, `media-edit.js`, `media-logic.js`: media browsing/admin/edit logic
- `globe.js`, `travel-admin.js`, `destination.js`, `destinations-edit.js`: travel globe and destination admin/editor logic
- `css/`: base, components, page styles, responsive layers
- `assets/data/`: `media.json`, `destinations.json`, `destinations-expanded.json`
- `workers/dmz-media-api/`: Worker API source/config/docs
- `scripts/smoke-check.mjs`: site/API smoke checks
- `tests/media-logic.test.cjs`: media logic validation
- Utility scripts: `Deploy Worker.bat`, `Smoke Check.bat`, `Test Media Logic.bat`, `Pcloudfare.bat`, `push.bat`

## 5) Primary Site Systems

- Training system: course pages and interactive demo experiences under `pages/training/**`
- Travel system: interactive globe + destination content + admin editing/publish controls
- Media system: searchable/filterable media grid, reel mode, admin media editor + publish workflow
- Contact system: unified form submission through `/api/contact`
- Quiz system: homepage modal quiz flow (with direct quiz route redirect via `/quiz`)

## 6) Worker API (Live Backend)

Code and config:

- `workers/dmz-media-api/src/index.js`
- `workers/dmz-media-api/wrangler.toml`

Public endpoints (current core):

- `GET /api/media`
- `POST /api/contact`
- `POST /api/client-telemetry`
- `GET /api/v2/destinations`
- `GET /api/v2/destinations/:id`

Admin endpoints (authenticated via bearer token):

- `POST /api/admin/login`
- media CRUD + bulk publish endpoints
- Stream direct/TUS upload helpers
- Stream date sync
- Images direct upload/delete
- destination v2 upsert/delete

Reference backend docs: `workers/dmz-media-api/README.md`

## 7) SEO and Indexing

- Canonical production host: `https://www.dmzscuba.com`
- `robots.txt` includes:
- `Sitemap: https://www.dmzscuba.com/sitemap.xml`
- `sitemap.xml` is valid XML and includes core URLs, including About and Quiz pages.

Search Console status:

- Sitemap submission: successful
- Index request quota was reached before submitting all priority URLs
- Pending manual indexing requests (still required):
- `https://www.dmzscuba.com/pages/about/index.html`
- `https://www.dmzscuba.com/quiz/index.html`

## 8) Local Run and Validation

- Serve site locally on port `8080` (or any static server)
- Optional tunnel helper: `Pcloudfare.bat` (`cloudflared tunnel --url http://localhost:8080`)

Smoke checks:

- Default command: `Smoke Check.bat`
- Live-target command: `Smoke Check.bat https://www.dmzscuba.com`
- Checks include core page loads and API sanity calls.

Media logic test:

- Run: `Test Media Logic.bat`
- Executes: `tests/media-logic.test.cjs`

## 9) Deployment Workflows

Site deploy (Cloudflare Pages):

1. Make edits in `H:/dmz-scuba-live`.
2. Commit on `main`.
3. Push `origin main`.
4. Cloudflare Pages auto-deploys the new live build.

Worker deploy (API backend):

- Run `Deploy Worker.bat`.
- This runs `npx wrangler deploy` inside `workers/dmz-media-api`.

## 10) Operational Rules

- Keep live and dev repos separate.
- Do not modify dev when working in live.
- Avoid destructive git operations for routine corrections.
- Use `git revert` for rollback when needed.

## 11) Known Follow-Ups

- Expand automated coverage beyond smoke + targeted logic tests.
- Clean remaining text-encoding artifacts where present.
- Keep this README and Worker docs in sync with endpoint/config changes.
- Submit pending Search Console indexing requests for About and Quiz when quota allows.

## 12) Project Development Summary (Past ~3.5 Weeks)

From January 26, 2026 to February 12, 2026, the site moved from initial structure to a launch-ready live platform:

- Week 1 (Jan 26-Jan 31): initial site foundation, media/admin backend wiring, contact flow setup, and first major page/content builds.
- Week 2 (Feb 1-Feb 7): major travel and destination system expansion, including v2 API migration, robust admin editing flows, Cloudflare Images integration, and reliability fixes for save/publish workflows.
- Week 3 (Feb 8-Feb 12): media reel/mobile polish, interactive tools camera-mode rollout, training-partner clarity updates, map-link behavior refinement for desktop/mobile, and SEO/indexing hardening (`robots.txt`, `sitemap.xml`, canonical/metadata alignment).
- Current status: live architecture is stable with production deployment flow in place; remaining post-launch housekeeping includes Search Console indexing requests for About and Quiz pages once quota resets.

## 13) Command Quick Reference

- Status: `git -C "H:/dmz-scuba-live" status --short --branch`
- Push: `git -C "H:/dmz-scuba-live" push origin main`
- Smoke check (live): `H:/dmz-scuba-live/Smoke Check.bat https://www.dmzscuba.com`
- Deploy Worker: `H:/dmz-scuba-live/Deploy Worker.bat`
