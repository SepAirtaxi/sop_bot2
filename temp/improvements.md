# CAT Onboard — Code Review Improvements

Identified during deep-dive code review (2026-02-25). Firebase Authentication (item 4 below) is already completed.

---

## 1. Switch to React Production Builds
**Effort:** 2 minutes | **Impact:** Faster app load

Lines 99-100 load `react.development.js` and `react-dom.development.js`. These are slower and include extra warnings meant for developers. Switch to `react.production.min.js` and `react-dom.production.min.js`.

Single biggest quick win.

---

## 2. Extract Duplicated Code (~400-500 lines)
**Effort:** 1-2 hours | **Impact:** ~10-12% smaller file, easier maintenance

| What's duplicated | Where | ~Lines |
|---|---|---|
| `renderMarkdown()` function | AskCATPage + GuidedProcessWizard | 10 |
| `renderContent()` / `renderSopContent()` | SOPArchivePage, DailyTasksPage, KnowledgeBasePage | 30 |
| `handleContentClick()` | Same 3 pages | 45 |
| Deep-link `useEffect` | Same 3 pages | 30 |
| `nextStep()` wizard logic | AskCATPage + SOPArchivePage | 15 |
| CRUD handlers (create/update/delete) | App component — same pattern x6 entity types | 150 |
| FirestoreService collection wrappers | e.g. `getSOPs()` just calls `getAll('sops')` | 35 |
| LocalStorage get/save pairs | Identical pattern x7 | 40 |
| Admin tabs (SOPs, Daily Tasks, KB) | Nearly identical list+edit views | 80 |
| Identical SVG icons (edit, trash, close, search) | Repeated across admin/modals | 50 |

---

## 3. Move Default Data to Separate File (~650 lines)
**Effort:** 30 minutes | **Impact:** ~15% smaller index.html

Lines 401-1039 contain the full text of 5 SOPs and 11 daily tasks hardcoded as JavaScript strings. Since Firestore is the real data store and these only exist for first-time seeding, they could be moved to a separate JSON file (e.g. `/data/defaults.json`) loaded on demand.

---

## 4. ~~Security: Replace Hardcoded Login~~ DONE
~~Login was purely cosmetic — anyone could bypass via DevTools.~~

Completed 2026-02-25: Firebase Authentication with email+password, admin-managed accounts, persistent sessions, friendly error messages, help text linking to sep@aircat.dk.

---

## 5. Add Firestore Security Rules
**Effort:** 30 minutes | **Impact:** Data protection

Now that Firebase Auth is in place, add Firestore security rules to restrict read/write to authenticated users only. Currently the Firestore config is visible in client-side code and data is accessible to anyone who knows it.

Example rules:
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if request.auth != null;
    }
  }
}
```

Natural next step after completing Firebase Auth.

---

## 6. Fix useEffect Dependencies
**Effort:** 15 minutes | **Impact:** Prevents future stale-data bugs

Several `useEffect` hooks reference state variables not listed in their dependency arrays:
- Deep-link effects in SOPArchivePage, DailyTasksPage, KnowledgeBasePage use `sops`/`dailyTasks`/`knowledgeBase` but don't include them as deps
- Works now because data rarely changes after load, but violates React rules

---

## 7. Memoize Expensive Render Operations
**Effort:** 30 minutes | **Impact:** Future-proofing performance

- `processGlossaryTerms()` runs complex regex matching on every render — should be wrapped in `useMemo`
- Contacts page sorts its list on every render without `useMemo`
- No components use `React.memo` — entire tree re-renders on any state change in App
- Not noticeable at current data size, but could get sluggish as glossary/SOPs grow

---

## 8. Persist Chat History Across Navigation
**Effort:** 15-30 minutes | **Impact:** Better UX

Chat messages in AskCATPage are lost when navigating to another page and back. Could lift chat state to the App component or persist in sessionStorage.

---

## 9. Reduce Prop Drilling
**Effort:** 1 hour | **Impact:** Cleaner code, easier to extend

The App component passes 36 props to AppShell, which forwards most to AdminPage, which forwards further. React Context could simplify this significantly. Code smell, not a bug.

---

## Priority Order

| # | Item | Effort | Impact |
|---|---|---|---|
| 1 | React production builds | 2 min | Faster app |
| 2 | Firestore security rules | 30 min | Data protection |
| 3 | Extract duplicated code | 1-2 hrs | ~400 fewer lines |
| 4 | Move default data to file | 30 min | ~650 fewer lines |
| 5 | Fix useEffect deps | 15 min | Prevents bugs |
| 6 | Memoize expensive renders | 30 min | Performance |
| 7 | Persist chat history | 15-30 min | Better UX |
| 8 | Reduce prop drilling | 1 hr | Cleaner code |

Items 1-5 together would bring the file from ~4200 lines to ~3100-3200 lines and make the app faster, safer, and more maintainable — all without changing how anything works or looks.
