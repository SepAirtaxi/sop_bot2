# SOP Assistant

An AI-powered Standard Operating Procedure assistant for employee onboarding. Built with vanilla JavaScript, React (via CDN), and Google's Gemini AI.

## Features

- **Two Interaction Modes:**
  - **View Documentation** - Chat-based search through SOPs
  - **Step-by-Step Guide** - Card-based wizard that walks users through procedures

- **Smart AI Guidance:**
  - Asks clarifying questions before providing steps
  - Recognizes context (e.g., EU vs non-EU shipments)
  - Cross-references multiple SOPs when needed

- **Admin Panel:**
  - Add/edit/delete SOPs
  - Manage categories
  - Copy-paste from Google Docs with auto section detection

## Project Structure

```
sop-bot/
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
cd sop-bot

# Initialize git
git init

# Add all files
git add .

# Commit
git commit -m "Initial commit - SOP Assistant"

# Add your GitHub repo as remote (create repo on GitHub first)
git remote add origin https://github.com/YOUR_USERNAME/sop-bot.git

# Push
git push -u origin main
```

### Step 2: Deploy to Vercel

1. Go to [vercel.com](https://vercel.com) and sign in with GitHub
2. Click **"Add New Project"**
3. Import your `sop-bot` repository
4. Before deploying, add your environment variable:
   - Click **"Environment Variables"**
   - Add: `GEMINI_API_KEY` = `your-api-key-here`
5. Click **"Deploy"**

### Step 3: Access Your App

After deployment, Vercel will give you a URL like:
`https://sop-bot-xxxxx.vercel.app`

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
- **Storage:** Browser localStorage (SOPs persist locally)

## Future Enhancements

- [ ] Firebase integration for cloud SOP storage
- [ ] User authentication with multiple users
- [ ] Analytics dashboard
- [ ] PDF export of step-by-step guides
- [ ] Multi-language support

## Troubleshooting

### API errors after deployment

- Check that `GEMINI_API_KEY` is set in Vercel Environment Variables
- Redeploy after adding/changing environment variables

### SOPs not saving

- SOPs are stored in browser localStorage
- Clear localStorage to reset to defaults
- For persistent storage across devices, Firebase integration is needed

## License

Private - CAT Flyservice Internal Use

---

Built with ❤️ for CAT Flyservice
