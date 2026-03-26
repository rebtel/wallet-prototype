# How to deploy a prototype to Vercel via GitHub

## What you'll need before starting

- A GitHub account with access to the **rebtel** organisation
- A Vercel account with **Developer** role (or higher) on the **Rebtel** organisation in Vercel
- Claude Code installed

## Important: Vercel organisation

When deploying on Vercel, make sure you are deploying under the **Rebtel organisation**, not your personal Vercel account. Vercel sometimes defaults to your personal user — double check the scope/team selector at the top of the Vercel dashboard before importing a project.

## Instructions to paste into Claude Code

Copy and paste the following prompt into Claude Code from your project directory:

---

```
I have a prototype I want to deploy so my team can see it. Help me do the following:

1. Initialise a git repo in this directory (if not already done)
2. Make sure the main HTML file is named index.html (Vercel requires this)
3. Install the GitHub CLI (gh) via Homebrew if not already installed
4. Authenticate with GitHub if not already logged in (gh auth login)
5. Create a new public repo under the "rebtel" GitHub organisation with a descriptive name based on my project
6. Push the code to the repo
7. Then tell me to go to https://vercel.com/new to import the repo and deploy it

Remind me to:
- Make sure I have Developer access on the Rebtel organisation in Vercel
- Check that I'm deploying under the Rebtel team in Vercel, not my personal account (Vercel sometimes defaults to personal)

After this is set up, every push to main will auto-deploy.
```

---

## Updating the prototype after initial deploy

Once the repo is connected to Vercel, any changes pushed to `main` will auto-deploy within seconds. In Claude Code, just ask it to make changes and then say "push to GitHub" — the live URL will update automatically.
