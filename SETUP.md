# First-Time Setup Guide

Everything you need to get from zero to your first Claude Code session in Salesforce.

---

## Prerequisites

| Tool | Required By | Install |
|------|-------------|---------|
| Salesforce CLI (`sf`) | Everyone | See below |
| Claude Code | Everyone | See README |
| Git | Developer + Team Lead | [git-scm.com](https://git-scm.com/) |
| Node.js (LTS) | Developer only | [nodejs.org](https://nodejs.org/) |

---

## Step 1: Install Salesforce CLI (Windows, no admin rights needed)

You do **not** need admin rights for this. Use the portable installer:

1. Go to [https://developer.salesforce.com/tools/salesforcecli](https://developer.salesforce.com/tools/salesforcecli)
2. Download **"Windows (x64) - tar.xz"** (not the installer — the archive)
3. Extract it anywhere you have write access, e.g. `C:\Users\YourName\Tools\sf`
4. Add the `bin` folder to your PATH:
   - Open **Start → Search → "Edit the system environment variables"**
   - Click **Environment Variables**
   - Under **User variables** (not System), select **Path** → Edit → New
   - Add: `C:\Users\YourName\Tools\sf\bin`
   - Click OK — restart your terminal
5. Verify: open a new terminal and run:
   ```
   sf --version
   ```

> **No terminal?** Use Windows Terminal (install from Microsoft Store — no admin needed) or PowerShell.

---

## Step 2: Authenticate to Your Salesforce Orgs

Run this once per org. A browser window will open — log in with your Salesforce credentials:

```bash
# Dev Sandbox
sf org login web --alias DEV_SANDBOX --instance-url https://test.salesforce.com

# UAT Sandbox (Team Lead only)
sf org login web --alias UAT_SANDBOX --instance-url https://test.salesforce.com
```

> Replace `DEV_SANDBOX` and `UAT_SANDBOX` with the aliases from `CLAUDE.md`.
> For Production: `--instance-url https://login.salesforce.com`

Verify your connections:
```bash
sf org list
```

---

## Step 3: Set Up the Salesforce DX Project (First Time Only)

The skills expect a standard Salesforce DX project structure. If you don't have one yet:

```bash
# In the folder where you want your project (e.g. C:\Projects\MySalesforceProject)
sf project generate --name MySalesforceProject --output-dir .

# This creates:
# force-app/main/default/   ← your metadata lives here
# sfdx-project.json         ← project config
```

Then copy the Claude Code files into that folder:
```bash
cp -r .claude CLAUDE.md .gitignore .deployments your-project-folder/
```

Or clone this repo directly into an empty project folder:
```bash
git clone https://github.com/NicoIO-beep/sf_cc_setup.git .
sf project generate --name MySalesforceProject --output-dir . --template standard
```

---

## Step 4: Pull Existing Metadata from Your Org

If your org already has Apex classes, LWC, Flows — pull them into your project:

```bash
# Pull all Apex classes
sf project retrieve start --metadata "ApexClass" -o DEV_SANDBOX

# Pull all LWC components
sf project retrieve start --metadata "LightningComponentBundle" -o DEV_SANDBOX

# Pull everything (takes a while)
sf project retrieve start --metadata "ApexClass,ApexTrigger,LightningComponentBundle,Flow,CustomObject" -o DEV_SANDBOX
```

Your metadata lands in `force-app/main/default/`. From this point, Claude Code can work with it.

---

## Step 5: Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

> Requires Node.js (LTS). If `npm install -g` fails due to permissions, run:
> ```bash
> npm install -g @anthropic-ai/claude-code --prefix C:\Users\YourName\AppData\Roaming\npm
> ```

Then start it in your project folder:
```bash
claude
```

---

## Step 6: Replace Org Aliases

Open `CLAUDE.md` and the 3 skill files, replace the placeholders:

| Placeholder | Replace with |
|-------------|-------------|
| `DEV_SANDBOX` | Your dev sandbox alias (e.g. `MySandbox_Dev`) |
| `UAT_SANDBOX` | Your UAT sandbox alias (e.g. `MySandbox_UAT`) |

Quick find-and-replace (run in your project folder):
```bash
# On Windows PowerShell:
Get-ChildItem -Recurse -Include "*.md" | ForEach-Object {
    (Get-Content $_.FullName) -replace 'DEV_SANDBOX','YOUR_DEV_ALIAS' -replace 'UAT_SANDBOX','YOUR_UAT_ALIAS' | Set-Content $_.FullName
}

# On bash/Git Bash:
grep -rl "DEV_SANDBOX" . --include="*.md" | xargs sed -i 's/DEV_SANDBOX/YOUR_DEV_ALIAS/g'
grep -rl "UAT_SANDBOX" . --include="*.md" | xargs sed -i 's/UAT_SANDBOX/YOUR_UAT_ALIAS/g'
```

---

## Troubleshooting

**`sf: command not found` after install**
→ The PATH was not updated. Close and reopen your terminal. If still not found, check that the `bin` folder path was added correctly to User variables (not System variables).

**`sf org login web` opens browser but fails to redirect**
→ Try a different browser. Make sure pop-ups are not blocked. If behind a corporate proxy, ask IT for the proxy settings.

**`sf org list` shows orgs as expired**
→ Re-authenticate: `sf org login web --alias DEV_SANDBOX --instance-url https://test.salesforce.com`

**`sf project generate` fails with "directory not empty"**
→ The folder already has files. Use `--force` flag or create a new empty folder first.

**Claude Code won't start (`npm install -g` failed)**
→ Install Node.js from [nodejs.org](https://nodejs.org/) first. If permissions block global install, use the prefix workaround shown in Step 5.

**`force-app/main/default/` folder is empty after retrieve**
→ Check that your org alias is correct (`sf org list`) and that the metadata type exists in the org. Try a single type first: `sf project retrieve start --metadata "ApexClass" -o DEV_SANDBOX`
