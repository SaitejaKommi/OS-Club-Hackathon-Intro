# Windows: Installing OpenCode and Setting Up OmniRoute APIs from Scratch

This guide assumes a clean Windows 10 or 11 PC with no developer tools installed. Open **PowerShell** (Start menu → type `PowerShell` → Open). You do not need to run it as administrator. Run each command in order.

> Placeholder: *Setup overview diagram (SVG) goes here: PowerShell → OpenCode → OmniRoute (localhost:20128) → AI providers.*

---

## 1. Prerequisites

### Step 1.1: Check that winget is available

```powershell
winget --version
```

`winget` is the Windows package manager built into Windows 10 (version 1809 or later) and Windows 11. Expected: a version number such as `v1.x.x`. If you see `not recognized`, install **App Installer** from the Microsoft Store, then reopen PowerShell.

### Step 1.2: Allow npm scripts to run

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned -Force
```

Windows blocks PowerShell scripts by default, which can stop `npm` from running. This command allows locally created scripts for your user account only. It does not need administrator rights.

### Step 1.3: Install Node.js LTS (includes npm)

```powershell
winget install --id OpenJS.NodeJS.LTS -e --accept-package-agreements --accept-source-agreements
```

Installs the long-term-support version of Node.js, which includes npm. Node runs OmniRoute, and npm installs OpenCode and OmniRoute.

### Step 1.4: Restart PowerShell

Close the PowerShell window completely and open a new one. The new install is not available in the old window until you do this.

### Step 1.5: Verify prerequisites

```powershell
node -v
npm -v
```

Expected: Node v20 or newer and npm 10 or newer.

### Optional: Docker Desktop

You do not need Docker for this guide. Install it only if you want to run OmniRoute in a container:

```powershell
winget install --id Docker.DockerDesktop -e --accept-package-agreements --accept-source-agreements
```

Restart Windows when prompted, then open Docker Desktop and wait for it to say "Engine running."

---

## 2. Install OpenCode

The `curl ... | bash` installer used on macOS does not run in Windows PowerShell. On Windows, install OpenCode through npm.

### Step 2.1: Install OpenCode globally

```powershell
npm install -g opencode-ai
```

Downloads OpenCode and makes the `opencode` command available system-wide.

### Step 2.2: Verify OpenCode

```powershell
opencode --version
```

Prints the installed version. If you see `not recognized`, close and reopen PowerShell, then try again.

---

## 3. Install and Configure OmniRoute

OmniRoute is a local gateway. OpenCode sends its requests to OmniRoute, which forwards them to your AI providers through one OpenAI-compatible endpoint: `http://localhost:20128/v1`.

### Step 3.1: Install OmniRoute

```powershell
npm install -g omniroute
```

Installs the `omniroute` command globally.

### Step 3.2: Start OmniRoute

```powershell
omniroute
```

Starts the gateway on `localhost:20128`. **Leave this PowerShell window open** while you use OpenCode. Open a second PowerShell window for the commands below.

### Step 3.3: Create an OmniRoute API key

1. In a browser, open `http://localhost:20128/dashboard/api-manager`.
2. Complete any first-run setup, then click **Create API Key**.
3. Copy the key (it starts with `sk-`). Treat it like a password.

### Step 3.4: Store the key as an environment variable

Replace `sk-your-omniroute-key` with the key you copied:

```powershell
[Environment]::SetEnvironmentVariable("OMNI_OPENCODE_API_KEY", "sk-your-omniroute-key", "User")
```

Saves the key for your Windows user account so OpenCode can read it without the key appearing in a config file. Environment variables set this way apply only to windows opened after this command.

Close the second PowerShell window and open a new one before continuing.

Confirm it saved:

```powershell
$env:OMNI_OPENCODE_API_KEY
```

Expected: your `sk-...` key. If the output is empty, reopen PowerShell and try again.

### Step 3.5: Create the OpenCode config file

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\.config\opencode" | Out-Null

@'
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "omniroute": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "OmniRoute",
      "options": {
        "baseURL": "http://localhost:20128/v1",
        "apiKey": "{env:OMNI_OPENCODE_API_KEY}"
      },
      "models": {
        "auto": {
          "name": "OmniRoute (Auto)"
        }
      }
    }
  }
}
'@ | Set-Content -Path "$HOME\.config\opencode\opencode.json" -Encoding UTF8
```

Creates the config folder and writes `opencode.json`. The `@' ... '@` block is a literal here-string, so PowerShell does not change the text inside it. The `{env:...}` placeholder tells OpenCode to read the key from the environment variable saved in Step 3.4.

> If you already have an `opencode.json`, do not overwrite it. Open it in Notepad and add only the `"provider"` block.

### Step 3.6: Keep OmniRoute local (recommended)

No command is needed. OmniRoute listens on `localhost` by default, so it is reachable only from this PC. Do not open port 20128 in Windows Firewall or your router unless you have added authentication and understand the risks.

---

## 4. Verification and Test Commands

### Step 4.1: Confirm OmniRoute is responding

```powershell
Invoke-RestMethod -Uri "http://localhost:20128/v1/models" -Headers @{ Authorization = "Bearer $env:OMNI_OPENCODE_API_KEY" }
```

Expected: a list of models returned as structured output. A `Unable to connect` error means OmniRoute is not running (go back to Step 3.2). A `401 Unauthorized` error means the key is wrong (check Steps 3.3 and 3.4).

### Step 4.2: Confirm the config file is valid JSON

```powershell
Get-Content "$HOME\.config\opencode\opencode.json" -Raw | ConvertFrom-Json | Out-Null; "Config OK"
```

Prints `Config OK` if the file parses. If you see an error, open the file in Notepad and fix the commas and brackets.

### Step 4.3: Send a test prompt through OmniRoute

```powershell
opencode run "Reply with one word: ready" --model omniroute/auto
```

Expected: a short reply from the model. If it fails, check that `omniroute` is still running in the other window.

### Step 4.4: Start OpenCode interactively

```powershell
cd $HOME
opencode
```

Opens the OpenCode interface. Use the model picker to choose **OmniRoute (Auto)**. Press `Ctrl+C` to exit.

### Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `opencode : ... not recognized` | PowerShell opened before install | Close and reopen PowerShell |
| `running scripts is disabled` | Execution policy blocks npm | Redo Step 1.2 |
| `Unable to connect` | OmniRoute not running | Run `omniroute` in a separate window |
| `401 Unauthorized` | Wrong or missing key | Recreate the key in the dashboard and redo Step 3.4 |
| `$env:OMNI_OPENCODE_API_KEY` is empty | Window opened before the variable was saved | Open a new PowerShell window |
| Model list is empty | Models hidden in OmniRoute | Check model visibility in the OmniRoute dashboard |

> Placeholder: *Link to setup video here (Windows walkthrough).*
