# macOS: Installing OpenCode and Setting Up OmniRoute APIs from Scratch

This guide assumes a clean Mac with no developer tools installed. Open **Terminal** (Applications → Utilities → Terminal) and run each command in order. The commands work in both zsh (the default on modern macOS) and bash.

> Placeholder: *Setup overview diagram (SVG) goes here: Terminal → OpenCode → OmniRoute (localhost:20128) → AI providers.*

---

## 1. Prerequisites

### Step 1.1: Install Apple's command line tools

```bash
xcode-select --install
```

Installs Git and the compiler tools that Homebrew needs. A pop-up appears; click **Install** and wait for it to finish. If you see "command line tools are already installed," skip ahead.

### Step 1.2: Install Homebrew (package manager)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Downloads and runs the official Homebrew installer. It asks for your Mac password. Follow the "Next steps" it prints.

### Step 1.3: Add Homebrew to your PATH (Apple Silicon Macs only)

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

Makes the `brew` command available in this and future Terminal windows. Intel Macs can skip this step.

### Step 1.4: Install Node.js (includes npm)

```bash
brew install node
```

Node.js runs OmniRoute, and npm (its package manager) installs OpenCode and OmniRoute.

### Step 1.5: Verify prerequisites

```bash
node -v
npm -v
curl --version | head -n 1
```

Expected: Node v20 or newer, npm 10 or newer, and a curl version line.

### Optional: Docker Desktop

You do not need Docker for this guide. Install it only if you want to run OmniRoute in a container:

```bash
brew install --cask docker
open -a Docker
```

Wait for the whale icon in the menu bar to stop animating before using `docker` commands.

---

## 2. Install OpenCode

### Step 2.1: Run the official installer

```bash
curl -fsSL https://opencode.ai/v2/install | bash
```

Downloads the installer and runs it. `-f` fails on server errors, `-s` runs quietly, `-S` still shows errors, and `-L` follows redirects.

### Step 2.2: Reload your shell

```bash
source ~/.zshrc 2>/dev/null || source ~/.bashrc
```

Refreshes PATH so the new `opencode` command is found without reopening Terminal.

### Step 2.3: Verify OpenCode

```bash
opencode --version
```

Prints the installed version. If you see `command not found`, open a new Terminal window and try again. If it still fails, use the npm fallback:

```bash
npm install -g opencode-ai
opencode --version
```

---

## 3. Install and Configure OmniRoute

OmniRoute is a local gateway. OpenCode sends its requests to OmniRoute, which forwards them to your AI providers through one OpenAI-compatible endpoint: `http://localhost:20128/v1`.

### Step 3.1: Install OmniRoute

```bash
npm install -g omniroute
```

Installs the `omniroute` command globally.

### Step 3.2: Start OmniRoute

```bash
omniroute
```

Starts the gateway on `localhost:20128`. **Leave this Terminal window open** while you use OpenCode. Open a second window for the commands below.

### Step 3.3: Create an OmniRoute API key

1. In a browser, open `http://localhost:20128/dashboard/api-manager`.
2. Complete any first-run setup, then click **Create API Key**.
3. Copy the key (it starts with `sk-`). Treat it like a password.

### Step 3.4: Store the key as an environment variable

Replace `sk-your-omniroute-key` with the key you copied:

```bash
echo 'export OMNI_OPENCODE_API_KEY="sk-your-omniroute-key"' >> ~/.zshrc
chmod 600 ~/.zshrc
source ~/.zshrc
```

Saves the key so OpenCode can read it without you pasting it into a config file. `chmod 600` limits read access to your user account. Keeping the key in an environment variable means it does not appear in the config file you share or commit.

> Using bash instead of zsh? Replace `~/.zshrc` with `~/.bashrc`.

### Step 3.5: Create the OpenCode config file

```bash
mkdir -p ~/.config/opencode
cat > ~/.config/opencode/opencode.json << 'EOF'
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
EOF
```

Creates the config folder and writes `opencode.json`. The quoted `'EOF'` stops Terminal from changing the `$` or `{}` text. The `{env:...}` placeholder tells OpenCode to read the key from the environment variable you saved in Step 3.4.

> If you already have an `opencode.json`, do not overwrite it. Open it in an editor and add only the `"provider"` block.

### Step 3.6: Keep OmniRoute local (recommended)

No command is needed. OmniRoute listens on `localhost` by default, so it is reachable only from this Mac. Do not expose port 20128 to your network or the internet unless you have added authentication and understand the risks.

---

## 4. Verification and Test Commands

### Step 4.1: Confirm OmniRoute is responding

```bash
curl -s http://localhost:20128/v1/models \
  -H "Authorization: Bearer $OMNI_OPENCODE_API_KEY"
```

Expected: a JSON list of models. A `Connection refused` error means OmniRoute is not running (go back to Step 3.2). A `401 Unauthorized` error means the key is wrong (check Step 3.3 and 3.4).

### Step 4.2: Confirm the config file is valid JSON

```bash
python3 -m json.tool ~/.config/opencode/opencode.json > /dev/null && echo "Config OK"
```

Prints `Config OK` if the file parses. If you see an error, reopen the file and fix the commas and brackets.

### Step 4.3: Send a test prompt through OmniRoute

```bash
opencode run "Reply with one word: ready" --model omniroute/auto
```

Expected: a short reply from the model. If it fails, check that `omniroute` is still running in the other window.

### Step 4.4: Start OpenCode interactively

```bash
cd ~
opencode
```

Opens the OpenCode interface. Use the model picker to choose **OmniRoute (Auto)**. Press `Ctrl+C` to exit.

### Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `opencode: command not found` | PATH not refreshed | Open a new Terminal window |
| `Connection refused` | OmniRoute not running | Run `omniroute` in a separate window |
| `401 Unauthorized` | Wrong or missing key | Recreate the key in the dashboard and redo Step 3.4 |
| OmniRoute missing from model list | Config not loaded | Check Step 4.2, then restart OpenCode |
| Model list is empty | Models hidden in OmniRoute | Check model visibility in the OmniRoute dashboard |

> Placeholder: *Link to setup video here (macOS walkthrough).*
