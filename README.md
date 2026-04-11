# SlopLobster



https://github.com/user-attachments/assets/0bc3c230-282a-4efb-a58e-365e82af710b



# SlopLobster

A local AI coding agent that runs entirely in your browser and talks to LM Studio. It can read/write files, run shell commands, search the web, and do iterative development — all on your machine, nothing leaves localhost.

This is a personal project. No guarantees about safety, correctness, or fitness for any purpose. It works well enough for me, but your mileage will vary.

## How It Works

```
┌─────────────────────────────────────────────────┐
│  Browser (single HTML file, no build step)       │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Agent    │──│ Tool     │──│ File System   │  │
│  │ Loop     │  │ Dispatch │  │ Access API    │  │
│  │ (stream) │  │          │  │ (read/write)  │  │
│  └────┬─────┘  └────┬─────┘  └───────────────┘  │
│       │              │                           │
│       │         ┌────┴─────┐                     │
│       │         │Companion │── Shell / Git /     │
│       │         │Server    │   Web Search        │
│       │         └──────────┘                     │
│       │                                          │
│       ▼                                          │
│  ┌──────────┐                                    │
│  │ LM Studio│  Local LLM inference               │
│  │ API      │  (OpenAI-compatible)               │
│  └──────────┘                                    │
└─────────────────────────────────────────────────┘
```

- **LM Studio** runs the LLM. SlopLobster streams tool calls from it in a loop.
- **File System Access API** (Chrome/Edge) gives direct read/write to a workspace directory — no server needed for file ops.
- **Companion server** (a small Python script) handles shell execution, git, and web search. Everything stays on 127.0.0.1.

## Setup

1. **Install [LM Studio](https://lmstudio.ai/)** and load a model. Tool-calling models work best (Qwen3.5 etc.).
2. **Open `index.html`** in Chrome or Edge (File System Access API required).
3. **Optional: Set up the companion** for shell/git/web search:
   - Click "Save" in the sidebar to download `SlopLobster-companion.py`
   - Run it: `python SlopLobster-companion.py`
   - Click "Connect" in the sidebar

That's it. No npm, no build, no API keys, no cloud.

## Features

### Agent Loop
- Streams responses with tool calling in a loop until the agent finishes
- `think` tool for visible chain-of-thought (collapsible)
- Parallel tool execution when tools don't depend on each other
- Automatic retry with web search when stuck in failure loops
- Text loop detection (Jaccard similarity on recent responses)
- Configurable iteration limits with auto-compaction

### File Operations
- `read_file` / `read_file_lines` — with image detection (sends to vision models visually)
- `edit_file` — search/replace with whitespace-normalized fallback and diff preview
- `write_file` — for new files only
- `list_directory` / `search_files` / `grep` — with recursive search
- `view_image` — SVG preview with rendered/code toggle, raster preview
- `delete_file` / `move_file`
- Write verification (reads back after write)

### Right Panel
- **Files** — click any file to preview with line numbers and copy-on-click
- **Changes** — tracks all modifications with compact diffs
- **Term** — live streaming terminal output from shell commands
- **Git** — branch, status, quick stage/log

### Context Management
- Token estimation with live meter
- Auto-compact at configurable threshold (default 85%)
- Progress persistence: saves `.sloplobster-progress.md` to workspace before compaction
- Agent reads progress file after compact to recover full context
- Configurable max iterations with soft warning and auto-stop

### Vision
- Screenshot capture with region selection (screen share → crop)
- Image paste from clipboard
- File drag-and-drop for images
- Image attachments in messages
- Vision capability auto-detected from model name/metadata

### Model Loading
- Load/unload models in LM Studio directly from SlopLobster
- Configure: context length override, flash attention, KV cache offload, MoE experts, eval batch size
- Shows loaded model status, load time, and config summary

### Other
- Conversation save/fork/delete with localStorage
- Export as Markdown
- Input history (arrow keys)
- Keyboard shortcuts (Ctrl+B, Ctrl+K, /help, etc.)
- Reference sites list for directing the agent to authoritative docs

## Safety Layers

This project has several safety mechanisms. They are **not guarantees** — they are reasonable precautions for a personal tool.

### Path Traversal Blocking
```
sanitizePath("../../etc/passwd")  → Error: ".." segment detected
sanitizePath("....\\\\passwd")    → Error: "...." segment detected
```
All file paths go through `sanitizePath()` which rejects `..` segments, null bytes, and suspicious dot-prefixed segments. File operations are additionally sandboxed to the workspace directory via the File System Access API handle.

### Dangerous Command Detection
Commands matching patterns like `rm -rf /`, `dd of=/dev/`, `mkfs`, `chmod 777 /`, `git push --force`, `curl | sh`, etc. require explicit user confirmation before execution.

### Localhost Only
- The companion server binds to `127.0.0.1` only
- LM Studio API is typically `localhost:1234`
- No external network calls from the HTML file itself
- Web search goes through the companion's DuckDuckGo scraping (no API key)

### Output Sanitization
- DOMPurify sanitizes all markdown output (no XSS from LLM responses)
- Binary file detection prevents reading non-text files as text
- Truncation limits on file reads, command output, and search results

### Resource Limits
- Configurable max file read size (default 500 KB)
- Command timeout (default 60s, auto-extended to 300s for package installs)
- Max output truncation (100 KB from companion)
- Max iterations before auto-stop (default 50)
- Context auto-compact prevents unbounded memory growth

### Sandboxed File System
The File System Access API only grants access to the directory the user explicitly picks. The companion server's file operations don't exist — all file I/O goes through the browser API, which is scoped to the chosen workspace.

### What It Doesn't Protect Against
- A sufficiently determined LLM could potentially social-engineer the user into approving a dangerous command
- If the companion server has bugs, shell commands could theoretically escape normal boundaries
- The LLM itself could produce misleading code that does something unexpected when run
- No sandboxing of the LLM's output code — if you run it, it runs with your user permissions

## Companion Server

The companion is a single Python file with no dependencies beyond the standard library. It provides:

- **Shell execution** with streaming output (NDJSON), process group killing on timeout, auto-fallback between `python`/`python3`
- **Web search** via DuckDuckGo HTML scraping (no API key needed)
- **URL fetching** with HTML-to-text extraction (strips nav, scripts, styles; detects JS-rendered pages)
- **Git operations** proxied through shell commands
- Windows support (auto-detects Git Bash, translates cmd.exe commands)

## Requirements

- **Browser**: Chrome 86+ or Edge 86+ (File System Access API + `getDisplayMedia`)
- **LM Studio**: [lmstudio.ai](https://lmstudio.ai/) with a loaded model
- **Companion** (optional): Python 3.8+ with no pip dependencies
- **Models**: Tool-calling models work best. Vision models (Qwen-3.5 etc.) enable image understanding. Non-tool-calling models will work but with degraded agent behavior.

## Tech

Single HTML file. No build step. No bundler. Dependencies loaded from CDN:

- Tailwind CSS
- JetBrains Mono + Space Grotesk fonts
- highlight.js
- marked
- DOMPurify
- html2canvas (screenshot fallback)
- Font Awesome icons

The companion server is pure stdlib Python.

## License

This is a personal project. Use it if it's useful to you, don't if it's not. No warranty, no liability, no support guarantees. The code is here for reference.
