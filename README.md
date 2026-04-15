# SlopLobster



https://github.com/user-attachments/assets/0bc3c230-282a-4efb-a58e-365e82af710b


## Plan demo video



https://github.com/user-attachments/assets/11ae062c-58f9-4906-9e07-1a31e5d42456



**Local AI coding agent** — runs entirely on your machine. No accounts, no API keys, no cloud.

SlopLobster gives you a Claude Code / Cursor-like agent experience powered by any local LLM via [LM Studio](https://lmstudio.ai/). It reads, writes, and edits your files, runs shell commands, searches the web, automates browsers, and manages git — all through a single self-contained HTML file.


---

## Features

### Core Agent
- **Iterative tool loop** — the LLM reads files, edits code, runs commands, and iterates until the task is done
- **Streaming responses** — see the model think and act in real time
- **Context management** — automatic compaction with progress file persistence when context fills up
- **Smart context injection** — large files are reduced to structure skeletons (function signatures with line numbers), then the model reads specific sections on demand
- **Sub-agents** — spawn focused read-only agents for context gathering, keeping the main context clean
- **Plan mode** — break complex tasks into reviewable steps before executing

### File Operations
- **Read, edit, create, delete, move** files via the File System Access API
- **LCS-based diffing** — precise diffs shown for every edit, with compact/expand toggle
- **Edit approval mode** — optionally require Accept/Reject before applying changes
- **Write verification** — reads files back after writing to confirm integrity
- **Syntax checking** — auto-runs `py_compile` / `node -c` after edits
- **Auto-verify loop** — runs your test command after edits and auto-fixes on failure (configurable retries)

### Shell & Git
- **Full shell access** via the companion server — any OS command works
- **Streaming command output** — see output as it arrives, not after completion
- **Auto-timeout extension** — package installs (`pip`, `npm`, `cargo`, etc.) automatically get 300s
- **Dangerous command detection** — `rm -rf /`, `dd`, `mkfs`, etc. require confirmation
- **Git integration** — status, diff, log, add, commit with dedicated tools

### Web & Browser
- **Web search** via DuckDuckGo with optional auto-fetch of top results
- **URL fetching** with HTML-to-text extraction (strips nav, scripts, styles, extracts links)
- **Browser automation** via Playwright — navigate, click, type, screenshot, evaluate JS, read console errors
- **Reference site directory** — curated list of documentation sites with fetch/search hints injected into the system prompt

### Media
- **Image understanding** — attach images, paste from clipboard, or capture screen regions (vision-capable models)
- **PDF reading** — extracts text from PDFs via pdf.js
- **SVG rendering** — shows rendered preview with code toggle

### UI
- **Right panel** with file preview (syntax highlighted), session changes, terminal log, and git status
- **Resizable right panel** (drag the left edge)
- **File tree** with modified-file indicators and inline actions
- **Conversation management** — save, fork, delete, search, export to Markdown
- **Keyboard shortcuts** — `/` to focus, `Ctrl+B` sidebar, `Ctrl+Shift+N` new chat, `Esc` to stop
- **Dark theme** with amber/gold accent, optimized for long coding sessions

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│  SlopLobster (single HTML file)                 │
│  ┌───────────┐  ┌──────────┐  ┌─────────────┐  │
│  │ Sidebar   │  │  Chat    │  │ Right Panel │  │
│  │ - Convs   │  │  Stream  │  │ - Files     │  │
│  │ - Files   │  │  Tools   │  │ - Changes   │  │
│  │ - Tools   │  │  Diffs   │  │ - Terminal  │  │
│  │ - Status  │  │  Images  │  │ - Git       │  │
│  └─────┬─────┘  └────┬─────┘  └──────┬──────┘  │
│        │             │               │          │
│  ──────┴─────────────┴───────────────┴──────   │
│  File System Access API (Chrome/Edge)           │
└──────────────────┬──────────────────────────────┘
                   │ HTTP
         ┌─────────┴──────────┐
         │  LM Studio (:1234) │  ← LLM inference
         └────────────────────┘
                   │ HTTP
         ┌─────────┴──────────┐
         │ Companion (:8765)  │  ← Shell, search, browser
         │ Python server      │
         └────────────────────┘
```

**Zero backend of its own** — SlopLobster is a pure frontend app. All heavy lifting is done by LM Studio and the companion server.

---

## Quick Start

### 1. Install LM Studio

Download from [lmstudio.ai](https://lmstudio.ai/). Load a model that supports tool/function calling (recommended): Qwen 3.5 or Gemma 4

**For best results**: use a model with ≥32k context and tool-use training. Check the `🔧` tag in the model selector — it indicates the model reports tool-use capability.

### 2. Start the Companion Server

SlopLobster has a **Save & Start** button on the welcome screen that downloads `SlopLobster-companion.py`. Or save it manually:

```bash
# Save the companion script (embedded in the HTML — click "Setup Companion" on welcome)
python SlopLobster-companion.py

# Optional: install Playwright for browser automation
pip install playwright
playwright install chromium

# Or if above doesn't work
python -m playwright install chromium
```

The companion runs on `http://127.0.0.1:8765` and provides:
- Shell command execution (with streaming)
- Web search (DuckDuckGo)
- URL content fetching
- AST-based code skeleton extraction
- Browser automation (if Playwright is installed)

**Without the companion**, SlopLobster falls back to a virtual shell with basic commands (`ls`, `cat`, `grep`, `find`, etc.) — file editing still works, but you won't have git, search, or real shell access.

### 3. Open SlopLobster

Open `index.html` in **Chrome** or **Edge** (required for the File System Access API). That's it — no build step, no `npm install`, no server to start.

### 4. Open a Workspace

Click **Open Workspace** in the sidebar and select your project directory. The agent can now read, create, and edit files in that directory.

---

## Usage

### Basic Workflow

1. **Type a task** in the input box and press Enter
2. **Watch the agent** think, read files, make edits, run commands
3. **Review diffs** that appear after each edit
4. ** intervene** with Esc to stop, or type follow-up instructions

### Attaching Files

- **Drag and drop** files onto the input area
- **Click the 📎 button** to attach files
- **Paste images** from clipboard (Ctrl+V anywhere on the page)
- **Click the 📷 button** to capture a screen region (vision models only)

### Slash Commands

| Command | Description |
|---------|-------------|
| `/help` | Show all commands |
| `/new` | New conversation |
| `/compact` | Manually compact context |
| `/dir` | Open workspace directory |
| `/model` | List available models |
| `/tools` | List available tools |
| `/shell` | Check companion status |
| `/status` | Show full status |
| `/export` | Export conversation as Markdown |

### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Enter` | Send message |
| `Shift+Enter` | New line |
| `Esc` | Stop generation / close modal |
| `Ctrl+Shift+N` | New conversation |
| `Ctrl+B` | Toggle sidebar |
| `Ctrl+K` | Search conversations |
| `Ctrl+,` | Open settings |
| `Ctrl+Shift+E` | Export conversation |
| `Ctrl+Shift+M` | Compact context |
| `Ctrl+Shift+P` | Toggle plan mode |
| `/` | Focus input |
| `↑` / `↓` | Input history |

### Plan Mode

Press `Ctrl+Shift+P` or click the Plan button to enable plan mode. Your next task will be broken into numbered steps that you can review before executing individually or all at once.

### Right Panel

- **Files** — click any file in the sidebar tree to preview it with syntax highlighting
- **Changes** — tracks all file modifications in the session with diffs and revert buttons
- **Term** — streaming log of all shell commands and their output
- **Git** — branch, status, quick stage/log actions

---

## Settings

Click ⚙️ in the header to configure:

| Setting | Default | Description |
|---------|---------|-------------|
| LM Studio URL | `http://localhost:1234` | LM Studio API endpoint |
| Companion URL | `http://127.0.0.1:8765` | Companion server endpoint |
| Temperature | 0.3 | Lower = more deterministic |
| Max Tokens | 60000 | Max response length |
| Max Iterations | 50 | Agent stops after this many tool loops |
| Context Window | 62768 | For context meter & auto-compact |
| Auto-compact | 85% | Compact when context exceeds this |
| Max File Read | 500 KB | Truncate files larger than this |
| Verify Writes | ✓ | Read back files after writing |
| Require Approval | ✗ | Show Accept/Reject before edits |
| Auto-verify Command | (empty) | e.g. `pytest`, `npm test` |
| Max Auto-fix Retries | 3 | Retries after test failure |
| Smart Context | 0 lines | File skeleton for large files (try 150-300) |

### Model Load Settings

When you select a model that isn't loaded, SlopLobster can request LM Studio to load it with optimized settings:

| Setting | Default | Description |
|---------|---------|-------------|
| Context Length Override | 0 (model default) | Lower saves VRAM |
| Flash Attention | ✓ | Lower memory, faster generation |
| KV Cache → GPU | ✓ | Faster than RAM offload |
| MoE Experts | 0 (all) | Fewer experts = less VRAM for Mixtral/Qwen-MoE |
| Eval Batch Size | 0 (default) | Higher = faster prompt processing |

### System Prompt

Fully customizable with save/load presets. The default prompt includes:
- Dynamic capability checklist (updates based on what's connected)
- Workspace awareness (prevents nested directory creation)
- Reference site directory (MDN, Python docs, React docs, etc.)
- File reading strategy guidelines
- Web search strategy with JS-rendered page workarounds
- Error recovery patterns
- Security boundary documentation

---

## Companion Server

The companion is a single Python file with no dependencies beyond the standard library. Optional dependency: `playwright` for browser automation.

```
SlopLobster-companion.py
├── Shell execution (streaming NDJSON)
├── Web search (DuckDuckGo HTML parsing)
├── URL fetching (HTML→text extraction)
├── AST signature extraction (Python, JS, generic)
├── Browser automation (Playwright wrapper)
└── Status/health endpoint
```

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/status` | Health check, platform info, Python env |
| POST | `/execute` | Run shell command (NDJSON stream) |
| POST | `/search` | DuckDuckGo search with optional fetch |
| POST | `/fetch` | Fetch and extract URL content |
| POST | `/ast_signatures` | Extract code skeleton from source |
| POST | `/browser_*` | Playwright browser automation |

### Security

- Binds to `127.0.0.1` only — no external access
- No authentication needed for localhost
- File operations are sandboxed by the browser's File System Access API (not the companion)
- SlopLobster validates all paths for traversal attacks before sending to the companion

---

## How It Works

### Tool Loop

SlopLobster uses the OpenAI-compatible tool/function calling API:

1. User sends a message
2. Message + system prompt + conversation history → LM Studio
3. LLM responds with either text or tool calls
4. Tool calls are executed locally (file I/O) or via companion (shell, search)
5. Tool results are appended to history and sent back to the LLM
6. Repeat until the LLM responds with text only

### Context Management

- **Token estimation**: ~3.8 characters per token
- **Context meter**: shows usage as percentage of configured window
- **Auto-compact**: when usage exceeds threshold, the conversation is summarized by the LLM, and a detailed `.sloplobster-progress.md` file is saved to the workspace
- **Post-compact**: the agent reads the progress file to recover full context and continues working
- **Smart context**: files over a configurable line threshold are reduced to their AST skeleton (function signatures with line numbers), saving thousands of tokens on large files

### Diff Engine

Uses a full **Longest Common Subsequence (LCS)** algorithm for precise diffs. For files where `m × n > 50,000`, falls back to a simple line-by-line comparison with truncation. Diffs are displayed in compact mode by default (changed lines + 1 line of context) with an expand toggle.

### Image Pipeline

1. Images are captured (screen share API or html2canvas fallback)
2. Compressed to JPEG at 1280px max dimension, 80% quality
3. Sent as `image_url` content parts in the API request
4. The LLM sees the actual pixels (if vision-capable)
5. Previews are shown in both the chat and the right panel

---

## Browser Support

| Browser | Status | Notes |
|---------|--------|-------|
| Chrome 86+ | ✅ Full | File System Access API, screen capture |
| Edge 86+ | ✅ Full | Same as Chrome |
| Firefox | ⚠️ Partial | No File System Access API — use companion for all file ops |
| Safari | ❌ No | No File System Access API or screen capture |

---

## Troubleshooting

**"No models" in selector**
- Make sure LM Studio is running and a model is loaded
- Check the API URL in Settings matches LM Studio's port (default 1234)

**Model selector shows different model than what's loaded**
- LM Studio's `/v1/models` endpoint lists all downloaded models, not just the loaded one
- Look for the `✓ LOADED` tag — only that model is actually active
- Use the 📥 button to load the selected model

**"Virtual shell only"**
- The companion server isn't running — start it with `python SlopLobster-companion.py`
- Check the companion URL in Settings matches the server's port

**Context filling up too fast**
- Enable Smart Context (Settings → Smart Context → 150-300 lines)
- Reduce Max File Read to truncate large files earlier
- Lower the Context Window setting to trigger auto-compact sooner
- Use plan mode to work in smaller steps

**Agent keeps making malformed tool calls**
- The model's context is likely full or nearly full — compact
- Large file writes get truncated by the API — use `edit_file` with small blocks instead of `write_file`
- Try a model with more context length

**"JavaScript-rendered page" when fetching URLs**
- `fetch_url` uses `urllib` which cannot execute JavaScript
- Use `web_search` with `site:domain.com <topic>` instead — DuckDuckGo's crawler executes JS
- Or use the browser automation tools to load the page in a real browser

**Companion connection drops mid-conversation**
- SlopLobster automatically falls back to the virtual shell
- Tool calls that require the companion will fail gracefully
- Reconnect by clicking the 🔄 button or restarting the companion

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| UI | Vanilla HTML/CSS/JS, Tailwind CSS |
| Markdown | marked.js |
| Syntax Highlighting | highlight.js |
| Sanitization | DOMPurify |
| PDF | pdf.js |
| Screenshots | html2canvas + Screen Capture API |
| LLM | LM Studio (OpenAI-compatible API) |
| Shell/Search | Python stdlib + Playwright |
| Storage | localStorage (conversations, settings) |

---
