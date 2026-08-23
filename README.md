# DMZScuba.com <!-- comprehensive production reference -->

![DMZ Scuba logo](assets/images/logos/dmz-scuba-logo-display.webp)

DMZScuba.com is a mobile-first, full-stack platform for a working scuba training and travel business. It combines a public marketing site, training catalog, interactive dive-planning tools, travel explorer, event calendar, media library, customer inquiry system, and an authenticated business-management console in one deliberately framework-free codebase.

The public experience is static HTML, CSS, and vanilla JavaScript deployed on Cloudflare Pages. Dynamic content and business operations are provided by a Cloudflare Worker, a Cloudflare D1 database, Cloudflare Stream, Cloudflare Images, and Resend. There is no browser-side framework, package manager, bundler, or compilation step.

**Production:** [www.dmzscuba.com](https://www.dmzscuba.com/)<br>
**Production management console:** [www.dmzscuba.com/management/](https://www.dmzscuba.com/management/)<br>
**Live Cloudflare Pages project:** [dmzscuba-live.pages.dev](https://dmzscuba-live.pages.dev/)

> This repository is the production repository. Day-to-day feature development happens in the separate `DMZScuba.com` development repository and is promoted here only after validation and explicit approval.

---

## Table of contents

1. [What this project does](#what-this-project-does)
2. [Technical highlights](#technical-highlights)
3. [System architecture](#system-architecture)
4. [Technology stack](#technology-stack)
5. [Repository map](#repository-map)
6. [Public-site architecture](#public-site-architecture)
7. [Public experiences, page by page](#public-experiences-page-by-page)
8. [Mobile experience](#mobile-experience)
9. [Training and conversion system](#training-and-conversion-system)
10. [Travel and destination system](#travel-and-destination-system)
11. [Events and registration system](#events-and-registration-system)
12. [Media platform](#media-platform)
13. [Quiz, forms, and CRM handoff](#quiz-forms-and-crm-handoff)
14. [Management console](#management-console)
15. [Frontend module reference](#frontend-module-reference)
16. [Cloudflare Worker](#cloudflare-worker)
17. [API reference](#api-reference)
18. [D1 data model](#d1-data-model)
19. [Static data and fallback behavior](#static-data-and-fallback-behavior)
20. [Authentication and security model](#authentication-and-security-model)
21. [Email system](#email-system)
22. [Design system and CSS architecture](#design-system-and-css-architecture)
23. [Accessibility](#accessibility)
24. [SEO and discoverability](#seo-and-discoverability)
25. [Performance and resilience](#performance-and-resilience)
26. [Configuration and secrets](#configuration-and-secrets)
27. [Local development](#local-development)
28. [Testing and release validation](#testing-and-release-validation)
29. [Deployment model](#deployment-model)
30. [Common extension workflows](#common-extension-workflows)
31. [Operational boundaries and technical debt](#operational-boundaries-and-technical-debt)
32. [Project philosophy](#project-philosophy)

---

## What this project does

DMZScuba.com supports the complete path from first visit to ongoing customer relationship:

```text
Discovery
  -> course, destination, event, media, or quiz
  -> context-specific call to action
  -> contact request, interest signup, or event registration
  -> automatic email and management record creation
  -> staff follow-up in the management console
  -> class, trip, calendar, and media updates
  -> published changes on the public site
```

The site is not only a brochure. It is an integrated operating system for the business:

- **Marketing and discovery:** home page, structured navigation, SEO metadata, social previews, mobile calls to action, and guided funnels.
- **Scuba training:** course catalog, course-specific landing pages, group/private options, referral paths, specialty programs, skill refresh, and educational simulations.
- **Travel discovery:** an interactive canvas globe, searchable destination cards, detailed destination pages, live trip-status indicators, destination media, and interest-list capture.
- **Events:** native calendar and list views, type filtering, shareable event links, capacity-aware registration, registration approval, and attendee email workflows.
- **Media:** filterable photo/video library, destination-aware discovery, Cloudflare Stream and YouTube playback, full-screen Reel Mode, and authenticated publishing tools.
- **Customer acquisition:** adaptive Dive Path Quiz, contact forms, course requests, travel-interest forms, event-alert subscriptions, automatic replies, and CRM record creation.
- **Business operations:** contacts, inquiries, classes, tasks, trips/calendar records, registration escrow, bulk actions, CSV import/export, dashboards, content studios, and mobile management.

---

## Technical highlights

- **Zero-build frontend:** 29 HTML documents, 28 JavaScript modules, and 16 stylesheets load directly in the browser.
- **Framework-free application architecture:** large application surfaces use plain state objects, DOM events, data attributes, observers, and isolated IIFEs.
- **Single API boundary:** browser requests use same-origin `/api/*`; a Pages Function forwards them to one Worker.
- **Serverless persistence:** D1 stores media, events, registrations, destinations, site settings, admin sessions, management records, and allowlisted training-funnel events.
- **Direct-to-cloud uploads:** the browser uploads video directly to Cloudflare Stream and images directly to Cloudflare Images using authenticated one-time upload URLs.
- **Pure Canvas globe:** the travel globe implements sizing, rotation, inertia, texture rendering, search, filtering, hit testing, pins, touch gestures, and zoom without Three.js or WebGL.
- **Two event representations:** recurring definitions are stored compactly and expanded into dated event instances for public rendering.
- **Integrated CRM capture:** public inquiries and quiz results become management records automatically after successful delivery.
- **Mobile-first management:** the desktop sidebar becomes a bottom-tab application shell; record editing becomes a bottom sheet with contextual floating save behavior.
- **Graceful fallback:** selected public content can fall back to versioned JSON when live API data cannot be loaded.
- **Automated structural tests:** Node tests protect mobile navigation, training funnels, API-facing UI contracts, media logic, and CSS block integrity.

---

## System architecture

### Runtime topology

```text
User browser
  |
  +-- Cloudflare Pages: www.dmzscuba.com
  |     |
  |     +-- static HTML
  |     +-- global and page-specific CSS
  |     +-- vanilla JavaScript modules
  |     +-- static images, icons, sprites, and fallback JSON
  |     +-- /management/ authenticated console
  |     |
  |     `-- /api/*
  |           |
  |           +-- _redirects proxy rule
  |           `-- functions/api/[[path]].js
  |                 |
  |                 `-- dmz-media-api Worker
  |                       |
  |                       +-- D1: dmz_media
  |                       +-- Cloudflare Stream API
  |                       +-- Cloudflare Images API
  |                       `-- Resend API
  |
  +-- YouTube embeds and thumbnails
  `-- HLS.js loaded on demand when native HLS is unavailable
```

### Public read flow

```text
Page loads
  -> feature JavaScript requests same-origin /api/...
  -> Pages forwards request to Worker
  -> Worker reads D1 and normalizes data
  -> JSON response returns with CORS headers
  -> page renders live data
  -> selected features use static JSON if the API is unavailable
```

### Authenticated write flow

```text
Admin login
  -> POST /api/admin/login
  -> Worker validates ADMIN_USER and ADMIN_PASS
  -> UUID session stored in D1 for 24 hours
  -> token returned to browser and stored as dmzMediaToken
  -> Authorization: Bearer <token> on protected requests
  -> Worker checks admin_sessions before every protected operation
```

### Direct media-upload flow

```text
Admin selects a file
  -> browser requests an authenticated one-time upload URL
  -> Worker asks Cloudflare Stream or Images for that URL
  -> browser uploads the file directly to the media service
  -> browser creates/updates media metadata through the Worker
  -> public media API returns the published item
```

The media bytes do not pass through the Worker. This keeps Worker execution small and avoids using the API process as a large-file relay.

---

## Technology stack

| Layer | Technology | Responsibility |
|---|---|---|
| Markup | Semantic HTML5 | Page structure, forms, dialogs, static content, SEO |
| Styling | Handwritten CSS | Tokens, components, page themes, responsive behavior |
| Client logic | Vanilla ES6+ JavaScript | Rendering, state, navigation, editors, uploads, simulations |
| Hosting | Cloudflare Pages | Static hosting, deployment, redirects, custom domain |
| Edge proxy | Cloudflare Pages Function | Same-origin forwarding from `/api/*` to the Worker |
| API | Cloudflare Worker | Auth, validation, CRUD, integrations, email orchestration |
| Database | Cloudflare D1 / SQLite | All dynamic business and public content |
| Video | Cloudflare Stream | Video upload, processing, thumbnailing, HLS/delivery |
| Images | Cloudflare Images | Direct image upload and delivery variants |
| Email | Resend | Internal notifications and customer-facing transactional email |
| External media | YouTube | Supported video source and embed playback |
| Tests | Node.js `node:test` | Logic and structural regression testing |

There is no root `package.json`, no frontend dependency installation, and no production build command. Node is used for tests and deployment tooling; Wrangler is invoked through `npx` for Worker deployment.

---

## Repository map

```text
.
|-- index.html                         Home page and quiz host
|-- management/
|   `-- index.html                    Authenticated operations console
|-- quiz/
|   `-- index.html                    Standalone quiz entry
|-- pages/
|   |-- about/
|   |-- contact/
|   |-- events/
|   |   |-- index.html                Public events page
|   |   |-- event.html                Shareable event detail page
|   |   `-- embed.html                Calendar/admin embed surface
|   |-- media/
|   |-- nfc/
|   |-- privacy/
|   |-- thanks/
|   |-- training/                     Training hub and course subtree
|   `-- travel/
|       |-- index.html                Globe and destination browser
|       `-- destination.html          Data-driven destination detail
|-- js/                               28 browser modules
|-- css/
|   |-- base.css                      Tokens and foundations
|   |-- components.css                Shared components and mobile drawer
|   |-- main.css                      Global layout and site sections
|   |-- responsive.css                Unified mobile behavior
|   `-- pages/                        Page-specific styles
|-- assets/
|   |-- data/                         JSON fallbacks and seeded content
|   |-- images/                       Heroes, destinations, logos, globe assets
|   |-- media/                        Local media and thumbnails
|   |-- icons/                        Favicons
|   `-- contact/dmz-scuba.vcf         Downloadable contact card
|-- functions/api/[[path]].js         Pages-to-Worker API proxy
|-- workers/dmz-media-api/
|   |-- src/index.js                  Worker implementation
|   |-- schema.sql                    D1 schema reference
|   |-- migrations/                   Numbered D1 migrations
|   |-- wrangler.toml                 Worker and D1 configuration
|   `-- README.md                     Worker-focused reference
|-- tests/
|   |-- media-logic.test.cjs
|   |-- mobile-experience.test.cjs
|   `-- training-funnel.test.cjs
|-- scripts/smoke-check.mjs           Deployed-site smoke checks
|-- _redirects                        Domain, API, and convenience routes
|-- _headers                          Special response headers
|-- robots.txt
|-- sitemap.xml
|-- RELEASE-CHECKLIST.md
|-- AGENTS.md                         Repository conventions for coding agents
`-- *.bat                             Windows development/deployment helpers
```

---

## Public-site architecture

### Shared page shell

Public pages include a minimal `<header class="site-header" data-site-nav>`. `js/main.js` replaces or hydrates that contract with the same navigation everywhere:

- DMZ Scuba logo and visible brand name
- desktop primary navigation
- mobile three-bar menu trigger
- viewport-level backdrop and slide-out navigation drawer
- active-page state
- keyboard focus management
- Escape-to-close behavior
- swipe-to-close behavior
- body scroll locking while the drawer is open

The mobile drawer is mounted at the end of `<body>`, outside page-specific stacking contexts. This prevents transformed heroes, filters, or page sections from clipping the menu.

### Shared JavaScript contracts

Most modules are IIFEs:

```js
(() => {
  "use strict";
  // private module state and behavior
})();
```

Intentional globals are limited. `main.js` exposes:

- `window.DMZForms` for consistent form serialization and submission
- `window.DMZTelemetry` for small client-failure and CTA event reports

Modules primarily coordinate through DOM contracts:

- `data-*` selectors identify behavior and content targets.
- classes represent visual state.
- `MutationObserver`, `ResizeObserver`, and `IntersectionObserver` respond to rendered state without a module loader.
- custom events and native click/change/input events keep feature modules loosely coupled.

### Shared form behavior

`main.js` provides a common submission path for contact-style forms:

1. Serialize normal fields and selected custom controls.
2. include form name, subject, page URL, and message context.
3. submit JSON to `/api/contact`.
4. show pending/success/error feedback.
5. report failures to `/api/client-telemetry`.
6. redirect to the thank-you page where configured.

Query parameters can prefill course, interest, location, name, and related fields. Prefilled controls receive a visible state, while required empty controls can receive a `needs-input` cue.

---

## Public experiences, page by page

### Home: `/`

The home page establishes the main journey around **Train. Travel. Explore.** It contains:

- mobile/desktop responsive hero imagery
- primary training call to action
- live homepage ticker from `/api/v2/home-ticker`
- three discovery lanes for training, travel, and media
- upcoming-event preview using the shared events engine
- public event-alert subscription
- adaptive Dive Path Quiz in an accessible modal
- trust and positioning content
- contact conversion section
- device-aware map links: Apple Maps on iOS, `geo:` on Android, Google Maps on desktop
- a dismissible mobile action bar that suppresses itself when an equivalent in-page action is visible

Ticker loading is fail-soft: the area remains hidden when no usable API or fallback content is available.

### About: `/pages/about/`

Explains the business mission, personalized instruction philosophy, founder, experience highlights, and call to action. It uses the same shared navigation, responsive hero system, footer, and contact routing as the rest of the site.

### Contact: `/pages/contact/`

Provides three acquisition paths:

- a short quick-contact form
- event-alert subscription
- a detailed Dive Now planning form

The planning form captures diving interest, experience, location, course, timing, group information, and open-ended goals. URL-prefill support lets the quiz, training pages, travel pages, and CTAs hand context into the form instead of making the visitor repeat it.

### Thank you: `/pages/thanks/`

Confirms form completion and keeps the visitor within the site journey. This page is intentionally simple and shares the standard site shell.

### Privacy: `/pages/privacy/`

Documents site form, analytics/advertising, and privacy practices. The page is indexed and included in shared footer navigation.

### NFC landing page: `/nfc/`

A compact, phone-first landing page intended for physical NFC cards. It provides:

- downloadable DMZ Scuba vCard
- click-to-call and email actions
- social profile access
- key training/travel/media links
- current opportunities populated from the event system

### Standalone quiz: `/quiz/`

Provides a direct route into the Dive Path Quiz. The same `quiz.js` logic also runs inside the home-page modal, so recommendations and capture behavior stay consistent.

---

## Mobile experience

Mobile is treated as a first-class product surface rather than a compressed desktop layout.

### Unified header behavior

- A single 64-pixel mobile header contract applies across public shell pages.
- The header remains visible at the start of the page, then hides after deliberate downward scrolling.
- Accumulated scroll thresholds prevent the header from flickering during small finger movements.
- Scrolling upward restores it.
- Sticky subnavigation, such as training shortcuts or media controls, moves into the vacated top position when the main header hides.
- `body.site-header-is-hidden` is the shared state hook used by dependent sticky elements.

### Mobile navigation drawer

- Three-bar trigger with `aria-expanded` and `aria-controls`
- fixed viewport overlay and backdrop
- slide-in drawer with full site navigation
- inert/`aria-hidden` state when closed
- focus trap while open
- focus restoration after close
- Escape, backdrop click, close button, navigation click, and swipe dismissal

### Contextual sticky actions

Key customer journeys include a bottom action bar:

- home
- training and Open Water
- travel and destination detail
- events and event detail
- media

The action uses safe-area insets, stays inside narrow viewports, can be dismissed for the current browser session, and can suppress itself while an equivalent CTA is visible in the page.

### Horizontal discovery rails

Training option cards and other intentionally horizontal tracks receive dynamic scroll dots. `main.js`:

- detects whether a rail actually overflows
- calculates the active page from scroll position
- updates dots during scrolling and resizing
- hides the dots if the viewport can already show everything

### Mobile containment

Responsive rules explicitly constrain canvas, cards, filters, embedded calendars, modal sheets, sticky controls, and destination layouts. The travel and destination pages use `overflow-x: clip` as a final containment boundary while their individual grids use `minmax(0, 1fr)` and viewport-aware widths.

### Management on mobile

At 680 pixels and below:

- desktop sidebar becomes a bottom tab bar
- Dashboard, Agenda, Contacts, Classes, Calendar, and More stay reachable
- secondary tools open in a More sheet
- the More sheet includes Media, Travel, the public DMZ Scuba home page, and Log Out
- editor becomes a near-full-height bottom sheet
- the floating Save button is fixed above the bottom navigation
- Save appears only after an actual form change
- bulk actions, media upload, search, and content editing reflow for touch

---

## Training and conversion system

The training section is a connected funnel, not a collection of isolated pages.

| Route | Purpose |
|---|---|
| `/pages/training/` | Training catalog and pathway hub |
| `/pages/training/discover-scuba/` | First-experience / try-scuba program |
| `/pages/training/open-water/` | Full Open Water certification presentation |
| `/pages/training/open-water-referral/` | Referral training path |
| `/pages/training/advanced-specialty/` | Advanced training overview |
| `/pages/training/specialty/` | Specialty catalog |
| `/pages/training/specialty/drysuit/` | Dry Suit course |
| `/pages/training/specialty/full-face-mask/` | Full Face Mask course |
| `/pages/training/specialty/nitrox/` | Computer Nitrox course |
| `/pages/training/specialty/wreck/` | Wreck Diver course |
| `/pages/training/skill-refresh/` | Returning-diver skill refresh |
| `/pages/training/course-builder/` | Class-date and course request builder |
| `/pages/training/interactive-tools/` | Dive-physics learning tools |

### Course-page pattern

Commercial training pages generally include:

- one search-oriented H1 and canonical URL
- course outcome and fit
- prerequisite and logistics information
- included items and training format
- expandable mobile sections
- context-specific sticky action
- course-prefilled route into the Course Builder

Automated tests enforce title, description, canonical link, one H1, image alt text, valid internal links, known course-prefill IDs, and the presence of course-specific mobile actions.

### Open Water choice architecture

The Open Water page presents group and private formats as a horizontally explorable choice surface. The behavior remains touch-native while visual dots clarify that additional options exist offscreen. Supporting content prioritizes outcome, included items, schedule expectations, and next action before lower-priority detail on mobile.

### Course Builder

The builder collects:

- requested course
- current certification
- group size
- group/private Open Water preference when applicable
- timing and scheduling needs
- payment preference
- contact details and goals

`course-builder.js` manages custom dropdown behavior and conditional Open Water format controls. Submission uses the shared form pipeline, so the request becomes both an email and a management inquiry.

### Interactive learning tools

Two self-contained browser simulations support dive education:

- **Boyle's Law Balloon Demo:** illustrates pressure/volume changes with depth.
- **Underwater Color Loss Demo:** visualizes wavelength/color loss underwater with interactive camera and scene behavior.

These pages contain their own rendering and animation code so they can run independently or inside the training-tools experience.

---

## Travel and destination system

### Travel landing page

The page guides visitors in this order:

1. understand the travel offering
2. explore the globe
3. browse destination cards
4. inspect a selected destination preview
5. review upcoming travel events
6. open the full destination detail or make contact

Search and tag pills filter both the globe pins and the destination list. Selecting either representation updates the other.

### Canvas globe

`js/globe.js` implements:

- device-pixel-ratio-aware canvas sizing
- offscreen texture buffers
- sphere texture projection
- continuous rotation and inertial drag
- mouse, touch, pinch, wheel, and button zoom
- pin projection and hit testing
- destination search and tag filtering
- home/reset behavior
- selected-pin highlighting
- synchronized card/list selection
- deferred animation until the globe is near the viewport

It uses Canvas 2D only. No mapping SDK, WebGL runtime, or 3D dependency is required.

### Live trip-status pins

The globe reads events and relates them to destination aliases. Pin state communicates whether a destination has no trip, a planned trip, an approaching departure, or an active trip. Because the status is derived from live event dates, staff can change travel momentum through the event system without editing globe code.

### Destination detail model

The detail page is data-driven through `?id=<destination-id>`. A destination can provide:

- name, subtitle, latitude, longitude, and tags
- hero and isometric/resort images
- short summary and long narrative
- experience and seasonality guidance
- logistics and day-to-day expectations
- resort/dive-operator notes
- dive sites, conditions, and non-diving options
- structured highlights and trip bullets
- budget, packing, skill, and safety context
- related media filtered from the public media library
- interest-list form with destination context

### Travel editing

There are two editing surfaces:

- authenticated controls embedded on public travel/destination pages
- the native Travel Studio inside `/management/`

The public editor supports structured basics, coordinates, images, summary copy, detailed copy, bullets, JSON inspection, add/delete, and publishing. Images upload directly to Cloudflare Images. The management studio provides list/search/edit/preview behavior inside the console.

---

## Events and registration system

### Public discovery

The events page combines:

- mobile jump navigation
- native calendar
- agenda/list presentation
- event-type filters
- selected-date event summaries
- registration availability checks
- shareable event details
- booking-oriented calls to action

Supported public type filters include training, travel, local dives, workshops, and community events.

### Event definitions and expansion

The Worker stores a calendar payload in D1. `events.js` expands compact definitions into dated instances. Definitions can include:

- start and end date/time
- location and tag/type
- recurring interval and unit
- multi-day duration
- skipped dates and occurrence overrides
- registration enable/close state
- capacity
- registration email configuration
- event-page content

The client expands recurrence only through the required coverage window, then sorts and renders instances.

### Event details and sharing

Visitors can open a modal from the calendar or list. A share action uses the native Web Share API where supported and clipboard fallback elsewhere. Shareable URLs can target the standalone `event.html` surface and optionally auto-open registration.

### Registration flow

```text
Visitor opens event
  -> client requests registration snapshot for source ID + date
  -> Worker returns capacity, close state, and approved/pending names
  -> visitor submits contact, certification, and party information
  -> Worker validates capacity and date
  -> registration row inserted with approval_status = pending
  -> attendee confirmation and internal notification sent through Resend
  -> registration appears in management Signups and class escrow views
```

Registration is keyed by event source ID plus occurrence date, allowing recurring occurrences to have independent rosters.

### Staff registration operations

Authenticated staff can:

- inspect occurrence-specific rosters
- approve or update a registration status
- resend an attendee email
- remove a registration
- close event registration
- send an event alert email to opted-in contacts
- edit registration email copy/template settings

### Calendar administration

`events-admin.js` is the large calendar-authoring surface. It supports definition editing, date selection, recurrence, multi-day behavior, preview synchronization, occurrence-level adjustments, event detail content, and publish/delete operations. Its UI state is stored in session storage so a refresh can restore the current editing context.

---

## Media platform

### Public library

The media page supports:

- text search
- media-type and tag filtering
- destination/location filtering
- sort modes including manual, date, views, and shuffle where available
- adjustable card size persisted in local storage
- responsive masonry presentation
- modal playback
- full-screen Reel Mode
- YouTube and Instagram outbound discovery

The library fetches media and destinations in parallel. Destination data turns raw location identifiers into recognizable filter labels.

### Supported media sources

| Source | Detection and playback |
|---|---|
| Cloudflare Stream | Stream ID or delivery URL; iframe/HLS/native fallback |
| YouTube | ID parsed from watch, short, or embed URL; YouTube embed API |
| Local/remote video | Native `<video>` playback |
| Photo | Standard responsive `<img>` |

Video thumbnails are taken from provider URLs where possible. For a directly playable video without a poster, the client can seek to an early frame, draw it to canvas, and create a JPEG data URL.

### Masonry and lazy behavior

The grid uses CSS columns plus JavaScript updates for dynamic media. Layout recalculation is batched with `requestAnimationFrame`. Intersection observers delay expensive video work and pause media that leaves the relevant viewport.

### Reel Mode

Reel Mode creates a full-screen vertical feed:

- locks and later restores the underlying page scroll position
- renders touch-friendly, vertically scrolling media cards
- activates media based on viewport intersection
- pauses inactive video
- coordinates sound state
- supports Stream, HLS, YouTube, local video, and photos
- loads HLS.js only when the browser cannot play HLS natively
- cleans up provider/controller state when closed

### Public-page media editor

Authenticated staff can open an editor on the media page. It provides manual item editing, search, ordering, bulk publish, Stream date synchronization, and a file-first upload path. It remains useful as a public-page preview/editor surface.

### Management Media Studio

The management console provides the primary simplified workflow:

1. press **Upload Media** or **Add Item**
2. select a photo or video from the device
3. enter title, description, location, and tags
4. press **Publish Media**
5. the dialog closes and upload progress remains visible
6. the item metadata is written to the live media library

File names do not prefill the public title. Provider URLs and thumbnail fields live under advanced hosting options rather than being presented as normal editorial inputs.

The studio also loads current live items for search and editing, tracks item edits/deletions, publishes a complete media diff, supports preview toggling, and can synchronize Stream creation dates.

---

## Quiz, forms, and CRM handoff

### Adaptive Dive Path Quiz

`quiz.js` provides two modes:

- **Quick Recommendation:** a short branching route for immediate direction.
- **Dive Path Builder:** a deeper assessment of experience, confidence, goals, travel, timeline, team, and skill focus.

Questions can include a `when(answers)` predicate. The active question set is recalculated as answers change, and answers to branches that become inactive are removed. This prevents stale hidden answers from influencing recommendations.

Results score routes such as certification, refresh, travel preparation, or direct consultation. The result builds a context-rich CTA and optional email capture.

### Quiz-to-CRM automation

On a quiz capture submission, the Worker can:

1. send the internal lead notification
2. send a quiz-results acknowledgment/template
3. find or create a Contact by email
4. store quiz route, mode, path, recommendation, and answer summary in contact extras
5. create a linked Inquiry with the quiz recommendation in its title and notes

### Public inquiry automation

Successful non-spam contact submissions generate a management Inquiry containing:

- source form and page
- submission timestamp
- contact information
- related course, event, destination, location, or interest
- serialized submitted fields
- message/notes
- default `new` status and normal priority

### Event-alert contacts

Event-alert signup finds a contact by lowercase email or creates a new Contact. It sets opt-in metadata in `extras`, including source and timestamps. The management console can list eligible subscribers and send an event alert through the protected API.

### Spam handling

The Worker recognizes honeypot values such as `honey` or `website` for contact submissions, and `honey`, `website`, or `company` for event-alert subscriptions. A populated trap returns a harmless success response without email delivery or normal processing.

---

## Management console

`/management/` is a single authenticated application shell implemented in one HTML document and 12 loaded JavaScript files. It is marked `noindex`, `nofollow`, and `noarchive`.

### Navigation model

Primary areas:

- Dashboard
- Agenda
- Contacts
- Inquiries
- Classes
- Calendar
- Tasks
- Signups
- Media Library
- Travel

Desktop uses a fixed sidebar and primary content pane. Mobile uses a bottom tab bar and a More sheet. The public home link and Log Out remain available from the mobile More sheet.

### Dashboard

The dashboard summarizes:

- open workload
- overdue and due-today items
- open inquiries
- open tasks
- classes and trips approaching within seven days
- direct links into matching records
- shortcuts to the homepage ticker and upcoming events

It reloads from the same management API and reacts when records are replaced or updated.

### Record model

All operational records use one `management_records` table. Common columns hold identity, status, priority, contact, dates, and notes. Type-specific information is stored in `data_json.extras`.

Supported editorial types:

| Type | Purpose |
|---|---|
| Contact | Long-term customer/diver profile and enrollment relationship |
| Inquiry | Lead, planning conversation, financial/follow-up pipeline |
| Class | Class schedule, capacity, roster, public calendar sync |
| Trip / Calendar | Public event and registration configuration |
| Task | Internal operational work item |
| Registration | Online signup representation/escrow where applicable |

### Contact records

Contacts support name, email, phone, certification level, source, profile notes, email-alert state, quiz metadata, and linked class enrollments. A contact can be added to a class from either side of the relationship.

### Inquiry pipeline

Inquiry status options represent the customer-development lifecycle:

```text
New / To Contact
  -> Reached Out
  -> Gathering Details
  -> Planning / Timing
  -> Payment
  -> Complete, Dead End, Not Fit, or Archived
```

Inquiries support incoming/outgoing direction, category, linked contacts, related activity, next step, follow-up date, owner, priority, and financial values. Outstanding balance is computed from amount owed minus amount paid and rendered as a badge.

### Classes

Classes can contain separate Classroom, Pool, and Open Water sessions. Each session has date, start/end time, and location. Class records include capacity, registration-close state, description, roster, and registration email configuration.

The rendered class status is date-aware: future sessions imply scheduled, current sessions can imply active, and all-past sessions can render complete. Explicit closed states remain authoritative.

Saving a class can synchronize its sessions into the public event calendar. The first session anchors registration while the full schedule remains available for customer communication.

### Registrations and escrow

The console combines class roster information with online event registrations. Staff can see pending registration data before it becomes a finalized class relationship, approve or convert records, resend email, or delete a registration.

### Agenda behavior

Agenda is the default operational queue. Closed states include:

- `complete`
- `completed`
- `closed`
- `archived`
- `cancelled`
- `dead_end`
- `not_fit`

These are hidden by default. **Show Done** reveals them and persists the preference in local storage. Search, type filters, priority, pinning, due dates, and sorting determine the visible work queue.

### Bulk operations

Select mode adds accessible check controls to visible record cards. Staff can:

- select/deselect all visible items
- mark complete
- set a shared follow-up date
- change status
- change priority
- archive
- permanently delete after confirmation

The module fetches full records before PUT operations so unchanged fields are preserved.

### Record editor and contextual save

The editor changes visible fields and labels by record type. On mobile it becomes a bottom sheet. A viewport-fixed Save button:

- begins hidden
- appears after an input, change, session add, or session removal makes the form dirty
- stays above page/editor content and mobile navigation
- submits the existing record form
- disappears after save/close/reset

### Productivity features

- quick status-advance buttons on record cards
- timestamped note appender for running activity logs
- pin/favorite behavior
- balance badges
- keyboard navigation and shortcut help
- CSV import with automatic/manual column mapping and preview
- CSV export
- homepage ticker editor
- media and travel content studios
- refresh and logout actions

### Keyboard shortcuts

| Key | Action |
|---|---|
| `1` | Dashboard |
| `2` | Agenda |
| `3` | Contacts |
| `4` | Inquiries |
| `5` | Classes |
| `6` | Calendar |
| `N` | New record |
| `/` | Search |
| `Ctrl/Cmd + F` | Focus record search in Operations |
| `?` | Shortcut help |
| `Escape` | Close active overlay/editor |

Keyboard handlers avoid firing shortcuts while the user is typing in an input, textarea, select, or editable element.

---

## Frontend module reference

| File | Responsibility |
|---|---|
| `js/main.js` | Shared navigation, mobile drawer, forms, telemetry, copy/toast, prefill, sticky CTAs, scroll dots, sticky header |
| `js/home.js` | Homepage ticker loading and fallback |
| `js/quiz.js` | Adaptive quiz, scoring, result routing, capture form |
| `js/course-builder.js` | Course Builder dropdowns and conditional Open Water format |
| `js/events.js` | Public event expansion, calendar/list rendering, filters, modals, sharing, registration |
| `js/events-admin.js` | Authenticated event/calendar authoring and preview synchronization |
| `js/event-detail.js` | Standalone shareable event detail page |
| `js/globe.js` | Canvas globe, destination list, filters, status pins, gestures |
| `js/destination.js` | Destination detail rendering, related media, interest form, page-level editing |
| `js/destinations-edit.js` | Legacy/embedded destination editing workflow and draft behavior |
| `js/travel-admin.js` | Travel-page editor authentication and CRUD |
| `js/media.js` | Public media filtering, masonry, providers, modal playback, Reel Mode |
| `js/media-logic.js` | Testable pure media search/filter/sort helpers |
| `js/media-edit.js` | Public media editor, direct uploads, drafts, bulk publish, Stream sync |
| `js/nfc.js` | NFC contact/opportunity page behavior |
| `js/management.js` | Core console state, records, events, forms, CRUD, class/contact/registration workflows |
| `js/management-dashboard.js` | Dashboard metrics and operational summaries |
| `js/management-media-studio.js` | Simplified live media search/edit/upload/publish workflow |
| `js/management-travel-studio.js` | Destination list/edit/preview studio |
| `js/management-bulk.js` | Multi-select and blanket record actions |
| `js/management-balance.js` | Outstanding-balance badge synchronization |
| `js/management-hide-complete.js` | Persistent Show Done behavior |
| `js/management-import-export.js` | CSV parsing, mapping, preview, import, and export |
| `js/management-keyboard.js` | Console keyboard shortcuts and help overlay |
| `js/management-more.js` | Mobile More sheet, badges, and active state |
| `js/management-note-logger.js` | Timestamped activity-note appender |
| `js/management-quick-advance.js` | Record-specific next-status action |
| `js/management-cal-toggle.js` | Retained compatibility module; no longer loaded by the console |

---

## Cloudflare Worker

`workers/dmz-media-api/src/index.js` is the system's server-side application. It is approximately 3,300 lines and owns:

- CORS policy and preflight responses
- admin authentication and session validation
- public contact and event-alert ingestion
- automatic CRM record creation
- media CRUD and bulk persistence
- direct Stream and Images upload authorization
- Stream creation-date synchronization
- destination CRUD
- event definition CRUD and expansion data
- occurrence-level registration CRUD and approval
- homepage ticker reads/writes
- event-alert subscriber selection and sending
- Resend templates and inline branded email fallbacks
- client telemetry logging

### Request dispatch

The Worker uses an explicit method/path chain in `fetch()`. Every response is wrapped with computed CORS headers. Unknown routes return JSON 404 rather than falling through to HTML.

### Normalization strategy

Inputs are normalized before persistence:

- text is trimmed and capped to field-specific lengths
- choices are constrained to allowed values where applicable
- IDs are normalized for event/destination keys
- nested feature data is stored under a controlled `extras` or `data_json` shape
- user-provided values inserted into email HTML pass through HTML escaping or rich-text conversion

### Schema bootstrapping

The Worker includes defensive `CREATE TABLE IF NOT EXISTS` and selected `ALTER TABLE` helpers for newer tables/columns. `schema.sql` remains the declarative reference for a new database, while runtime guards help an existing D1 database tolerate incremental deployment.

---

## API reference

All browser code should call same-origin `/api/*`. Production routing forwards requests to the Worker.

### Public endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/media` | Return public media grouped as video/media and photo items |
| `POST` | `/api/contact` | Send inquiry/quiz/interest email and create CRM records |
| `POST` | `/api/event-alert-subscribe` | Create/update an opted-in contact |
| `POST` | `/api/client-telemetry` | Log sanitized operational events and persist allowlisted training-funnel events in D1 |
| `GET` | `/api/v2/events` | Return event calendar payload |
| `GET` | `/api/v2/events/:sourceId/registrations?date=YYYY-MM-DD` | Return occurrence capacity and public roster snapshot |
| `POST` | `/api/v2/events/:sourceId/registrations` | Create occurrence registration |
| `GET` | `/api/v2/home-ticker` | Return homepage ticker setting |
| `GET` | `/api/v2/destinations` | Return all destination objects |
| `GET` | `/api/v2/destinations/:id` | Return one destination |

### Authentication

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/admin/login` | Validate admin credentials and create 24-hour D1 session |

### Protected management endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/admin/management` | List records; supports Worker-side filters where supplied |
| `POST` | `/api/admin/management` | Create record |
| `PUT` | `/api/admin/management/:id` | Replace/update normalized record |
| `DELETE` | `/api/admin/management/:id` | Permanently delete record |
| `GET` | `/api/admin/event-alert-subscribers` | List eligible opted-in contacts |

### Protected event endpoints

| Method | Path | Purpose |
|---|---|---|
| `PUT` | `/api/admin/v2/events` | Publish calendar payload |
| `DELETE` | `/api/admin/v2/events` | Remove primary calendar payload |
| `PUT` | `/api/admin/v2/events/:sourceId/registrations/:registrationId/approval` | Update approval state |
| `POST` | `/api/admin/v2/events/:sourceId/registrations/:registrationId/email` | Resend registration email |
| `DELETE` | `/api/admin/v2/events/:sourceId/registrations/:registrationId` | Delete registration |
| `POST` | `/api/admin/v2/events/:sourceId/alerts` | Send event-alert email to eligible contacts |

### Protected content endpoints

| Method | Path | Purpose |
|---|---|---|
| `PUT` | `/api/admin/v2/home-ticker` | Publish ticker setting |
| `PUT` | `/api/admin/v2/destinations/:id` | Create/update destination |
| `DELETE` | `/api/admin/v2/destinations/:id` | Delete destination |
| `POST` | `/api/admin/media` | Create media item |
| `PUT` | `/api/admin/media/:id` | Update media item |
| `DELETE` | `/api/admin/media/:id` | Delete media item |
| `PUT` | `/api/admin/media-bulk` | Publish ordered media set and deletions |
| `POST` | `/api/admin/stream-date-sync` | Sync D1 dates from Stream metadata |

### Protected upload endpoints

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/admin/stream-direct-upload` | Return one-time Stream direct-upload URL |
| `POST` | `/api/admin/stream-tus-upload` | Initialize resumable TUS Stream upload |
| `POST` | `/api/admin/images-direct-upload` | Return Images upload URL, ID, and delivery URL |
| `POST` | `/api/admin/images-delete` | Delete hosted image by ID or supported delivery URL |

### Response and error conventions

- JSON uses `{ ok: true, ... }` for many mutation responses.
- Errors use `{ ok: false, error: "..." }` and an appropriate 4xx/5xx status.
- Protected endpoints return `401 Unauthorized` when the token is missing or expired.
- CORS headers are attached to success and error responses.
- public data endpoints use normalized wrapper objects rather than returning raw D1 rows.

---

## D1 data model

Database binding: `DB`<br>
Database name: `dmz_media`

### `media_items`

Stores public media metadata.

| Column | Meaning |
|---|---|
| `id` | Stable media ID |
| `type` | Video/photo classification |
| `title`, `description` | Editorial copy |
| `tags`, `badge`, `thumb_text`, `meta` | Display/filter metadata |
| `url`, `thumb_url`, `stream_id` | Provider and thumbnail references |
| `location` | Destination/location association |
| `sort_order` | Manual ordering |
| `created_at` | Editorial/provider creation date |

Index: `idx_media_type`.

### `admin_sessions`

| Column | Meaning |
|---|---|
| `token` | Random UUID bearer token |
| `created_at` | Session issue time |
| `expires_at` | 24-hour expiration timestamp |

Expired rows are ignored during authentication. The current code does not use cookies or refresh tokens.

### `funnel_events`

Stores the approved training journey from a course-page view through a completed inquiry. Each row uses a random event ID for idempotency and a random per-tab session ID for funnel grouping. The table stores only normalized page paths and an event-specific allowlist of course, device, campaign label, CTA, destination, source-page, experience, and group fields.

The Worker does not persist full URL query strings, IP addresses, user-agent strings, names, email addresses, phone numbers, messages, form-field contents, `gclid`, or arbitrary event properties in this table. A daily scheduled handler deletes rows older than 400 days by default.

Indexes support retention cleanup, event/time reporting, and session/time funnel analysis.

### Destination tables

The schema contains historical destination tables (`destinations_base`, `destinations_expanded`, and `destinations`) plus the current `destinations_v2` table.

`destinations_v2` stores:

- `id`
- full destination object in `data_json`
- `created_at`
- `updated_at`

The Worker can seed/migrate v2 data from earlier destination representations when needed.

### `events_v2`

Stores a keyed calendar payload:

- `calendar_key` (normally `primary`)
- `data_json` containing definitions/templates/events
- creation/update timestamps

### `event_registrations_v2`

Stores occurrence-level registrations:

- event `source_id`
- occurrence `event_date`
- registrant identity/contact/certification
- additional guest count and computed party size
- `approval_status`, defaulting to `pending`
- creation timestamp

Index: `idx_event_regs_source_date` on source ID and date.

### `site_settings`

Generic JSON settings keyed by `setting_key`. The homepage ticker uses this table.

### `management_records`

Common operational columns:

- `id`, `record_type`, `title`
- `status`, `priority`, `owner`
- contact name/email/phone
- due date and related event
- notes
- `data_json` for type-specific `extras`
- creation/update timestamps

Indexes:

- `idx_management_type_status`
- `idx_management_due_date`

This flexible model makes new CRM fields possible without a D1 migration for every type-specific property.

---

## Static data and fallback behavior

| File | Role |
|---|---|
| `assets/data/destinations.json` | Base destination fallback and seed content |
| `assets/data/destinations-expanded.json` | Detailed legacy destination fallback |
| `assets/data/events.json` | Minimal event fallback |
| `assets/data/events-expanded.json` | Expanded event seed/reference data |
| `assets/data/media.json` | Media fallback/seed structure |
| `assets/data/home-ticker.json` | Ticker fallback |

Dynamic production content is expected to come from the Worker. Static files provide local-preview resilience, initial/legacy data, and controlled failover for selected public reads. They should not be assumed to mirror the latest D1 content automatically.

When changing seeded destination content, keep base and expanded IDs aligned. Production editorial changes should be published through the authenticated API/editor so D1 remains current.

---

## Authentication and security model

### Implemented controls

- Admin credentials live in Worker secrets, not repository source.
- Successful login creates a random UUID session in D1.
- Sessions expire after 24 hours.
- Protected routes call `requireAuth()` before reading or mutating admin data.
- Browser tokens are sent as bearer tokens.
- CORS allows DMZ domains, local development, Cloudflare Pages previews, and configured origins.
- Preflight requests advertise only the headers/methods required by the application.
- Public form honeypots reduce low-effort bot submissions.
- Client telemetry enforces approved origins, bounded payloads, event and field allowlists, normalized paths, and idempotent D1 inserts.
- User-controlled text is normalized, length-bounded, and HTML-escaped for generated email/UI contexts.
- Management is excluded from search indexing through robots metadata.
- Destructive UI operations require explicit selection and, where appropriate, confirmation.
- Direct-upload credentials remain in Worker secrets; clients receive only one-time upload URLs.

### Trust boundaries

- The Pages Function is a transport proxy, not an authorization layer.
- The Worker is the enforcement boundary for protected operations.
- The management token is stored in `localStorage` under `dmzMediaToken`; any script executing in the same origin could access it.
- Public event registration exposes only the roster information intentionally returned by the snapshot handler.
- `/api/client-telemetry` logs bounded, sanitized diagnostics and persists only allowlisted training-funnel events to D1; it is not a general event-ingestion database.

### Current limitations

The repository does not currently implement a global Content Security Policy, cookie-based `HttpOnly` sessions, multifactor authentication, server-side rate limiting, a CAPTCHA, or automatic expired-session cleanup. These are sensible future hardening areas if the threat model or administrative user count grows.

Never commit `.dev.vars`, API tokens, passwords, Wrangler auth material, exported production records, or D1 backups containing personal information.

---

## Email system

All transactional email is sent through Resend by the Worker.

### Email categories

- internal contact-form notification
- general inquiry acknowledgment
- Dive Path Quiz acknowledgment
- destination-interest confirmation
- event registration confirmation
- internal event registration notification
- registration resend
- event alert broadcast

### Template selection

Optional Resend template IDs are read from environment variables. When a configured template is unavailable for a supported flow, the Worker can use branded inline HTML/text builders. Destination-specific selection recognizes Catalina/Southern California, Roatan, Playa del Carmen, Mermet Springs, Haigh Quarry, Key Largo, and Cozumel.

### Event email modes

An event can use:

1. a Resend template ID with variables
2. custom body HTML inside the DMZ email shell
3. complete custom HTML
4. generated default confirmation content

Event merge data includes attendee and event context such as name, date, time, location, party size, and registration link where configured.

### Delivery sequencing

CRM records are generally saved after successful email delivery for the corresponding public inquiry path. A Resend configuration/delivery failure returns an error rather than claiming the customer request completed normally.

---

## Design system and CSS architecture

### File responsibilities

| File | Scope |
|---|---|
| `css/base.css` | Color tokens, typography, reset, body foundation |
| `css/components.css` | Buttons, cards, header, navigation drawer, shared controls |
| `css/main.css` | General site sections and desktop/global layout |
| `css/responsive.css` | Shared breakpoint behavior and mobile contracts |
| `css/pages/*.css` | Home, events, media, travel, destination, management, and other page-specific systems |

### Visual language

The design uses a cinematic deep-ocean palette:

```css
--bg: #050b14;
--bg-2: #071325;
--text: #eaf2ff;
--muted: rgba(234, 242, 255, 0.72);
--accent: #e21b23;
--glow: rgba(85, 185, 255, 0.18);
--radius: 18px;
```

Common motifs include translucent deep-blue panels, fine blue borders, soft glow, rounded cards, high-contrast white text, restrained red conversion actions, and large underwater imagery.

### Breakpoints

The project uses page-appropriate breakpoints, with major mobile contracts around 780 and 680 pixels. The management console's application-shell conversion is centered at 680 pixels. Form controls use a 16-pixel mobile font size to avoid unwanted browser zoom.

### Motion

Motion is used for drawers, sticky controls, card feedback, canvas rotation, and media. `prefers-reduced-motion: reduce` rules remove or reduce nonessential transitions/animation where the shared responsive layer controls them.

---

## Accessibility

Accessibility is implemented as an application behavior, not only static markup.

Current patterns include:

- semantic landmarks and headings
- one H1 enforced on training pages
- descriptive image `alt` attributes enforced in tests
- `aria-current` for active navigation
- `aria-expanded`, `aria-controls`, `aria-hidden`, and `aria-pressed` state synchronization
- modal/dialog roles
- focus trapping in the mobile drawer and quiz
- focus restoration after overlays close
- Escape-key dismissal
- inert mobile navigation while closed
- live regions for form, upload, filter, calendar, and publishing feedback
- native buttons for interactive controls
- keyboard shortcuts that defer while the user is typing
- minimum touch-friendly mobile controls
- reduced-motion support
- visible scroll dots for otherwise ambiguous horizontal content

Accessibility is not represented as a formal WCAG certification; manual assistive-technology and color-contrast audits remain valuable release practices.

---

## SEO and discoverability

The site includes:

- route-specific titles and meta descriptions
- canonical URLs under `https://www.dmzscuba.com/`
- `robots.txt` with sitemap declaration
- XML sitemap for public discovery routes
- one-H1 validation across training pages
- descriptive, location-aware training titles
- homepage Open Graph and Twitter card metadata
- NFC social metadata
- homepage JSON-LD for business/site context
- crawl exclusion for management
- clean permanent redirect from apex domain to `www`

The training test suite also rejects known encoding artifacts and stale price copy, protecting search snippets and page trust.

---

## Performance and resilience

### Performance choices

- no framework runtime, hydration layer, or bundle bootstrap
- no production compile step
- page-specific scripts/styles rather than one universal application bundle
- responsive mobile-specific hero/logo assets
- native lazy image loading where appropriate
- observer-driven deferred media and globe work
- `requestAnimationFrame` batching for layout/scroll updates
- provider-direct media uploads
- HLS.js loaded only on demand
- compact recurring-event storage with client expansion
- Cloudflare edge delivery for static assets and API

### Resilience choices

- selected API reads have JSON fallback paths
- ticker and optional content fail without breaking the surrounding layout
- media supports multiple providers and playback fallbacks
- clipboard has a legacy textarea fallback
- event sharing falls back from Web Share to clipboard/manual guidance
- image/video upload status remains visible during long mobile uploads
- defensive schema creation/migration helpers protect older D1 deployments
- API errors use predictable JSON shapes so the UI can render feedback

---

## Configuration and secrets

### Versioned Worker configuration

`workers/dmz-media-api/wrangler.toml` defines:

- Worker name: `dmz-media-api`
- entry point: `src/index.js`
- compatibility date: `2026-01-01`
- D1 binding: `DB`
- public nonsecret defaults such as allowed origins and sender labels

### Required/optional runtime values

| Variable or binding | Purpose |
|---|---|
| `DB` | D1 database binding |
| `ADMIN_USER` | Management username secret |
| `ADMIN_PASS` | Management password secret |
| `RESEND_API_KEY` | Resend API secret |
| `RESEND_FROM_EMAIL` | Sender address |
| `RESEND_FROM_NAME` | Sender display name |
| `RESEND_TO` | Internal notification recipient |
| `CF_ACCOUNT_ID` | Cloudflare account for Stream/Images |
| `CF_STREAM_TOKEN` | Stream API token |
| `CF_IMAGES_ACCOUNT_ID` | Optional Images-specific account override |
| `CF_IMAGES_TOKEN` | Images API token |
| `CF_IMAGES_DELIVERY` | Images delivery base/hash |
| `CF_IMAGES_VARIANT` | Default delivery variant |
| `ALLOWED_ORIGINS` | Additional explicit CORS origins |
| `RESEND_TEMPLATE_*` | Optional template IDs for quiz, inquiry, travel, and event alerts |

Use Wrangler secrets or the Cloudflare dashboard for secret values. The placeholders/comments in `wrangler.toml` are documentation, not credentials.

---

## Local development

### Prerequisites

- Git
- a modern browser
- Python 3 for the bundled static server helper
- Node.js for tests and smoke checks
- npm/npx and Cloudflare credentials only when deploying the Worker

### Static preview

On Windows:

```powershell
.\Python Server.bat
```

Then open:

```text
http://localhost:8080
```

Do not rely on opening HTML through `file://` for final testing. Same-origin API routing, module behavior, URL resolution, and browser security features are more representative through HTTP.

### Optional external preview

```powershell
.\Pcloudfare.bat
```

This opens a Cloudflare tunnel to the local server for device testing.

### Useful validation commands

```powershell
node --check js\main.js
node --check js\management.js
node --check workers\dmz-media-api\src\index.js
node --test tests\*.test.cjs
node scripts\smoke-check.mjs --base https://dmzscuba-com.pages.dev
git diff --check
```

### Browser cache busting

HTML references many frequently changed CSS/JS assets with query-string versions. When behavior changes but the file path does not, increment the relevant version so deployed browsers do not reuse a stale asset.

---

## Testing and release validation

### `tests/media-logic.test.cjs`

Tests pure media helpers including:

- search-text aggregation
- key normalization
- item/tag/location filtering
- all-selected-tag semantics
- search term matching
- date/views/shuffle sorting

### `tests/mobile-experience.test.cjs`

Protects structural contracts for:

- accessible mobile drawer
- shared header and brand
- touch-friendly mobile controls
- header/subnav scroll coordination
- events mobile planning flow
- media direct upload and management Media Studio
- agenda bulk operations and closed-record visibility
- contextual floating Save behavior
- mobile management Home/Log Out actions
- sticky-action viewport containment
- public-page unified header adoption
- training rail scroll dots
- travel/globe/destination containment
- presence of optimized mobile assets
- balanced CSS blocks in edited stylesheets

### `tests/training-funnel.test.cjs`

Walks the training subtree and validates:

- title, meta description, canonical, and one H1
- image alt attributes
- local links and fragments
- absence of known encoding/stale-price strings
- course-specific mobile CTA
- Course Builder prefill links
- validity of every referenced course ID

### Deployed smoke checks

`scripts/smoke-check.mjs` checks:

- home, contact, media, and travel HTML markers
- `/api/media` response shape
- `/api/v2/destinations` response shape
- contact honeypot path without delivering a real message

Options:

```text
--base <url>
--timeout-ms <number>
--skip-api
--verbose
```

### Manual release checks

The release checklist adds browser-level verification for:

- public pages and forms
- mobile navigation and sticky actions
- media filters and Reel Mode
- admin login
- publishing and uploads
- affected management operations

---

## Deployment model

DMZ Scuba intentionally uses separate development and production repositories/projects.

| Environment | GitHub repository | Cloudflare Pages project | Primary URL |
|---|---|---|---|
| Development | `dmz34705/DMZScuba.com` | `dmzscuba-com` | `dmzscuba-com.pages.dev` |
| Production | `dmz34705/DMZScuba-live` | `dmzscuba-live` | `www.dmzscuba.com` |

### Development release flow

1. edit only the development repository
2. validate syntax and targeted behavior
3. run automated tests
4. commit and push development `main`
5. verify the development Pages deployment
6. perform mobile/device review

### Production promotion flow

1. obtain explicit approval for the tested commit range
2. confirm both worktrees are clean
3. fetch the development repository into the live clone as a temporary remote
4. cherry-pick only approved commits into live `main`
5. run tests in the live worktree
6. deploy the Worker only if Worker source/configuration changed
7. push live `main` using the repository's intentional one-time protection bypass
8. remove the temporary remote
9. verify `dmzscuba-live.pages.dev` and `www.dmzscuba.com`

The live clone includes a push guard. A deliberate production push uses `ALLOW_MAIN_PUSH=1` for that command only.

### Worker deployment

Only required when `workers/dmz-media-api/**` changes:

```powershell
.\Deploy Worker.bat
```

This runs `npx wrangler deploy` from the Worker directory. A valid Cloudflare login or `CLOUDFLARE_API_TOKEN` must be available.

### Routing

Important `_redirects` rules:

```text
apex dmzscuba.com -> https://www.dmzscuba.com/:splat (301)
/api/*            -> Worker /api/:splat (200 proxy)
/management       -> /management/index.html (200 rewrite)
/training         -> /pages/training/index.html (200 rewrite)
/nfc              -> /pages/nfc/index.html (200 rewrite)
```

---

## Common extension workflows

### Add a public shell page

1. add `pages/<route>/index.html`
2. include canonical, description, one H1, shared CSS, shared header contract, and `main.js`
3. add page-specific CSS only if shared components are insufficient
4. add navigation/footer links if appropriate
5. add the route to `sitemap.xml`
6. add it to unified-shell tests when it uses the full public header
7. verify desktop, 780-pixel, 680-pixel, and narrow-phone layouts

### Add or change a training course

1. update/create the course page
2. add a stable `sdi-*` Course Builder option
3. use that exact value in course-prefilled links
4. include course-specific mobile CTA
5. update catalog navigation and sitemap
6. run `training-funnel.test.cjs`

### Add a destination

1. choose a stable slug ID
2. supply name, coordinates, tags, summaries, images, and detail content
3. publish through the v2 destination API/editor
4. keep fallback base/expanded JSON IDs aligned if fallback content is also updated
5. add event-alias terms if trip titles/locations will not naturally match the destination
6. test globe pin, card, detail page, related media, interest form, and mobile containment

### Add an event feature

1. update event normalization in Worker and client as required
2. preserve definition/instance/source ID/date semantics
3. update admin authoring and public expansion together
4. consider recurrence, multi-day, override, registration, and share-link behavior
5. verify both native public calendar and authenticated admin embed

### Add a management record field

1. add the form control to `management/index.html`
2. add the field to the relevant `typeConfigs` entry in `management.js`
3. read/write it through record `extras` unless it is truly cross-type
4. update CSV mappings/headers if it should import/export
5. update Worker normalization limits/allowed values
6. test create, edit, contextual Save, card rendering, mobile bottom sheet, bulk preservation, and export

### Add a Worker route

1. define and normalize the handler
2. add exact method/path dispatch in `fetch()`
3. call `requireAuth()` for protected behavior
4. return consistent JSON errors
5. ensure all response paths receive CORS
6. update this API table and Worker README
7. run Worker syntax check
8. deploy Worker before frontend code that depends on the route

### Add a media provider

1. implement URL/ID detection
2. implement thumbnail strategy
3. implement normal modal playback
4. implement Reel Mode activation, pause, sound, and cleanup
5. support destination-related media rendering
6. update editor validation and tests

---

## Operational boundaries and technical debt

This section is intentionally candid. It helps a new developer understand where care is required.

- **Large modules:** `management.js`, `events-admin.js`, `media.js`, and several page stylesheets are substantial. Feature modules reduce coupling, but future refactoring could extract more pure logic and shared API utilities.
- **No module loader:** script order and DOM contracts matter. A missing `data-*` hook may silently disable a feature.
- **Some legacy destination code remains:** v2 APIs are current, while earlier tables/files/editor paths remain for compatibility and migration. New work should prefer `/api/v2/destinations` and `/api/admin/v2/destinations/:id`.
- **Shared bearer token:** several editors share `dmzMediaToken`. This is convenient but couples all admin surfaces to one browser-side session mechanism.
- **Client-side recurrence expansion:** public rendering depends on synchronized expansion rules between event surfaces. Changes require cross-surface testing.
- **No build-time lint pipeline:** syntax/tests and disciplined review are the release gates. There is no TypeScript or bundler to catch selector/data-shape mismatch.
- **Static fallback drift:** fallback JSON is not automatically synchronized from D1.
- **Limited security headers:** `_headers` currently defines the security contact content type, not a full CSP/HSTS/permissions policy suite.
- **No formal migration runner:** schema guards and `schema.sql` are used instead of numbered migrations.
- **No formal license file:** repository reuse terms are not declared in this project.
- **Browser/device validation remains important:** Canvas, video autoplay, sticky positioning, safe areas, virtual keyboards, and mobile uploads have platform-specific behavior that automated structural tests cannot fully reproduce.

---

## Project philosophy

DMZScuba.com is designed around three ideas:

1. **The customer journey and the business workflow should share data.** A quiz, registration, class, destination, email, and follow-up should not become disconnected islands.
2. **Mobile usability is a product requirement.** Most visitors arrive on phones, and the management console must also be useful from a phone in the field.
3. **Simple technology can still support a sophisticated product.** The project demonstrates that carefully structured HTML, CSS, vanilla JavaScript, and serverless primitives can deliver rich interaction without a framework or build system.

The result is both a public scuba-business experience and a custom operational platform: a live example of product design, frontend engineering, serverless API design, content modeling, workflow automation, and continual mobile refinement in one production system.
