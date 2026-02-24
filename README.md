# CAT Onboard

An AI-powered onboarding assistant for CAT Flyservice. Built with React (via CDN), Tailwind CSS, and Google's Gemini AI.

## Features

### Ask CAT (AI Chat)
The main page. An AI-powered chat interface using Google Gemini 2.5 Flash. New hires can ask free-form questions about procedures, contacts, terms, or anything related to CAT Flyservice. The AI has access to all SOPs, glossary, contacts, daily tasks, and knowledge base content. Supports two modes:
- **Chat mode** — conversational Q&A with Markdown-rendered responses
- **Wizard mode** — generates a step-by-step guided process as interactive cards

### Contacts
Internal company directory displayed as a sortable table. Shows name, role, phone number, and email (with clickable mailto links) for each employee.

### Daily Tasks
Searchable list of recurring daily tasks and procedures. Uses a two-panel layout: task list on the left, full Markdown content viewer on the right. Tasks are organized by category.

### Glossary
Aviation abbreviations, technical terms, and company-specific definitions. Filterable by category with a search bar that matches both term names and meanings. Entries are split into abbreviations (e.g., AOG, MEL) and general terms.

### Knowledge Base
A library of articles and reference material for new hires. Two-panel layout with category filter tabs, search, and full Markdown content viewer. Content is managed through the Admin panel.

### Pricing
Markup tables for customers and entities. Two-panel layout with a searchable entity list on the left and detailed pricing fields on the right. Each entity has custom label/value pairs and optional notes.

### SOP Archive
Browsable archive of all Standard Operating Procedures. Two-panel layout with category tabs, search, and a full Markdown viewer. Each SOP can also be launched as an AI-generated step-by-step wizard guide.

### Admin Panel
Tabbed management interface for all content types. Administrators can create, edit, and delete entries across six tabs: SOPs, Daily Tasks, Knowledge Base, Glossary, Contacts, and Pricing. SOP content supports Markdown and paste-from-Google-Docs with auto section detection.

## Project Structure

```
cat-onboard/
├── index.html          # Main application (React via CDN)
├── api/
│   └── chat.js         # Serverless function for Gemini API
├── vercel.json         # Vercel deployment config
├── package.json        # Project metadata
└── README.md           # This file
```

## Quick Start (Local Testing)

For local testing without deployment, you can use the standalone version with the API key embedded. **Only use this for testing, not production.**

1. Open `index.html` in a browser
2. Login with: `cat` / `cat1234`

Note: The deployed version uses a serverless function to keep the API key secure.

## Deployment to Vercel

### Prerequisites

- GitHub account
- Vercel account (free tier works fine)
- Gemini API key from [Google AI Studio](https://aistudio.google.com/app/apikey)

### Step 1: Push to GitHub

```bash
# Navigate to project folder
cd cat-onboard

# Initialize git
git init

# Add all files
git add .

# Commit
git commit -m "Initial commit - CAT Onboard"

# Add your GitHub repo as remote (create repo on GitHub first)
git remote add origin https://github.com/YOUR_USERNAME/cat-onboard.git

# Push
git push -u origin main
```

### Step 2: Deploy to Vercel

1. Go to [vercel.com](https://vercel.com) and sign in with GitHub
2. Click **"Add New Project"**
3. Import your `cat-onboard` repository
4. Before deploying, add your environment variable:
   - Click **"Environment Variables"**
   - Add: `GEMINI_API_KEY` = `your-api-key-here`
5. Click **"Deploy"**

### Step 3: Access Your App

After deployment, Vercel will give you a URL like:
`https://cat-onboard-xxxxx.vercel.app`

Share this with your team!

## Configuration

### Changing Login Credentials

Edit the `CONFIG.credentials` object in `index.html`:

```javascript
credentials: { username: 'your-username', password: 'your-password' }
```

### Adding/Editing SOPs

1. Login to the app
2. Click the gear icon (⚙️) to access Admin Panel
3. Click "Add New SOP" or edit existing ones
4. Paste content from Google Docs - sections are auto-detected

### Customizing for Your Company

1. Update the `DEFAULT_SOPS` array in `index.html` with your procedures
2. Change branding (title, colors) in the HTML/CSS
3. Modify the system prompts in `GeminiService.buildSystemPrompt()` for your context

## Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `GEMINI_API_KEY` | Google Gemini API key | Yes |

## Tech Stack

- **Frontend:** React 18 (via CDN), Tailwind CSS
- **AI:** Google Gemini 2.5 Flash
- **Hosting:** Vercel (serverless functions + static hosting)
- **Database:** Firebase Firestore (with localStorage fallback)

## Future Enhancements

- [ ] User authentication with multiple users/roles
- [ ] Analytics dashboard
- [ ] PDF export of step-by-step guides
- [ ] Multi-language support (Danish/English)

## Troubleshooting

### API errors after deployment

- Check that `GEMINI_API_KEY` is set in Vercel Environment Variables
- Redeploy after adding/changing environment variables

### SOPs not saving

- Data is stored in Firebase Firestore with localStorage as fallback
- Clear localStorage to reset to defaults if Firebase is unavailable

## License

Private - CAT Flyservice Internal Use

---

Built for CAT Flyservice
