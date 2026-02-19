# First-Time Setup Guide

Step-by-step from zero to your first Claude Code session in Salesforce.
**Each step checks first — only install if something is missing.**

---

## Step 1: Check & Install Salesforce CLI

**Check first:**
```bash
sf --version
```
→ Shows a version (e.g. `@salesforce/cli/2.x.x`)? **Skip to Step 2.**
→ `command not found`? Install below.

**Install (no admin rights needed):**
1. Go to [https://developer.salesforce.com/tools/salesforcecli](https://developer.salesforce.com/tools/salesforcecli)
2. Download **"Windows (x64) - tar.xz"** — the archive, not the installer
3. Extract it anywhere you have write access, e.g. `C:\Users\YourName\Tools\sf`
4. Add the `bin` folder to your PATH:
   - Start → Search → **"Edit the system environment variables"**
   - Click **Environment Variables**
   - Under **User variables** (not System) → select **Path** → Edit → New
   - Add: `C:\Users\YourName\Tools\sf\bin`
   - Click OK — restart your terminal
5. Verify: open a new terminal and run `sf --version`

---

## Step 2: Check & Install Git

**Check first:**
```bash
git --version
```
→ Shows a version? **Skip to Step 3.**
→ `command not found`? Install below.

**Install:**
- Download from [git-scm.com](https://git-scm.com/) — no admin needed if you choose "Install for current user only"
- During install: keep all defaults, just click Next
- Verify: open a new terminal and run `git --version`

---

## Step 3: Check & Install Node.js

**Check first:**
```bash
node --version
```
→ Shows a version (e.g. `v20.x.x`)? **Skip to Step 4.**
→ `command not found`? Install below.

**Install:**
- Download the **LTS** version from [nodejs.org](https://nodejs.org/)
- No admin needed if you choose "Install for current user only"
- Verify: open a new terminal and run `node --version`

---

## Step 4: Check & Install Claude Code

**Check first:**
```bash
claude --version
```
→ Shows a version? **Skip to Step 5.**
→ `command not found`? Install below.

**Install:**
```bash
npm install -g @anthropic-ai/claude-code
```

If that fails due to permissions:
```bash
npm install -g @anthropic-ai/claude-code --prefix C:\Users\YourName\AppData\Roaming\npm
```

Verify: `claude --version`

---

## Step 5: Clone the Repo

**Check first — is the project already cloned?**
```bash
ls CLAUDE.md
```
→ File exists? **Skip to Step 6** — you already have the project.
→ `No such file`? Clone it:

**In VS Code:**
- `Ctrl+Shift+P` → type **"Git: Clone"** → paste the repo URL → choose your project folder → click **"Open"**

**Or in terminal:**
```bash
git clone https://github.com/NicoIO-beep/sf_cc_setup.git .
```

---

## Step 6: Authenticate to Your Salesforce Orgs

**Check first:**
```bash
sf org list
```
→ Your org aliases already listed? **Skip to Step 7.**
→ Empty or missing? Authenticate:

```bash
# Dev Sandbox
sf org login web --alias DEV_SANDBOX --instance-url https://test.salesforce.com

# UAT Sandbox (Team Lead only)
sf org login web --alias UAT_SANDBOX --instance-url https://test.salesforce.com
```

> Replace `DEV_SANDBOX` and `UAT_SANDBOX` with the aliases from `CLAUDE.md`.
> For Production: use `--instance-url https://login.salesforce.com`

A browser window opens — log in with your Salesforce credentials.

---

## Step 7: Set Up the Salesforce DX Project Structure

**Check first:**
```bash
ls force-app
```
→ Folder exists? **Skip to Step 8.**
→ Missing? Create it:

```bash
sf project generate --name MySalesforceProject --output-dir .
```

This creates `force-app/main/default/` — the folder where all your metadata lives.

---

## Step 8: Pull Existing Metadata from Your Org

**Check first:**
```bash
ls force-app/main/default/classes
```
→ Apex classes already there? **Skip this step.**
→ Empty or missing? Pull from your org:

```bash
# Pull Apex classes
sf project retrieve start --metadata "ApexClass" -o DEV_SANDBOX

# Pull LWC components
sf project retrieve start --metadata "LightningComponentBundle" -o DEV_SANDBOX

# Pull everything at once (takes a while)
sf project retrieve start --metadata "ApexClass,ApexTrigger,LightningComponentBundle,Flow,CustomObject" -o DEV_SANDBOX
```

---

## Step 9: Replace Org Aliases

**Check first — are the placeholders already replaced?**
```bash
grep -r "DEV_SANDBOX" CLAUDE.md
```
→ No output? Aliases are already replaced. **Skip to Step 10.**
→ Shows matches? Replace them:

**PowerShell:**
```powershell
Get-ChildItem -Recurse -Include "*.md" | ForEach-Object {
    (Get-Content $_.FullName) -replace 'DEV_SANDBOX','YOUR_DEV_ALIAS' -replace 'UAT_SANDBOX','YOUR_UAT_ALIAS' | Set-Content $_.FullName
}
```

**Git Bash / bash:**
```bash
grep -rl "DEV_SANDBOX" . --include="*.md" | xargs sed -i 's/DEV_SANDBOX/YOUR_DEV_ALIAS/g'
grep -rl "UAT_SANDBOX" . --include="*.md" | xargs sed -i 's/UAT_SANDBOX/YOUR_UAT_ALIAS/g'
```

> **Windows note:** `sed -i` in Git Bash on Windows can behave differently depending on the Git installation — it may create backup files or fail silently. If the replacement doesn't work, use the **PowerShell variant above** — it is the safer choice on Windows.

---

## Step 10: Start Claude Code

```bash
claude
```

Then use a skill:
```
/build   Write me an Apex trigger for Account
/data    SOQL query for all open Opportunities this quarter
/deploy  Deploy AccountTrigger from dev to UAT
```

You're ready.

---

## Troubleshooting

**`sf: command not found` after install**
→ Close and reopen your terminal. Check that the `bin` path was added to **User variables**, not System variables.

**`sf org login web` opens browser but fails to redirect**
→ Try a different browser. Make sure pop-ups are not blocked.

**`sf org list` shows orgs as expired**
→ Re-authenticate: `sf org login web --alias DEV_SANDBOX --instance-url https://test.salesforce.com`

**`sf project generate` fails with "directory not empty"**
→ The folder already has files. Either use `--force` or skip — your existing project is fine.

**`npm install -g` fails with permission error**
→ Use the `--prefix` workaround shown in Step 4.

**`force-app/main/default/classes` is empty after retrieve**
→ Check org alias with `sf org list`. Try retrieving a single class first to verify the connection works.
