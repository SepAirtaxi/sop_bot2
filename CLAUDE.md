# CAT Onboard

## What This Is
Internal onboarding assistant for CAT Flyservice (Danish aircraft maintenance company). New hires use it to look up standard operating procedures, contacts, glossary terms, daily tasks, knowledge base articles, pricing info, and warehouse part locations. An AI chat ("Ask CAT") powered by Gemini answers questions based on SOP content.

## Tech Stack
- **Single-file app**: Everything lives in `index.html` (~5065 lines)
- **React 18 + Tailwind CSS** via CDN (production builds) — no build step, no bundler
- **Babel standalone** for JSX transpilation in-browser
- **Firebase Firestore** as sole data store (live config in `CONFIG.firebase`)
- **Firebase Authentication** (email + password) — admin creates accounts in Firebase Console
- **Vercel** for hosting + serverless functions
- **Gemini 2.5 Flash** via `/api/chat.js` serverless proxy
- **marked.js** for Markdown rendering + **DOMPurify** for HTML sanitization

## Project Structure
```
cat-onboard/
├── index.html          # Entire frontend app (React components, data, styles)
├── api/
│   └── chat.js         # Vercel serverless function — proxies to Gemini API
├── vercel.json         # SPA rewrite rules
├── package.json        # Metadata only (no dependencies)
├── .env.example        # GEMINI_API_KEY placeholder
├── temp/
│   └── daily_tasks.md  # Reference data for daily tasks
└── CLAUDE.md           # This file
```

## Architecture

### Single-File Structure (`index.html`)
The entire app is one HTML file with embedded `<script type="text/babel">`. Sections are separated by comment banners (`// === SECTION NAME ===`). The major sections in order:

| Lines (approx) | Section |
|-----------------|---------|
| 1–89 | HTML head, CSS styles, animations, glossary tooltip styles |
| 100–135 | CONFIG + Firebase initialization (Firestore + Auth) |
| 138–330 | `FirestoreService` — generic CRUD + collection-specific helpers (incl. partLocations, users) |
| 257–360 | Helpers: `timeAgo()`, formatters, `processContentLinks()`, `processGlossaryTerms()` |
| 362–447 | Shared UI helpers: `renderMarkdown()`, `renderRichContent()`, `createCrossLinkHandler()`, `ExpandToggleButton`, icon components, `useEscapeKey` hook |
| 448–868 | `DEFAULT_SOPS` — hardcoded SOP content (Markdown strings, with `sopNumber`) |
| 870–1090 | `DEFAULT_DAILY_TASKS` (with `dtNumber`), `DEFAULT_CATEGORIES` data |
| 1092–1150 | `INITIAL_GLOSSARY` data |
| 1152–1160 | `INITIAL_CONTACTS` data |
| 1162–1435 | `INITIAL_PART_LOCATIONS` (271 entries with category, subcategory, locations array) |
| 1440–1592 | `GeminiService` — builds system prompt, calls `/api/chat` |
| 1594–1701 | `NAV_ITEMS` config + `NavIcon` SVG component |
| 1703–1779 | `AiAvatar` — bot chat avatar with sparkle icon + cat easter egg (`AiSparkleIcon`, `CatFaceIcon`, `aiEasterEggScheduler`) |
| 1781–2030 | `App` — root component (auth state + role loading, data loading, migration, CRUD handlers) |
| 1959–2022 | `LoginScreen` |
| 2024–2195 | `AppShell` — sidebar + content area, page routing, deep-link navigation |
| 2197–2440 | `AskCATPage` — chat interface + inline wizard mode |
| 2442–2567 | `GuidedProcessWizard` — step-by-step card UI |
| 2569–2814 | `SOPArchivePage` — searchable SOP list with full viewer + cross-links |
| 2816–2954 | `DailyTasksPage` — categorized task checklist + cross-links |
| 2956–3122 | `KnowledgeBasePage` — article list + viewer + cross-links |
| 3124–3226 | `GlossaryPage` — abbreviations + terms display |
| 3228–3272 | `ContactsPage` — contact cards |
| 3274–3409 | `PricingPage` — pricing entity display |
| 3411–3557 | `PartLocationsPage` — searchable/sortable table with location badges |
| 3559–3670 | `AdminPage` — tabbed admin panel (8 tabs) |
| 3673–3753 | `AdminSOPsTab` |
| 3755–3840 | `AdminDailyTasksTab` |
| 3842–3927 | `AdminKnowledgeBaseTab` |
| 3929–4072 | `SOPEditor` — reused for SOPs, daily tasks, KB articles (with `[[]]` toolbar) |
| 4074–4165 | `AdminGlossaryTab` |
| 4167–4239 | `GlossaryFormModal` |
| 4241–4311 | `AdminContactsTab` |
| 4313–4383 | `AdminPricingTab` |
| 4385–4489 | `AdminPartLocationsTab` |
| 4490–4603 | `PartLocationFormModal` |
| 4604–4734 | `PricingEntityFormModal` |
| 4736–4793 | `ContactFormModal` |
| 4930–4995 | `AdminUsersTab` — users table (email, role badge, last login, first seen) |
| 4997–5000 | ReactDOM render |

### Routing
State-based via `currentPage` in `AppShell`. No URL router — just a switch on page ID:
- `ask-cat` (default), `sop-archive`, `daily-tasks`, `knowledge-base`, `glossary`, `contacts`, `part-locations`, `pricing`, `admin`
- `handleNavigate(page, target?)` supports deep-linking with optional `{ type, number }` target for cross-link navigation

### Data Flow
1. `App` component loads all data from Firestore on mount (parallel `Promise.all`); no localStorage fallback
2. On first Firestore load with empty collections, it seeds from hardcoded defaults
3. Migration runs on load: any SOPs/DailyTasks/KB articles missing auto-numbers get assigned (max+1, alphabetical) and persisted to Firestore
4. All CRUD operations go through `App`'s handler functions: Firestore write succeeds first, then React state updates; if Firestore fails, state is untouched and error propagates to the admin UI
5. Data is passed down as props to page components

### Authentication & Roles
- Firebase Authentication with email + password (`firebase.auth().signInWithEmailAndPassword()`)
- No self-registration — admin creates users in Firebase Console (Authentication > Users)
- `onAuthStateChanged` listener in `App` is async: calls `FirestoreService.upsertUser()` on login, sets `isAdmin` state from the returned role
- Persistent login across tabs/refreshes (handled by Firebase SDK)
- Login screen shows "Need access? Contact sep@aircat.dk"
- Logged-in user's email displayed in sidebar above logout button

### Role System
- Two roles: `admin` and `user` — stored in `users` Firestore collection (doc ID = Firebase Auth UID)
- `sep@aircat.dk` is hardcoded as admin on first login; all other users default to `user`
- On every login, `upsertUser(uid, email)` creates the doc (if new) or updates `lastLogin` (if existing) — role is never overwritten after creation
- `isAdmin` boolean prop flows from `App` → `AppShell` → page routing
- Only difference: admins see the Admin Panel button in the sidebar and can access the admin page; non-admins are redirected to Ask CAT if they navigate to `/admin`
- **Admin panel → Users tab**: lists all users who have logged in at least once (email, role badge, last login, first seen)

### AI Chat (`AskCATPage`)
- Two modes: **Chat** (free-form Q&A) and **Wizard** (step-by-step guided process)
- `GeminiService.buildSystemPrompt()` injects all SOP content into the system prompt
- Chat calls `/api/chat.js` which proxies to Gemini 2.5 Flash
- Responses rendered as Markdown via `marked.js`

## Firebase Collections
| Collection | Key Fields |
|------------|------------|
| `sops` | sopNumber, title, category, content (Markdown), timestamps |
| `glossary` | type (abbreviation/term), name, meaning, context, category, relatedTerms |
| `contacts` | name, role, phone, email |
| `dailyTasks` | dtNumber, title, category, content (Markdown), timestamps |
| `knowledgeBase` | kbNumber, title, category, content (Markdown), timestamps |
| `pricing` | entityName, ranges (array of {from, to, markup}), timestamps |
| `partLocations` | category, subcategory, locations (array of code strings), timestamps |
| `users` | email, role (`admin`/`user`), createdAt, lastLogin (doc ID = Firebase Auth UID) |

## Environment Variables
| Variable | Where | Purpose |
|----------|-------|---------|
| `GEMINI_API_KEY` | Vercel env vars | Google Gemini API key for chat |

## Conventions
- All components are function components using hooks
- Component names are PascalCase, function names camelCase
- Tailwind utility classes for all styling (no separate CSS files beyond the `<style>` block)
- SOP/KB/DailyTask content is stored as Markdown strings
- Admin components follow the pattern: list view + editor/modal for create/edit
- Navigation SVG icons are inline in the `NavIcon` component
- Shared icon components (`EditIcon`, `DeleteIcon`, `CloseIcon`, `EmptyStateIcon`, `SearchIcon`) used across admin tabs
- Shared UI utilities: `renderMarkdown()` (with DOMPurify), `renderRichContent()`, `createCrossLinkHandler()`, `ExpandToggleButton`
- `useEscapeKey(onClose)` custom hook for modal dismiss; modals also close on backdrop click

### Auto-Numbering
- SOPs, Daily Tasks, and KB articles each have a permanent auto-assigned number (`sopNumber`, `dtNumber`, `kbNumber`)
- Numbers are permanent — deleting an item leaves a gap (never reassigned)
- Formatted as `SOP-001`, `DT-001`, `KB-001` via `formatSopNumber()`, `formatDtNumber()`, `formatKbNumber()`
- New items get `max(existing numbers) + 1`
- Migration on load assigns numbers to any items missing them (alphabetical order)

### Cross-Linking
- Markdown content supports `[[SOP-001]]`, `[[DT-001]]`, `[[KB-001]]` syntax
- `processContentLinks(html)` post-processes rendered Markdown to replace these with styled clickable elements
- Same-type links (e.g. `[[SOP-002]]` within an SOP) select in-place; cross-type links navigate to the target page
- Deep-link navigation via `handleNavigate(page, { type, number })` in `AppShell`
- Editor toolbar includes a `[[]]` button for easy cross-link insertion
- Editor preview renders cross-links visually

### Expand View Toggle
- SOP Archive, Daily Tasks, Knowledge Base, and Ask CAT pages each have an `isExpanded` state
- **Animated**: left panel collapses via CSS `width` transition (320px → 0) with simultaneous opacity fade; content area expands via `max-width` transition (`64rem` → `9999px`) — both use `cubic-bezier(0.4,0,0.2,1)` over 350ms
- Left panel always rendered (never conditionally unmounted) so the animation plays; `overflow-hidden` on the panel clips content during collapse
- Ask CAT: message bubbles transition `max-width` between `75%` and `90%` with the same easing
- Shared `ExpandToggleButton` component renders the toggle, styled as a gray pill (`bg-gray-100`) with expand/collapse SVG icons

### Sticky Content Header (SOP Archive, Daily Tasks, Knowledge Base)
- When a document is selected, a compact sticky bar renders **above** the scrollable body — buttons stay visible at all times regardless of scroll position
- Layout: left side = category pill + ID number (one row, small) + title (`text-base font-semibold`) below; right side = action buttons unchanged
- Bar uses `flex-shrink-0` on the outer div and sits inside a `flex flex-col overflow-hidden` right-panel wrapper; scrollable body is a sibling `flex-1 overflow-y-auto` div
- The inner max-width wrapper also carries the expand transition so title and buttons track the content width during animation
- `min-w-0` on the title wrapper prevents long titles from pushing buttons off-screen

### Glossary Auto-Highlight
- `processGlossaryTerms(html, glossaryItems)` scans rendered HTML in SOP/DT/KB viewers for glossary terms
- Matched terms display as blue text with dotted underline; hovering shows a dark tooltip with name, meaning, and context
- Abbreviations matched **case-sensitive**, terms matched **case-insensitive** (whole-word via `\b`)
- Only the **first occurrence** per document is highlighted to avoid visual noise
- Skips content inside `<code>`, `<pre>`, and `<a>` tags (no interference with cross-links or code blocks)
- Longer terms match first (sorted by name length descending) to prevent partial matches
- Runs after `processContentLinks()` in the render pipeline: `processGlossaryTerms(processContentLinks(marked.parse(content)), glossary)`
- Pure CSS tooltips (no JS hover handlers) — `.glossary-term` and `.glossary-tooltip` classes

## Deployment
- Hosted on Vercel at production URL
- `vercel dev` for local development
- `vercel --prod` for production deploy
- SPA rewrites configured in `vercel.json`

## Current Status (updated 2026-02-27)
### Completed
- Role-based access (2026-02-27): `users` Firestore collection, `upsertUser()` on login, `isAdmin` prop flow, Admin Panel gated behind admin role, Users tab in admin panel showing all logged-in accounts with role badge + timestamps; `sep@aircat.dk` hardcoded as sole admin on first login
- Full app shell with sidebar navigation (9 pages + admin)
- Ask CAT chat with Gemini integration + wizard mode
- SOP Archive with search and full viewer
- Daily Tasks page with categories
- Knowledge Base page
- Glossary page (abbreviations + terms)
- Contacts page
- Pricing page
- Admin panel with tabs for all content types (SOPs, Daily Tasks, KB, Glossary, Contacts, Pricing, Part Locations)
- Firebase Firestore as sole data store (localStorage fallback removed 2026-02-26)
- Firebase Authentication (email + password, admin-managed accounts, persistent sessions)
- Auto-numbering for SOPs (SOP-001), Daily Tasks (DT-001), and KB articles (KB-001)
- Cross-linking system: `[[SOP-001]]` / `[[DT-001]]` / `[[KB-001]]` syntax in Markdown with click-to-navigate
- Glossary auto-highlight: glossary terms in SOP/DT/KB content automatically show blue with hover tooltips
- Robust error handling: CRUD failures propagate to admin UI with alert messages
- Document IDs (SOP-001, DT-001, KB-001) styled bold + blue in secondary list menus for visibility
- Code health cleanup (2026-02-26): React production builds, DOMPurify HTML sanitization, shared UI components (icons, ExpandToggleButton, renderRichContent, createCrossLinkHandler), useEscapeKey hook + backdrop close on modals, fixed useEffect dependency arrays, removed dead code (unused handler functions, legacy localStorage cleanup)
- Part Locations module (2026-02-26): searchable/sortable table page with category filter and location badges, admin CRUD tab with form modal (dynamic locations list, category autocomplete), 271 seed entries from Excel, Firestore `partLocations` collection
- Animated expand/collapse (2026-02-27): left panel slides out via CSS width/opacity transition, content area expands via max-width transition — smooth `cubic-bezier` easing on all three pages + Ask CAT bubble widths
- Sticky action bar (2026-02-27): title + buttons extracted into a `flex-shrink-0` header above the scroll container on SOP Archive, Daily Tasks, and Knowledge Base — buttons always visible regardless of scroll position; compact two-line left side (category pill + ID on row 1, title on row 2)
- Ask CAT rich rendering (2026-02-27): chat bot responses now run through the full `processContentLinks` + `processGlossaryTerms` pipeline — `[[SOP-001]]`/`[[DT-001]]`/`[[KB-001]]` cross-links render as clickable chips that navigate to the target document, and glossary terms show blue hover tooltips; `onCrossLink` prop threaded from `AppShell` → `AskCATPage`
- Ask CAT avatar + easter egg (2026-02-27): `NavIcon chat` replaced with AI sparkles icon (two 4-pointed stars); bot chat bubble avatar replaced with `AiAvatar` component showing `AiSparkleIcon`; a singleton `aiEasterEggScheduler` fires a `cat-easter-egg` CustomEvent every 1–3 minutes — all visible avatars simultaneously scaleX-flip to `CatFaceIcon` for ~3.5s then flip back
- Ask CAT cross-link activation (2026-02-27): `GeminiService.buildSystemPrompt` now includes document numbers in headings passed to Gemini (e.g. `## Title [SOP-005] (Category: ...)`) and the system prompt explicitly instructs Gemini to use `[[SOP-001]]`/`[[DT-001]]`/`[[KB-001]]` notation — previously Gemini only wrote plain bold titles which the front-end pipeline ignored

### Planned / Not Yet Started
- Firestore security rules (restrict read/write to authenticated users)
- Analytics dashboard
- PDF export of step-by-step guides
- Multi-language support (Danish/English)

---

## Session Update Protocol
When you (the user) say **"update the roadmap"** at the end of a session, Claude should:
1. Update the "Current Status" section above with any completed work or new plans
2. Update the line count table if `index.html` structure changed significantly
3. Add any new conventions or architectural decisions discovered during the session
4. Keep this file concise — it's read at the start of every session
