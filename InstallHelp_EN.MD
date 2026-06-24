# Paperclip — Installation and Control Guide with Claude Code

## What is Paperclip

Paperclip is a **self-hosted multi-agent AI platform**: a system where AI agents (Claude, Codex, local models via Ollama/LM Studio, etc.) work together within a virtual organizational structure — with companies, projects, tasks, approvals, and workflows.

Think of it as a "virtual office" where:
- the **board** (you, or Claude acting on your behalf) assigns work
- the **agents** execute it autonomously
- everything is tracked, approvable, and versioned

Paperclip runs entirely locally on your Mac — no data leaves the machine.

---

## Installation

### Prerequisites

- Node.js 18+ (check with `node --version`)
- npm or npx available in the terminal

### Installation and first run

```bash
# Run onboarding (installs and configures everything automatically)
npx paperclipai onboard --yes
```

The `--yes` flag accepts all defaults: `local_trusted` mode, accessible from `localhost` only, no authentication required.

Once done, start the server:

```bash
npx paperclipai run
```

The server runs at **`http://localhost:3100`** — the UI is accessible from a browser, but in this guide we control it from Claude Code.

### Deployment modes

| Mode | Description | Auth required |
|------|-------------|---------------|
| `local_trusted` | Default. Localhost only, no auth | No |
| `lan` | Accessible on the local network | Yes |
| `tailnet` | Accessible via Tailscale | Yes |

This guide uses `local_trusted`.

---

## Controlling Paperclip with Claude Code

There are three ways to have Claude Code manage Paperclip. Each has a different role, different limitations, and a different architecture. Understanding the difference is essential.

---

### Mode 1 — CLI (`npx paperclipai`)

**How it works:** Claude runs terminal commands via the `Bash` tool.

**Setup:** none. In `local_trusted` mode you just need the server running.

```bash
# Set the context once
npx paperclipai context set --api-url http://localhost:3100

# Example commands
npx paperclipai company list
npx paperclipai agent list --company-id <id>
npx paperclipai issue create --company-id <id> --title "Implement feature X"
npx paperclipai invite create --company-id <id> --payload-json '{"email":"...","role":"member"}'
```

**What it can do:** manage companies, agents, issues, invites, approvals, skills, backups, worktrees.

**Limitations:**
- Some commands are missing compared to the REST API (e.g. `company create` is not in the CLI)
- Requires the server to be running
- Text output — less structured to parse than raw JSON

**When to use it:** day-to-day administrative operations, simple automations.

---

### Mode 2 — Direct REST API (`curl` / fetch)

**How it works:** Claude calls the Paperclip server's HTTP API directly via `curl`.

**Setup:** none in `local_trusted`. No authentication headers needed.

```bash
# Create a company
curl -s -X POST http://localhost:3100/api/companies \
  -H "Content-Type: application/json" \
  -d '{"name":"My Org","shortname":"org"}'

# Create an agent
curl -s -X POST http://localhost:3100/api/companies/<id>/agents \
  -H "Content-Type: application/json" \
  -d '{"name":"builder","adapterType":"codex_local"}'

# Create a task
curl -s -X POST http://localhost:3100/api/companies/<id>/issues \
  -H "Content-Type: application/json" \
  -d '{"title":"Analyse the code","description":"..."}'
```

**What it can do:** everything — it is the full interface. 41 endpoints covering companies, agents, issues, documents, approvals, costs, activity, plugins, secrets, and routines.

**Limitations:**
- More verbose to write than the CLI
- Requires managing IDs manually
- In authenticated mode (`lan`/`tailnet`) a Bearer token is required

**When to use it:** operations not available in the CLI, complex automations, integration with other systems.

---

### Mode 3 — MCP Server (`@paperclipai/mcp-server`)

**How it works:** a separate process that exposes Paperclip tools as native Claude Code tools, without going through the terminal.

**Setup:**
```json
// ~/.claude/mcp.json
{
  "mcpServers": {
    "paperclip": {
      "command": "npx",
      "args": ["-y", "@paperclipai/mcp-server"],
      "env": {
        "PAPERCLIP_API_URL": "http://localhost:3100",
        "PAPERCLIP_API_KEY": "<agent-api-key>",
        "PAPERCLIP_COMPANY_ID": "<default-company-id>"
      }
    }
  }
}
```

**Why an API key is required (and why it belongs to an agent):**

Paperclip does not issue generic API keys — it only issues them for "agent" entities. This is because the MCP server was designed for a specific scenario: an AI worker agent that uses the server to interact with its own tasks (check out, update status, add comments, report completion). The agent authenticates with its key to say "this is me, I finished this task".

When you use the MCP server to *command* the company instead of *working inside it*, you are bending the mechanism backwards: you create a "dummy" agent (e.g. `claude-mcp`) just to get the key, but that agent never executes tasks — it is only an authentication identity.

```
Intended design use:
  Paperclip board → assigns task → Agent uses MCP server → executes task → updates Paperclip

Use as orchestrator (not ideal):
  Claude uses MCP server → commands company, agents, issues → ...but it is the wrong tool for this role
```

**What it can do:** 23 read tools + 16 write tools — issues, agents, approvals, comments, documents, workspaces.

**Limitations:**
- Designed for workers, not admins
- Requires a registered agent in the company just to have the key
- `PAPERCLIP_COMPANY_ID` is optional (default); tools accept `companyId` per request
- Requires restarting Claude Code to load new servers

**When to use it:** when you want Claude to operate *as* a worker agent inside a Paperclip workflow — not to administer the platform.

---

## Comparison summary

| | CLI | REST API | MCP Server |
|--|-----|----------|------------|
| Setup | None | None | Agent + API key + mcp.json |
| Auth (local_trusted) | Not needed | Not needed | API key required |
| Functional coverage | ~80% | 100% | ~60% (worker-focused) |
| Ideal role | Daily admin | Full automation | Worker agent |
| How Claude uses it | `Bash` tool | `Bash` tool (curl) | Native MCP tools |

**Recommendation:** to drive Paperclip with Claude Code as orchestrator/admin, use **CLI + REST API**. The MCP server is only useful if you want Claude to behave as a worker agent inside a Paperclip workflow.

---

## Practical example — Autonomous software development team

This example shows how to create a Paperclip company structured as a software development team, with 8 specialised agents coordinated by a Tech Lead.

> The attached `software_development_company_claudecode_fixed.zip` file contains the exportable dump of this company, importable with `npx paperclipai company import`.

### Team architecture

The principle is to have a **single point of contact** (Tech Lead) to give natural-language instructions to. The Tech Lead breaks down the work and coordinates the specialists.

| Agent | Role |
|-------|------|
| **tech-lead** | Single point of contact. Receives briefs, breaks them into technical tasks, coordinates all other agents, reports status |
| **architect** | Designs the software structure — components, interfaces, technology choices |
| **backend-developer** | Implements APIs, business logic, databases |
| **frontend-developer** | Implements UI and integration with backend APIs |
| **test-engineer** | Writes tests (unit, integration, e2e) in parallel with development |
| **security-reviewer** | Analyses code for vulnerabilities before merge — can block with explicit approval |
| **devops** | CI/CD, deployment, infrastructure, monitoring |
| **tech-writer** | API documentation, README, changelog |

### Creating the company via REST API

The `company create` command is not available in the CLI, so the REST API is used directly:

```bash
# 1. Create the company
curl -s -X POST http://localhost:3100/api/companies \
  -H "Content-Type: application/json" \
  -d '{"name":"AutoDev Team","shortname":"autodev"}'
# Save the returned id → COMPANY_ID

# 2. Create the agents (repeat for each one)
curl -s -X POST http://localhost:3100/api/companies/$COMPANY_ID/agents \
  -H "Content-Type: application/json" \
  -d '{"name":"tech-lead","title":"Tech Lead","adapterType":"codex_local"}'

curl -s -X POST http://localhost:3100/api/companies/$COMPANY_ID/agents \
  -H "Content-Type: application/json" \
  -d '{"name":"architect","title":"Software Architect","adapterType":"codex_local"}'

# ... (repeat for backend-developer, frontend-developer,
#      test-engineer, security-reviewer, devops, tech-writer)
```

### Assigning a task to the Tech Lead

Once the structure is created, you only talk to the Tech Lead:

```bash
TECHLEAD_ID="<tech-lead-agent-id>"

curl -s -X POST http://localhost:3100/api/companies/$COMPANY_ID/issues \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Build a system process monitoring web app",
    "description": "Create a cross-platform web application (Windows, macOS, Linux) that displays active processes on the device in real time.\n\nRequirements:\n- Python backend with Flask\n- Process reading via psutil (cross-platform)\n- Frontend with auto-refresh every 2-3 seconds\n- Table with PID, name, CPU%, memory%, status, user\n- Column sorting, filter by name\n- Total CPU and RAM system indicators\n\nCoordinate the team specialists for architecture, development, testing, security review and documentation.",
    "assigneeAgentId": "$TECHLEAD_ID",
    "priority": "high"
  }'
```

> **Note:** use `assigneeAgentId` (not `assigneeId`) when assigning to an agent. `assigneeId` is for human users and will leave the issue unassigned — agents will not pick it up.

### Configuring local models after import

> **Note:** due to a bug in Paperclip's import process, when importing a company whose agents were configured with an adapter such as OpenCode and a local model, they may be reset to Claude Code regardless of how they were configured in the file. After every import you need to reassign the adapter manually.

The following example uses **gemma4:e4b via Ollama**.

> **Important — context length:** before assigning tasks to agents, make sure the model's context window is set to 64k or more (tested: 64k works; smaller values may not — especially with models that have fewer billion parameters). A short context causes `step failed` even on runs reported as "succeeded". See the [**Context length — critical setting**](#context-length--critical-setting) section for both Ollama and LM Studio.

#### Prerequisites

1. Ollama running with the model downloaded:
   ```bash
   ollama pull gemma4:e4b
   ollama serve   # already running if the icon is in the menu bar on macOS
   ```

2. If Ollama models do not appear in OpenCode (see the dedicated section below), create `~/.config/opencode/config.json` with the model (whether Ollama needs to be declared explicitly depends on the OpenCode version):
   ```json
   {
     "$schema": "https://opencode.ai/config.json",
     "provider": {
       "ollama": {
         "npm": "@ai-sdk/openai-compatible",
         "name": "Ollama (local)",
         "options": { "baseURL": "http://127.0.0.1:11434/v1" },
         "models": {
           "gemma4:e4b": { "name": "Gemma 4 E4B" }
         }
       }
     }
   }
   ```

#### Via UI (for each agent)

1. Open Paperclip → Company → agent → **Settings**
2. **Adapter** section → select **OpenCode**
3. In the **Model** selector, choose the model listed under `ollama` (e.g. `ollama / Gemma 4 E4B`)
4. If a **secondary model** (fallback) is present, disable it or leave it empty — the default model may not be available and can cause silent errors
5. Save and repeat for all 8 agents

#### Via REST API (all agents at once)

First retrieve the agent IDs:

```bash
COMPANY_ID="<company-id>"

curl -s http://localhost:3100/api/companies/$COMPANY_ID/agents \
  | python3 -c "
import sys, json
agents = json.load(sys.stdin)
for a in agents:
    print(a['id'], a['name'])
"
```

Then update a single agent (replace `<AGENT_ID>`):

```bash
curl -s -X PATCH http://localhost:3100/api/companies/$COMPANY_ID/agents/<AGENT_ID> \
  -H "Content-Type: application/json" \
  -d '{"adapterType":"opencode_local","adapterConfig":{"model":"ollama/gemma4:e4b"}}'
```

Or update all of them in a loop:

```bash
COMPANY_ID="<company-id>"

# Retrieve all IDs
AGENT_IDS=$(curl -s http://localhost:3100/api/companies/$COMPANY_ID/agents \
  | python3 -c "import sys,json; [print(a['id']) for a in json.load(sys.stdin)]")

# Update all sequentially
for ID in $AGENT_IDS; do
  curl -s -X PATCH http://localhost:3100/api/companies/$COMPANY_ID/agents/$ID \
    -H "Content-Type: application/json" \
    -d '{"adapterType":"opencode_local","adapterConfig":{"model":"ollama/gemma4:e4b"}}' \
    | python3 -c "import sys,json; a=json.load(sys.stdin); print(a.get('name'), '→', a.get('adapterType'))"
done
```

To use a different model (e.g. LM Studio), replace `ollama/gemma4:e4b` with `lmstudio/<model-id>` and make sure the `lmstudio` section is present in OpenCode's `config.json` (see the LM Studio section below).

---

### Why this approach works

**Real parallelism:** while the Backend Developer writes the Flask APIs, the Frontend Developer works on the interface and the Test Engineer prepares the tests. No one waits.

**Quality gate:** the Security Reviewer must explicitly approve before code is merged. If they find a problem (e.g. exposure of sensitive system data), they block the flow.

**Scalability:** the same structure handles a small project (the web app above) and a large one. Add tasks to the Tech Lead, they route them.

**Tunable autonomy:** you can let the team work completely on its own on well-defined tasks, or set human approvals at critical checkpoints (architecture, production deploy).

---

## Recommended models for Apple Silicon

For Paperclip agents that write code autonomously, the best family available on Ollama is **Qwen2.5-Coder** (Alibaba): it follows complex instructions, produces structured code, and handles long contexts.

### MacBook Pro M5 — 24 GB RAM

```bash
ollama pull qwen2.5-coder:14b
```

~8.5 GB in memory — runs comfortably, excellent speed on M5. This is the sweet spot for 24 GB: capable enough for autonomous agents, light enough not to stress the RAM.

### Mac Studio M2/M4 Max — 64–96 GB RAM

```bash
ollama pull qwen2.5-coder:32b
```

~19 GB in memory — very high quality, suitable for complex multi-file tasks.

### Mac Studio M2/M4 Ultra — 128–192 GB RAM

```bash
ollama pull qwen2.5-coder:72b
```

~43 GB in memory — excellent quality, close to cloud models for coding tasks.

---

## Local models — Ollama and LM Studio

Paperclip agents are not tied to Claude. You can run them on local models via **Ollama** or **LM Studio**, using **OpenCode** as the adapter. This section describes how to configure both.

The project with agents configured with OpenCode is available here: `software_development_company_claudecode_fixed_local.zip`

---

### Adapter: OpenCode

In Paperclip, when creating or editing an agent, you can choose `opencode_local` as the `adapterType`. OpenCode is the process that bridges Paperclip and AI models — both cloud and local. It natively supports Ollama and, with manual configuration, LM Studio.

---

### Ollama (recommended — simpler setup than LM Studio)

#### Installation and models

```bash
# Install Ollama from https://ollama.com
# Start the server (starts automatically at login on macOS)
ollama serve

# Download models (examples)
ollama pull llama3.2          # 3B parameters — lightweight, low accuracy
ollama pull llama3.2:8b       # 8B — good balance
ollama pull gemma4:26b        # 26B — high quality, requires more RAM
ollama pull qwen3.5           # excellent for coding

# Verify a model responds
ollama run llama3.2 "hello"
```

#### Context length — critical setting

Paperclip sends agents a very long system prompt (protocol instructions, company state, issue context, etc.). If the model's context window is too short, the prompt arrives truncated and the model does not understand what to do — it returns `step failed` or empty responses even though the run is reported as "succeeded".

**Set the context length to 64k or more** in the Ollama app:

1. Open the Ollama app (menu bar icon) → **Settings**
2. Find the **Context Length** field (or `num_ctx`) and set it to `65536` (64k)
3. Restart Ollama

Alternatively, set it per model by creating a Modelfile:

```bash
# Create a Modelfile extending an existing model with 64k context
cat > /tmp/Modelfile <<'EOF'
FROM gemma4:e4b
PARAMETER num_ctx 65536
EOF
ollama create gemma4-e4b-64k -f /tmp/Modelfile
```

A 4k context (default for some models) causes `step failed`. With 64k, Paperclip agents work correctly.

**In LM Studio** the context length is set per model at load time: in the **Developer → Local Server** tab, when you click **Load model**, there is an **arrow (▶)** to the right of the model name — clicking it opens a panel where you can specify the context window size before starting the server.

#### Ollama endpoint

Ollama exposes an OpenAI-compatible server at:

```
http://127.0.0.1:11434
```

You can list available models with:

```bash
curl http://127.0.0.1:11434/v1/models
```

#### Configuration in Paperclip

1. Edit the agent → `adapterType: opencode_local`
2. Select the model from the dropdown (e.g. `ollama/llama3.2`)

In some versions of OpenCode, Ollama is discovered automatically with no configuration. However, if Ollama models do not appear in the model selector (Ctrl+P → "Change model"), you need to declare them explicitly in `~/.config/opencode/config.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": { "baseURL": "http://127.0.0.1:11434/v1" },
      "models": {
        "llama3.2:3b": { "name": "Llama 3.2 3B" },
        "qwen2.5-coder:14b": { "name": "Qwen 2.5 Coder 14B" }
      }
    }
  }
}
```

Replace the models with those returned by `curl http://127.0.0.1:11434/v1/models`.

#### Testing models in OpenCode

To verify which models OpenCode sees before assigning them to an agent:

```bash
opencode
# then: Ctrl+P → "Change model"
# the list of all available models will appear
```

---

### LM Studio

#### Enabling the local server

1. Open LM Studio
2. Right sidebar → **Developer** → **Local Server**
3. Set **Status: Enabled**
4. Click **Load model** and select the desired model

#### Finding the model ID

LM Studio exposes its server at `http://127.0.0.1:1234`. To find the exact ID of the loaded model:

```bash
curl http://127.0.0.1:1234/v1/models
```

The JSON response contains the `id` field — for example `gemma-4-e4b-it`. This is the name you will use in the configuration.

#### Configuring OpenCode for LM Studio

Unlike Ollama, LM Studio **is not discovered automatically** by OpenCode. You must register it manually in the OpenCode configuration file:

**`~/.config/opencode/config.json`**

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "lmstudio": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "LM Studio (local)",
      "options": { "baseURL": "http://127.0.0.1:1234/v1" },
      "models": {
        "gemma-4-e4b": { "name": "Gemma 4 E4B (local)" }
      }
    }
  },
  "model": "lmstudio/gemma-4-e4b"
}
```

Replace `gemma-4-e4b` with the `id` obtained from the curl above.

#### Configuration in Paperclip

After saving `config.json`, in Paperclip the agent with `opencode_local` adapter will show the `lmstudio/<model-id>` model in the dropdown.

---

### Generic adapters or non-default ports: When `OPENAI_API_KEY` and `OPENAI_BASE_URL` are needed

With the **OpenCode** adapter these environment variables **are not necessary** — neither for Ollama nor for LM Studio. OpenCode manages the connection internally.

They are needed in these cases:

| Case | Required variables |
|------|--------------------|
| Generic OpenAI-compatible adapter (not OpenCode) | `OPENAI_BASE_URL=http://127.0.0.1:11434/v1` (Ollama) or `http://127.0.0.1:1234/v1` (LM Studio) + `OPENAI_API_KEY=ollama` (dummy value but required) |
| Ollama on a non-standard port or remote host | `OPENAI_BASE_URL=http://<host>:<port>/v1` |
| Force OpenCode to use a specific endpoint, overriding auto-discovery | explicit `OPENAI_BASE_URL` |

---

### Zhipu AI — GLM 5.2 (cloud model)

OpenCode supports any provider with an OpenAI-compatible API, including **Zhipu AI** (bigmodel.cn). This lets you use GLM models directly from the cloud, with no local hardware required.

> **Prerequisites:** an account on [open.bigmodel.cn](https://open.bigmodel.cn) with a generated API key. See the [Creating a Zhipu AI account and getting an API key](#creating-a-zhipu-ai-account-and-getting-an-api-key) section below.

#### 1. Configure OpenCode for Zhipu AI

Open (or create) `~/.config/opencode/config.json` and add the `zhipuai` provider inside the `provider` section:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "zhipuai": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Zhipu AI (cloud)",
      "options": {
        "baseURL": "https://open.bigmodel.cn/api/paas/v4/",
        "apiKey": "YOUR_API_KEY_HERE"
      },
      "models": {
        "glm-4-flash": { "name": "GLM 5.2" }
      }
    }
  }
}
```

> **Note on model ID:** the key inside `models` (e.g. `glm-4-flash`) must match the **exact API identifier** of the model — verify it in the bigmodel.cn console under **Available Models** or **API → Documentation**. The `name` value is just a local label shown in OpenCode. For GLM 5.2, the API ID may be something like `glm-5.2`, `glm-5-flash`, or similar — always check in the console.

If you already have other providers in the file (e.g. Ollama), add `zhipuai` alongside them inside the same `provider` section:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": { ... },
    "zhipuai": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Zhipu AI (cloud)",
      "options": {
        "baseURL": "https://open.bigmodel.cn/api/paas/v4/",
        "apiKey": "YOUR_API_KEY_HERE"
      },
      "models": {
        "glm-4-flash": { "name": "GLM 5.2" }
      }
    }
  }
}
```

After saving, verify that OpenCode sees the new model:

```bash
opencode
# Ctrl+P → "Change model" — "Zhipu AI (cloud) / GLM 5.2" should appear
```

#### 2. Configure Paperclip agents to use GLM 5.2

> Practical example: import `software_development_company_claudecode_fixed.zip` and after import configure the agents to use GLM 5.2 via the REST API, without modifying the ZIP file.

**Via UI (for each agent):**

1. Open Paperclip → Company → agent → **Settings**
2. **Adapter** section → select **OpenCode**
3. In the **Model** selector, choose the model listed under `zhipuai` (e.g. `zhipuai / GLM 5.2`)
4. If a **secondary model** (fallback) is shown, disable it or leave it empty
5. Save and repeat for all agents

**Via REST API (all agents at once):**

```bash
COMPANY_ID="<company-id>"
MODEL="zhipuai/glm-4-flash"   # replace with the exact API model ID

# Retrieve all agent IDs
AGENT_IDS=$(curl -s http://localhost:3100/api/companies/$COMPANY_ID/agents \
  | python3 -c "import sys,json; [print(a['id']) for a in json.load(sys.stdin)]")

# Update all agents
for ID in $AGENT_IDS; do
  curl -s -X PATCH http://localhost:3100/api/companies/$COMPANY_ID/agents/$ID \
    -H "Content-Type: application/json" \
    -d "{\"adapterType\":\"opencode_local\",\"adapterConfig\":{\"model\":\"$MODEL\"}}" \
    | python3 -c "import sys,json; a=json.load(sys.stdin); print(a.get('name'), '→', a.get('adapterType'), a.get('adapterConfig',{}).get('model',''))"
done
```

#### Context length

Zhipu AI cloud models support large context windows (128k or more) by default — no additional configuration is needed. The `step failed` issue caused by a short context is specific to local models (Ollama/LM Studio with a 4k default). See the [Context length — critical setting](#context-length--critical-setting) section for details on local models.

---

### Creating a Zhipu AI account and getting an API key

1. Go to [open.bigmodel.cn](https://open.bigmodel.cn) and click **Register**
2. Sign up with an email address (Google login is also available)
3. In the console, go to **API Keys** (left sidebar)
4. Click **+ Create API Key**, give it a name, and copy the generated key — it will not be shown again
5. Paste the key into the `apiKey` field in OpenCode's `config.json` as shown above

> **Free tier:** Zhipu AI provides a free credit on registration, sufficient for testing and development. Check the dashboard for your account's limits.

> **Security:** never commit the API key to code or versioned files. Keep it only in `~/.config/opencode/config.json`, which is outside any repository.

---

### Which model to choose

| Model | Parameters | Best for |
|-------|-----------|----------|
| `llama3.2:3b` | 3B | Quick tests — not reliable for autonomous coding |
| `llama3.2:8b` | 8B | Simple tasks, basic instructions |
| `qwen3.5` | variable | Excellent for coding, follows instructions well |
| `gemma4:26b` | 26B | High quality, requires ~20 GB RAM |

**Rule of thumb:** agents that write code autonomously in Paperclip need at least 7–14B parameters and preferably a coding-optimised model (Qwen, DeepSeek Coder, CodeLlama). 3B models tend not to follow instructions precisely and fail Paperclip's verification probe.
