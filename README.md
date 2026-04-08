# Claude Code with Azure AI Foundry

A practical guide to running Claude Code using Azure-hosted Claude models via Microsoft Foundry — with persistent shell setup and multi-model switching.

> This guide improves on the [original tutorial](https://github.com/xomicsdatascience/ClaudeCode_with_Azure) with step-by-step instructions and a reusable shell workflow that works for any number of deployed models.

---

## Prerequisites

Before starting, make sure you have:

- Claude Code installed (`npm install -g @anthropic-ai/claude-code`)
- Access to [Azure AI Foundry](https://ai.azure.com) with a paid Azure subscription
- At least one Claude model deployed in your Foundry resource (see Step 1 below)

---

## Step 1: Deploy Claude Models in Azure AI Foundry

1. Go to [ai.azure.com](https://ai.azure.com) and log in
2. Navigate to **Model catalog** → search for Claude → select your preferred model → click **Deploy**
3. Give your deployment a name — this becomes your `deployment-name`, used later with `claude --model`
4. Repeat for any additional models you want to use
5. Once deployed, go to **My assets → Models + endpoints** to confirm they're active

**Note your resource name** — found in the **Overview** page under *Endpoints and keys*, in a URL like:
```
https://{resource-name}.services.ai.azure.com
```
The `{resource-name}` part is what you'll use throughout this guide.

**Example:** if your endpoint URL is `https://my-lab-resource.services.ai.azure.com`, your resource name is `my-lab-resource`.

You can name your deployments anything you like — just keep track of them:
```
# Example deployment names you might choose:
my-sonnet    → deployed claude-sonnet-4-6
my-haiku     → deployed claude-haiku-4-5
my-opus      → deployed claude-opus-4-5
```

---

## Step 2: Get Your API Key(s)

1. In [ai.azure.com](https://ai.azure.com), go to your resource's **Overview → Endpoints and keys**
2. Copy your API key
3. Save it to a secure file — **do not paste it directly into the terminal** (it will be saved in your command history)

```bash
# Create a folder for your keys
mkdir -p ~/api_keys

# Save your key — one file per model if they have different keys,
# or one shared file if all models use the same key
echo "paste-your-key-here" > ~/api_keys/azure_key.txt

# Restrict permissions so only you can read it
chmod 600 ~/api_keys/azure_key.txt
```

> **Tip:** Most Azure resources share one API key across all deployed models. If that's the case, you only need one key file and can point all models to it.

**Example — shared key across all models (most common):**
```bash
mkdir -p ~/api_keys
echo "paste-your-key-here" > ~/api_keys/azure_key.txt
chmod 600 ~/api_keys/azure_key.txt
```
Both `my-sonnet` and `my-haiku` deployments point to the same `azure_key.txt` in Step 3.

**Example — separate key per model:**
```bash
mkdir -p ~/api_keys
echo "paste-sonnet-key-here" > ~/api_keys/claude_sonnet.txt
echo "paste-haiku-key-here"  > ~/api_keys/claude_haiku.txt
chmod 600 ~/api_keys/claude_sonnet.txt
chmod 600 ~/api_keys/claude_haiku.txt
```
Each deployment gets its own key file — you'll reference each path separately in the shell function (Step 3).

---

## Step 3: Set Up Persistent Shell Functions

Instead of setting environment variables manually every time, add shell functions to your `.zshrc` (or `.bashrc` if you use bash). This lets you switch between your default Anthropic account and Azure with a single command.

### Back up your shell config first

```bash
cp ~/.zshrc ~/.zshrc.backup
```

### Open your shell config

```bash
nano ~/.zshrc
```

### Paste the following at the **bottom** of the file

The template below works for any number of models — just fill in your own deployment names, resource name, and key file paths.

```bash
# ─── Claude Code: Azure AI Foundry Switching ───────────────────────────
#
# SETUP CHECKLIST:
#   [ ] Replace "your-resource-name" with your Azure resource name
#   [ ] Replace each "your-deployment-name-N" with your actual deployment names
#   [ ] Point each model to the correct key file path
#   [ ] Set your preferred default model in ${1:-...}
#
# USAGE:
#   claude-azure                        → uses your default deployment
#   claude-azure <deployment-name>      → uses a specific deployment
#   claude-default                      → switches back to Anthropic account
#   claude-mode                         → shows which mode you're in

claude-azure() {
  local model=${1:-"your-default-deployment-name"}   # ← set your default here

  export CLAUDE_CODE_USE_FOUNDRY=1
  export ANTHROPIC_FOUNDRY_RESOURCE="your-resource-name"   # ← your Azure resource name

  case $model in
    "your-deployment-name-1")
      export ANTHROPIC_FOUNDRY_API_KEY=$(cat ~/api_keys/your-key-file.txt)
      ;;
    "your-deployment-name-2")
      export ANTHROPIC_FOUNDRY_API_KEY=$(cat ~/api_keys/your-key-file.txt)
      ;;
    # Add more deployments following the same pattern:
    # "your-deployment-name-N")
    #   export ANTHROPIC_FOUNDRY_API_KEY=$(cat ~/api_keys/your-key-file.txt)
    #   ;;
    *)
      echo "❌ Unknown deployment: $model"
      echo "Available: your-deployment-name-1, your-deployment-name-2"
      return 1
      ;;
  esac

  echo "✅ Switched to Azure Foundry — $model"
  echo "👉 Run: claude --model $model"
}

# Switch back to your default Anthropic account
claude-default() {
  unset CLAUDE_CODE_USE_FOUNDRY
  unset ANTHROPIC_FOUNDRY_RESOURCE
  unset ANTHROPIC_FOUNDRY_API_KEY
  echo "✅ Switched back to Anthropic default"
}

# Check which mode you're currently in
claude-mode() {
  if [ -n "$CLAUDE_CODE_USE_FOUNDRY" ]; then
    echo "🔵 Azure Foundry ($ANTHROPIC_FOUNDRY_RESOURCE)"
  else
    echo "🟣 Anthropic default"
  fi
}

# ────────────────────────────────────────────────────────────────────────
```

### Example: two models with a shared API key

```bash
claude-azure() {
  local model=${1:-"my-sonnet"}

  export CLAUDE_CODE_USE_FOUNDRY=1
  export ANTHROPIC_FOUNDRY_RESOURCE="my-lab-resource"

  case $model in
    "my-sonnet")
      export ANTHROPIC_FOUNDRY_API_KEY=$(cat ~/api_keys/azure_key.txt)
      ;;
    "my-haiku")
      export ANTHROPIC_FOUNDRY_API_KEY=$(cat ~/api_keys/azure_key.txt)
      ;;
    *)
      echo "❌ Unknown deployment: $model"
      echo "Available: my-sonnet, my-haiku"
      return 1
      ;;
  esac

  echo "✅ Switched to Azure Foundry — $model"
  echo "👉 Run: claude --model $model"
}
```

### Example: three models with separate API keys

```bash
claude-azure() {
  local model=${1:-"my-sonnet"}

  export CLAUDE_CODE_USE_FOUNDRY=1
  export ANTHROPIC_FOUNDRY_RESOURCE="my-lab-resource"

  case $model in
    "my-sonnet")
      export ANTHROPIC_FOUNDRY_API_KEY=$(cat ~/api_keys/sonnet_key.txt)
      ;;
    "my-haiku")
      export ANTHROPIC_FOUNDRY_API_KEY=$(cat ~/api_keys/haiku_key.txt)
      ;;
    "my-opus")
      export ANTHROPIC_FOUNDRY_API_KEY=$(cat ~/api_keys/opus_key.txt)
      ;;
    *)
      echo "❌ Unknown deployment: $model"
      echo "Available: my-sonnet, my-haiku, my-opus"
      return 1
      ;;
  esac

  echo "✅ Switched to Azure Foundry — $model"
  echo "👉 Run: claude --model $model"
}
```

### Save and reload

In nano: `Ctrl+O` → Enter to save, then `Ctrl+X` to exit.

```bash
source ~/.zshrc
```

---

## Step 4: Launch Claude Code with Azure

```bash
# Switch to Azure using your default deployment, then launch
claude-azure
claude --model your-default-deployment-name

# Switch to a specific deployment, then launch
claude-azure your-deployment-name-2
claude --model your-deployment-name-2

# Check which mode you're in at any time
claude-mode

# Switch back to your regular Anthropic account
claude-default
```

> **Why two commands?**
> `claude-azure` sets the environment (tells Claude Code to use Azure and loads your credentials).
> `claude --model` launches Claude Code and selects which deployed model to use.
> The name passed to `--model` must match your **deployment name** in Azure AI Foundry exactly.

**Using the example setup from above:**
```bash
claude-azure                # switches to Azure, defaults to my-sonnet
claude --model my-sonnet    # launch with sonnet

claude-azure my-haiku       # switch to haiku
claude --model my-haiku     # launch with haiku
```

---

## Step 5: Verify the Connection

After launching Claude Code, run inside the session:

```
/status
```

You should see:
- `CLAUDE_CODE_USE_FOUNDRY: 1`
- Your resource name
- The deployment name you selected

If all three appear, you're connected successfully. ✅

---

## VS Code Setup (Optional)

Add this to your `settings.json` (`Cmd+,` → search "Claude Code" → *Edit in settings.json*):

```json
{
  "claude-code.environmentVariables": {
    "CLAUDE_CODE_USE_FOUNDRY": "1",
    "ANTHROPIC_FOUNDRY_RESOURCE": "your-resource-name",
    "ANTHROPIC_FOUNDRY_API_KEY": "your-api-key"
  }
}
```

> **Limitation:** VS Code settings are static — you can only point to one deployment at a time and must edit `settings.json` manually to switch. For multi-model switching, use the **integrated terminal** inside VS Code instead, where your `.zshrc` functions work the same way.

---

## Troubleshooting

**`permission denied: ~/.zshrc`**
```bash
ls -la ~/.zshrc              # check ownership
chmod 644 ~/.zshrc           # fix permissions
sudo chown $USER ~/.zshrc    # fix ownership if needed
```

**`zsh: command not found: claude-azure`**
```bash
source ~/.zshrc   # reload your shell config
```

**Connection fails / authentication error**
- Check that `ANTHROPIC_FOUNDRY_RESOURCE` is the resource name only — no `https://`, no trailing slash
- Confirm the key file exists and is readable: `cat ~/api_keys/azure_key.txt`
- Verify the deployment is still active in [ai.azure.com](https://ai.azure.com) under *My assets → Models + endpoints*

**`Unknown deployment` error in the shell function**
- The name you pass to `claude-azure` must match what's in your `case` block exactly
- The name you pass to `claude --model` must match your **Azure deployment name** exactly

---

## Quick Reference

| Command | What it does |
|---|---|
| `claude-azure` | Switch to Azure (uses your default deployment) |
| `claude-azure <deployment-name>` | Switch to Azure with a specific deployment |
| `claude-default` | Switch back to Anthropic account |
| `claude-mode` | Show current mode |
| `claude --model <deployment-name>` | Launch Claude Code with a specific deployment |
| `/status` | Verify connection inside Claude Code |

---

## Authentication Alternative: Azure CLI / Entra ID (Enterprise)

If your organization uses Entra ID instead of API keys:

```bash
# Log in once via Azure CLI
az login

# Only these two vars are needed — no API key required
export CLAUDE_CODE_USE_FOUNDRY=1
export ANTHROPIC_FOUNDRY_RESOURCE="your-resource-name"
```

Claude Code detects your Azure CLI session automatically. You can simplify your `claude-azure` function by removing all `ANTHROPIC_FOUNDRY_API_KEY` lines if your org uses this approach.
