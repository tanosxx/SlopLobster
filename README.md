# SlopLobster

> **⚠️ Heads up — this is a vibe-coded personal project, not production software.** It gives an AI agent direct shell access to your machine. That's inherently dangerous. I built it because existing agents were too heavy, too cloud-dependent, or too locked down for my workflow. It works well for me. It might delete your files, run bad commands, or catch fire. Read the security section below, use it in a sandbox or VM if you're smart, and don't blame me.

---

SlopLobster is a local AI coding agent that runs entirely in your browser. It connects to [LM Studio](https://lmstudio.ai/) for LLM inference and an optional Python companion server for real shell access, git operations, and web search. No cloud APIs, no accounts, no telemetry — everything stays on your machine.

```
┌─────────────────────────────────────────────┐
│              Your Browser                    │
│  ┌─────────────────────────────────────┐    │
│  │         SlopLobster UI              │    │
│  │   (single HTML file, no build)      │    │
│  └──────┬──────────┬──────────┬────────┘    │
│         │          │          │              │
│    File System   Tool       Screenshot      │
│    Access API   Dispatch    (html2canvas)   │
│         │          │                         │
├─────────┼──────────┼─────────────────────────┤
│         │          │       Your Machine      │
│  ┌──────▼──────┐   │   ┌──────────────────┐ │
│  │  Workspace   │   │   │  LM Studio       │ │
│  │  (direct     │   │   │  localhost:1234  │ │
│  │   read/write)│   │   │  (OpenAI-compat) │ │
│  └─────────────┘   │   └──────────────────┘ │
│                    │   ┌──────────────────┐ │
│                    └──▶│  Companion       │ │
│                        │  localhost:8765  │ │
│                        │  (shell/git/web) │ │
│                        └──────────────────┘ │
└─────────────────────────────────────────────┘
```

## Features

**Agent Loop** — The model thinks, uses tools, observes results, and iterates autonomously until the task is done. No manual step-by-step prompting needed.

**File Operations** — Read, write, and edit files directly through the browser's File System Access API. The edit tool uses exact search/replace blocks with whitespace-tolerant fallback matching and LCS-based diff generation for review.

**Shell Execution** — Two tiers:
- *Virtual shell* (always available): `ls`, `cat`, `head`, `tail`, `grep`, `find`, `tree`, `wc`, `file`, `diff`, `mkdir`, `touch` — operates directly on the workspace without any server
- *Full shell* (companion): any OS command, with auto-extended timeouts for package installs (pip, npm, cargo, etc.)

**Web Search** — DuckDuckGo search via the companion, with optional auto-fetch of top result content for deeper reading. Results are cached for 5 minutes.

**Git Integration** — `git_status`, `git_diff`, `git_log`, `git_add`, `git_commit` tools when the companion is running.

**Image Understanding** — If your model supports vision (LLaVA, Qwen-VL, Pixtral, etc.), the agent can view image files and take screenshots of the current UI state to inspect error messages, tool output, or code changes visually.

**Context Management** — Tracks estimated token usage, auto-compacts when the context window fills up, and saves a detailed progress file (`.sloplobster-progress.md`) to the workspace so the agent can recover full context after compaction.

**Conversation Management** — Save, search, fork, and delete conversations. All stored in localStorage. Export any conversation as Markdown.

## Quick Start

### 1. LM Studio

[Download LM Studio](https://lmstudio.ai/), load a model that supports tool/function calling (recommended: Qwen 2.5 Coder 32B, Llama 3.1 70B, or similar), and start the local server on `localhost:1234`.

### 2. SlopLobster

Open `index.html` in Chrome or Edge. That's it — no install, no build step. The app will auto-detect LM Studio on startup.

### 3. Companion Server (optional but recommended)

Click **Save** in the sidebar to download `SlopLobster-companion.py`, then run it:

```bash
python SlopLobster-companion.py
```

Click **Connect** in the sidebar. This enables real shell execution, git, and web search. Without it, you get a sandboxed virtual shell with basic file commands only.

## Requirements

| Component | Requirement |
|-----------|-------------|
| Browser | Chrome or Edge (File System Access API) |
| LLM | LM Studio with a function-calling model |
| Companion | Python 3.7+ (standard library only, no pip installs) |
| Vision | A vision-capable model in LM Studio (optional) |

## Security

This is the part you should actually read.

**What's dangerous:**
- The companion server executes arbitrary shell commands on your machine. If the model decides to run `rm -rf /`, that's a real problem.
- The agent can write to any file in your workspace directory.
- The agent can delete files in your workspace.
- LLMs hallucinate. A model might misinterpret your intent and do something destructive.

**What's protected:**
- **Path traversal blocking** — All file paths are sanitized. `../` segments are rejected. The agent cannot escape your workspace directory through path manipulation.
- **Dangerous command detection** — Patterns like `rm -rf /`, `dd of=/dev/`, `mkfs`, `chmod 777 /`, `git push --force`, `shutdown`, and others trigger a confirmation dialog before execution. This is regex-based and not exhaustive — it's a speed bump, not a guarantee.
- **Localhost only** — The companion server binds to `127.0.0.1` exclusively. Nothing is exposed to the network.
- **No data leaves your machine** — No telemetry, no analytics, no cloud calls. The only network traffic is to `localhost:1234` (LM Studio) and `localhost:8765` (companion).
- **Command timeouts** — Shell commands time out after a configurable duration (default 60s, auto-extended to 300s for package installs). Processes are killed on timeout.
- **Write verification** — After writing a file, the content is read back and compared to confirm the write was successful.
- **No `sudo` passthrough** — The companion doesn't elevate privileges. If you need sudo, that's on you.

**What's NOT protected:**
- The dangerous command detection is regex-based and can be bypassed with creative command construction
- If you give the agent a workspace that contains sensitive files (SSH keys, credentials, `~/.bashrc` symlinks), it can read them
- The model could construct a command that passes the regex filters but is still destructive
- There's no sandboxing beyond the workspace directory boundary

**Recommendations:**
- Use a dedicated project directory, not your home folder
- Don't point it at anything you can't afford to lose
- Consider running in a VM or container for untrusted models
- Keep an eye on what it's doing — tool calls are displayed in real time
- Use a smaller, less capable model if you're worried about agent autonomy

## How It Works

The agent loop follows a think → act → observe cycle:

1. **Think** — The model uses the `think` tool to reason about the task (visible to you as a collapsible block)
2. **Act** — The model calls tools (read files, edit code, run commands, search the web). Multiple independent tools can run in parallel.
3. **Observe** — Tool results are fed back to the model as `tool` messages
4. **Repeat** — The cycle continues until the model stops calling tools and responds with text

The system prompt is dynamically constructed based on current state:
- What tools are available (depends on companion connection)
- Whether the model supports vision
- What OS/shell the companion is running on
- Project type detection (Python, Node, Rust, Go)
- Current context usage percentage

## Configuration

Open Settings (gear icon) to configure:

| Setting | Default | Description |
|---------|---------|-------------|
| API URL | `http://localhost:1234` | LM Studio endpoint |
| Companion URL | `http://127.0.0.1:8765` | Companion server endpoint |
| Temperature | 0.3 | Lower = more deterministic |
| Max Tokens | 60000 | Per-response token limit |
| Max Iterations | 50 | Agent loop limit before auto-stop |
| Max File Read | 500 KB | Truncation threshold for file reads |
| Command Timeout | 60s | Shell command timeout |
| Context Window | 62768 | For context meter estimation |
| Auto-compact | 85% | Context usage % that triggers compaction |
| Verify Writes | on | Read-back verification after file writes |

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Enter` | Send message |
| `Shift+Enter` | New line |
| `Esc` | Stop generation / close modal |
| `Ctrl+Shift+N` | New conversation |
| `Ctrl+B` | Toggle sidebar |
| `Ctrl+K` | Search conversations |
| `/` | Focus input |
| `Ctrl+,` | Open settings |
| `Ctrl+Shift+E` | Export conversation |
| `Ctrl+Shift+M` | Compact context |

## Slash Commands

| Command | Description |
|---------|-------------|
| `/help` | Show available commands |
| `/new` | New conversation |
| `/compact` | Manually compact context |
| `/dir` | Open workspace directory |
| `/model` | List available models |
| `/tools` | List available tools |
| `/shell` | Check companion status |
| `/status` | Show full status |
| `/export` | Export as Markdown |

## Limitations

- **Single HTML file** — The entire app is one file. It's ~70KB of HTML/CSS/JS. This means no bundler, no hot reload, no component framework. It also means you can just open it.
- **No streaming tool calls** — Tool call arguments are buffered until complete. You see the thinking in real time, but tool execution starts after the full response chunk.
- **localStorage only** — Conversations are stored in localStorage (typically 5-10MB limit). Large conversations will hit the quota. The app handles this by removing old conversations.
- **No multi-file editing atomically** — Each `edit_file` call is a separate write. Parallel edits to the same file would conflict.
- **Virtual shell is limited** — Without the companion, you get basic file commands but no real OS shell, no git, no web search.
- **DuckDuckGo scraping** — Web search works by parsing DDG's HTML. It can break if DDG changes their markup or starts blocking more aggressively.

## Tech

- **UI**: Tailwind CSS, Space Grotesk + JetBrains Mono fonts
- **Markdown**: marked.js with highlight.js for code blocks
- **Screenshots**: html2canvas
- **Icons**: Font Awesome
- **Companion**: Python standard library only (http.server, subprocess, urllib)

