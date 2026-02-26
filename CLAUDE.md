# CAT Onboard

## What This Is
Internal onboarding assistant for CAT Flyservice (Danish aircraft maintenance company). New hires use it to look up standard operating procedures, contacts, glossary terms, daily tasks, knowledge base articles, and pricing info. An AI chat ("Ask CAT") powered by Gemini answers questions based on SOP content.

## Tech Stack
- **Single-file app**: Everything lives in `index.html` (~4200 lines)
- **React 18 + Tailwind CSS** via CDN — no build step, no bundler
- **Babel standalone** for JSX transpilation in-browser
- **Firebase Firestore** as sole data store (live config in `CONFIG.firebase`)
- **Firebase Authentication** (email + password) — admin creates accounts in Firebase Console
- **Vercel** for hosting + serverless functions
- **Gemini 2.5 Flash** via `/api/chat.js` serverless proxy
- **marked.js** for Markdown rendering in chat

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
| 1–88 | HTML head, CSS styles, animations, glossary tooltip styles |
| 99–137 | CONFIG + Firebase initialization (Firestore + Auth) |
| 132–244 | `FirestoreService` — generic CRUD + collection-specific helpers |
| 246–262 | Legacy localStorage cleanup (one-time key removal) |
| 264–395 | Helpers: `timeAgo()`, formatters, `processContentLinks()`, `processGlossaryTerms()` |
| 396–770 | `DEFAULT_SOPS` — hardcoded SOP content (Markdown strings, with `sopNumber`) |
| 723–944 | `DEFAULT_DAILY_TASKS` (with `dtNumber`), `DEFAULT_CATEGORIES` data |
| 945–1006 | `INITIAL_GLOSSARY` data (versioned with `GLOSSARY_VERSION`) |
| 1007–1017 | `INITIAL_CONTACTS` data |
| 1018–1170 | `GeminiService` — builds system prompt, calls `/api/chat` |
| 1171–1203 | `NAV_ITEMS` config + `NavIcon` SVG component |
| 1205–1656 | `App` — root component (auth state, data loading, migration, CRUD handlers) |
| 1657–1714 | `LoginScreen` |
| 1715–1879 | `AppShell` — sidebar + content area, page routing, deep-link navigation |
| 1880–2128 | `AskCATPage` — chat interface + inline wizard mode |
| 2127–2258 | `GuidedProcessWizard` — step-by-step card UI |
| 2259–2520 | `SOPArchivePage` — searchable SOP list with full viewer + cross-links |
| 2521–2673 | `DailyTasksPage` — categorized task checklist + cross-links |
| 2674–2854 | `KnowledgeBasePage` — article list + viewer + cross-links |
| 2855–2958 | `GlossaryPage` — abbreviations + terms display |
| 2959–3004 | `ContactsPage` — contact cards |
| 3005–3141 | `PricingPage` — pricing entity display |
| 3142–3246 | `AdminPage` — tabbed admin panel |
| 3247–3326 | `AdminSOPsTab` |
| 3326–3410 | `AdminDailyTasksTab` |
| 3410–3493 | `AdminKnowledgeBaseTab` |
| 3494–3638 | `SOPEditor` — reused for SOPs, daily tasks, KB articles (with `[[]]` toolbar) |
| 3639–3726 | `AdminGlossaryTab` |
| 3727–3799 | `GlossaryFormModal` |
| 3800–3866 | `AdminContactsTab` |
| 3867–3933 | `AdminPricingTab` |
| 3934–4064 | `PricingEntityFormModal` |
| 4065–4122 | `ContactFormModal` |
| 4123–4131 | ReactDOM render |

### Routing
State-based via `currentPage` in `AppShell`. No URL router — just a switch on page ID:
- `ask-cat` (default), `sop-archive`, `daily-tasks`, `knowledge-base`, `glossary`, `contacts`, `pricing`, `admin`
- `handleNavigate(page, target?)` supports deep-linking with optional `{ type, number }` target for cross-link navigation

### Data Flow
1. `App` component loads all data from Firestore on mount (parallel `Promise.all`); no localStorage fallback
2. On first Firestore load with empty collections, it seeds from hardcoded defaults
3. Migration runs on load: any SOPs/DailyTasks/KB articles missing auto-numbers get assigned (max+1, alphabetical) and persisted to Firestore
4. All CRUD operations go through `App`'s handler functions: Firestore write succeeds first, then React state updates; if Firestore fails, state is untouched and error propagates to the admin UI
5. Data is passed down as props to page components

### Authentication
- Firebase Authentication with email + password (`firebase.auth().signInWithEmailAndPassword()`)
- No self-registration — admin creates users in Firebase Console (Authentication > Users)
- `onAuthStateChanged` listener in `App` manages `currentUser` state + `authLoading`
- Persistent login across tabs/refreshes (handled by Firebase SDK)
- Login screen shows "Need access? Contact sep@aircat.dk"
- Logged-in user's email displayed in sidebar above logout button

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
| `pricing` | name, type, fields (array of {label, value}), notes |

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
- SVG icons are inline in the `NavIcon` component

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

## Current Status (updated 2026-02-26)
### Completed
- Full app shell with sidebar navigation (8 pages + admin)
- Ask CAT chat with Gemini integration + wizard mode
- SOP Archive with search and full viewer
- Daily Tasks page with categories
- Knowledge Base page
- Glossary page (abbreviations + terms)
- Contacts page
- Pricing page
- Admin panel with tabs for all content types (SOPs, Daily Tasks, KB, Glossary, Contacts, Pricing)
- Firebase Firestore as sole data store (localStorage fallback removed 2026-02-26)
- Firebase Authentication (email + password, admin-managed accounts, persistent sessions)
- Auto-numbering for SOPs (SOP-001), Daily Tasks (DT-001), and KB articles (KB-001)
- Cross-linking system: `[[SOP-001]]` / `[[DT-001]]` / `[[KB-001]]` syntax in Markdown with click-to-navigate
- Glossary auto-highlight: glossary terms in SOP/DT/KB content automatically show blue with hover tooltips
- Robust error handling: CRUD failures propagate to admin UI with alert messages

### Planned / Not Yet Started
- Firestore security rules (restrict read/write to authenticated users)
- Role-based access (admin vs regular user)
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
