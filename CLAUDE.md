# CAT Onboard

## What This Is
Internal onboarding assistant for CAT Flyservice (Danish aircraft maintenance company). New hires use it to look up standard operating procedures, contacts, glossary terms, daily tasks, knowledge base articles, and pricing info. An AI chat ("Ask CAT") powered by Gemini answers questions based on SOP content.

## Tech Stack
- **Single-file app**: Everything lives in `index.html` (~3900 lines)
- **React 18 + Tailwind CSS** via CDN — no build step, no bundler
- **Babel standalone** for JSX transpilation in-browser
- **Firebase Firestore** for persistent data (live config in `CONFIG.firebase`)
- **LocalStorage** as fallback data store
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
| 1–55 | HTML head, CSS styles, animations |
| 66–98 | CONFIG + Firebase initialization |
| 99–207 | `FirestoreService` — generic CRUD + collection-specific helpers |
| 208–263 | `LocalStorage` — fallback store for all collections |
| 264–287 | `timeAgo()` helper |
| 288–704 | `DEFAULT_SOPS` — hardcoded SOP content (Markdown strings) |
| 705–919 | `DEFAULT_DAILY_TASKS`, `DEFAULT_CATEGORIES` data |
| 920–977 | `INITIAL_GLOSSARY` data (versioned with `GLOSSARY_VERSION`) |
| 978–991 | `INITIAL_CONTACTS` data |
| 992–1141 | `GeminiService` — builds system prompt, calls `/api/chat` |
| 1142–1173 | `NAV_ITEMS` config + `NavIcon` SVG component |
| 1175–1580 | `App` — root component (auth state, data loading, CRUD handlers) |
| 1581–1637 | `LoginScreen` |
| 1638–1791 | `AppShell` — sidebar (240px) + content area, page routing |
| 1792–2037 | `AskCATPage` — chat interface + inline wizard mode |
| 2038–2169 | `GuidedProcessWizard` — step-by-step card UI |
| 2170–2401 | `SOPArchivePage` — searchable SOP list with full viewer |
| 2402–2525 | `DailyTasksPage` — categorized task checklist |
| 2526–2676 | `KnowledgeBasePage` — article list + viewer |
| 2677–2780 | `GlossaryPage` — abbreviations + terms display |
| 2781–2826 | `ContactsPage` — contact cards |
| 2827–2963 | `PricingPage` — pricing entity display |
| 2964–3072 | `AdminPage` — tabbed admin panel |
| 3073–3150 | `AdminSOPsTab` |
| 3151–3233 | `AdminDailyTasksTab` |
| 3234–3316 | `AdminKnowledgeBaseTab` |
| 3317–3444 | `SOPEditor` — reused for SOPs, daily tasks, KB articles |
| 3445–3536 | `AdminGlossaryTab` |
| 3537–3605 | `GlossaryFormModal` |
| 3606–3672 | `AdminContactsTab` |
| 3673–3739 | `AdminPricingTab` |
| 3740–3870 | `PricingEntityFormModal` |
| 3871–3928 | `ContactFormModal` |
| 3929–3937 | ReactDOM render |

### Routing
State-based via `currentPage` in `AppShell`. No URL router — just a switch on page ID:
- `ask-cat` (default), `sop-archive`, `daily-tasks`, `knowledge-base`, `glossary`, `contacts`, `pricing`, `admin`

### Data Flow
1. `App` component loads data on mount: tries Firestore first, falls back to LocalStorage, then falls back to hardcoded defaults
2. On first Firestore load with empty collections, it seeds from defaults
3. All CRUD operations go through `App`'s handler functions, which update both Firestore and local state
4. Data is passed down as props to page components

### Authentication
Simple hardcoded login: `cat` / `cat1234` (checked in `LoginScreen`, stored in `sessionStorage`)

### AI Chat (`AskCATPage`)
- Two modes: **Chat** (free-form Q&A) and **Wizard** (step-by-step guided process)
- `GeminiService.buildSystemPrompt()` injects all SOP content into the system prompt
- Chat calls `/api/chat.js` which proxies to Gemini 2.5 Flash
- Responses rendered as Markdown via `marked.js`

## Firebase Collections
| Collection | Key Fields |
|------------|------------|
| `sops` | title, category, content (Markdown), timestamps |
| `glossary` | type (abbreviation/term), name, meaning, context, category, relatedTerms |
| `contacts` | name, role, phone, email |
| `dailyTasks` | title, category, content (Markdown), timestamps |
| `knowledgeBase` | title, category, content (Markdown), timestamps |
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

## Deployment
- Hosted on Vercel at production URL
- `vercel dev` for local development
- `vercel --prod` for production deploy
- SPA rewrites configured in `vercel.json`

## Current Status (updated 2025-02-24)
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
- Firebase Firestore integration with auto-seeding
- LocalStorage fallback
- Login screen

### Planned / Not Yet Started
- User authentication with multiple users/roles
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
