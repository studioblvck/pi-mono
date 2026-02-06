# Pi Monorepo -- Architecture Guide

A plain-English walkthrough of everything inside this repo: what each piece does, how they connect, where security lives, and how you can jump in and start building powerful agentic systems.

---

## 1. The 30-Second Elevator Pitch

Pi is a **toolkit for building AI agents**. Think of it as a layered cake:

- **Bottom layer** -- talk to any LLM (OpenAI, Anthropic, Google, Mistral, local models...)
- **Middle layer** -- give an LLM tools (bash, file editing, search) and let it act autonomously
- **Top layer** -- polished apps: a terminal coding assistant, a Slack bot, a web chat UI, a GPU pod manager

Everything is TypeScript, everything is open-source (MIT), and all seven packages live in this single monorepo.

---

## 2. Package Map -- The Big Picture

```mermaid
graph TD
    subgraph "Applications (what users run)"
        CA["pi-coding-agent<br/><i>Terminal coding CLI</i>"]
        MOM["pi-mom<br/><i>Slack bot</i>"]
        PODS["pi-pods<br/><i>GPU pod manager</i>"]
        WEBUI["pi-web-ui<br/><i>Web chat components</i>"]
    end

    subgraph "Core Libraries (what apps are built on)"
        AGENT["pi-agent-core<br/><i>Agent runtime</i>"]
        AI["pi-ai<br/><i>Unified LLM API</i>"]
        TUI["pi-tui<br/><i>Terminal UI library</i>"]
    end

    CA --> AGENT
    CA --> AI
    CA --> TUI
    MOM --> AGENT
    MOM --> AI
    MOM --> CA
    PODS --> AGENT
    WEBUI --> AI
    WEBUI --> TUI
    AGENT --> AI

    style AI fill:#4a9eff,color:#fff
    style AGENT fill:#7c5cfc,color:#fff
    style TUI fill:#2db87f,color:#fff
    style CA fill:#ff6b6b,color:#fff
    style MOM fill:#ffa94d,color:#fff
    style PODS fill:#e599f7,color:#fff
    style WEBUI fill:#20c997,color:#fff
```

**Read the arrows as "depends on".** Everything flows down to `pi-ai`, the foundation that talks to LLMs.

---

## 3. What Each Package Does (Plain English)

### 3.1 `pi-ai` -- The Universal Translator

**One sentence:** Lets you call *any* LLM provider with a single, consistent API.

```mermaid
graph LR
    YOU["Your Code"] --> PIAI["pi-ai"]
    PIAI --> OpenAI
    PIAI --> Anthropic
    PIAI --> Google
    PIAI --> Bedrock["AWS Bedrock"]
    PIAI --> Mistral
    PIAI --> Ollama["Ollama / vLLM<br/>(local models)"]
    PIAI --> MORE["...15+ providers"]

    style PIAI fill:#4a9eff,color:#fff
```

**Why it matters:** Without this, you'd write different code for every provider. With it, switching from GPT-4 to Claude to Gemini is a one-line change.

**Key concepts:**
| Concept | What it means |
|---------|--------------|
| **Provider** | A company/service that hosts LLMs (OpenAI, Anthropic, etc.) |
| **Model** | A specific AI brain (e.g. `gpt-4o`, `claude-sonnet-4-20250514`, `gemini-2.0-flash`) |
| **Streaming** | Getting the response word-by-word instead of waiting for the whole thing |
| **Tool calling** | The LLM says "I want to call function X with args Y" -- your code runs it, sends the result back |
| **Thinking** | Some models can "think out loud" before answering (reasoning tokens) |

**Where to look:**
- `packages/ai/src/providers/` -- one file per provider (anthropic.ts, openai-responses.ts, google.ts...)
- `packages/ai/src/stream.ts` -- the unified streaming entry point
- `packages/ai/src/types.ts` -- all the TypeScript types
- `packages/ai/src/models.generated.ts` -- auto-generated catalog of every known model

---

### 3.2 `pi-agent-core` -- The Brain Loop

**One sentence:** Takes an LLM and tools, then runs a loop: ask the LLM -> it picks a tool -> run the tool -> feed result back -> repeat until done.

```mermaid
sequenceDiagram
    participant User
    participant Agent as pi-agent-core
    participant LLM as LLM (via pi-ai)
    participant Tools as Tools (bash, edit, etc.)

    User->>Agent: "Fix the bug in app.ts"
    Agent->>LLM: system prompt + user message
    LLM-->>Agent: tool_call: read("app.ts")
    Agent->>Tools: execute read("app.ts")
    Tools-->>Agent: file contents
    Agent->>LLM: here's the file
    LLM-->>Agent: tool_call: edit("app.ts", fix)
    Agent->>Tools: execute edit("app.ts", fix)
    Tools-->>Agent: success
    Agent->>LLM: edit succeeded
    LLM-->>Agent: "Done! I fixed the null check on line 42."
    Agent-->>User: display response
```

**Key concepts:**
| Concept | What it means |
|---------|--------------|
| **Agent loop** | The ask-tool-respond cycle that keeps going until the LLM says "I'm done" |
| **AgentTool** | A function the LLM can call -- has a name, description, parameter schema, and execute function |
| **Steering** | Interrupting the agent mid-run to redirect it ("Actually, focus on the tests instead") |
| **Follow-up** | Queueing a message to send after the current turn finishes |
| **Events** | The agent emits events (turn_start, message_update, tool_execution, etc.) so UIs can react in real-time |

**Where to look:**
- `packages/agent/src/agent.ts` -- the `Agent` class
- `packages/agent/src/agent-loop.ts` -- the core loop logic
- `packages/agent/src/types.ts` -- message and event types

---

### 3.3 `pi-coding-agent` -- The Star of the Show

**One sentence:** A terminal app (like Cursor or Claude Code) that uses the agent loop to read, write, edit, and execute code on your machine.

```mermaid
graph TD
    subgraph "pi-coding-agent"
        CLI["CLI Entry<br/>parse args, pick mode"]

        subgraph "Three Run Modes"
            INT["Interactive Mode<br/>(full TUI)"]
            PRINT["Print Mode<br/>(pipe-friendly stdout)"]
            RPC["RPC Mode<br/>(JSON stdin/stdout)"]
        end

        subgraph "Core Engine"
            SESSION["Session Manager<br/>save/load/branch conversations"]
            TOOLS["Built-in Tools<br/>bash, read, write, edit,<br/>grep, find, ls"]
            COMPACT["Compaction<br/>auto-summarize old context"]
            EXT["Extensions<br/>custom tools, commands, UI"]
            SKILLS["Skills<br/>reusable prompt templates"]
        end
    end

    CLI --> INT
    CLI --> PRINT
    CLI --> RPC
    INT --> SESSION
    INT --> TOOLS
    SESSION --> COMPACT
    SESSION --> EXT
    SESSION --> SKILLS

    style CLI fill:#ff6b6b,color:#fff
    style INT fill:#ff8787,color:#fff
    style PRINT fill:#ffa8a8,color:#fff
    style RPC fill:#ffc9c9,color:#000
```

**The three modes explained:**

| Mode | When to use | How it works |
|------|------------|--------------|
| **Interactive** | Day-to-day coding | Full terminal UI with editor, markdown, syntax highlighting |
| **Print** | Piping output | Streams text to stdout -- great for `pi "explain this" \| less` |
| **RPC** | Embedding in other tools | JSON protocol over stdin/stdout -- other programs can control pi |

**Built-in tools the LLM can use:**

```mermaid
graph LR
    LLM["LLM Decision"]
    LLM --> BASH["bash<br/>Run shell commands"]
    LLM --> READ["read<br/>Read files"]
    LLM --> WRITE["write<br/>Create files"]
    LLM --> EDIT["edit<br/>Modify files (diff-based)"]
    LLM --> GREP["grep<br/>Search file contents"]
    LLM --> FIND["find<br/>Search file names"]
    LLM --> LS["ls<br/>List directories"]

    style LLM fill:#7c5cfc,color:#fff
```

**Where to look:**
- `packages/coding-agent/src/cli.ts` -- CLI entry point
- `packages/coding-agent/src/core/tools/` -- each tool implementation
- `packages/coding-agent/src/core/agent-session.ts` -- session management
- `packages/coding-agent/src/core/extensions/` -- extension system
- `packages/coding-agent/src/modes/` -- the three run modes

---

### 3.4 `pi-tui` -- The Pretty Terminal

**One sentence:** A library for building rich terminal UIs with smart re-rendering (only redraws what changed).

```mermaid
graph TD
    subgraph "pi-tui Component Tree"
        TUI["TUI (root)"]
        TUI --> BOX["Box (layout)"]
        BOX --> MD["Markdown"]
        BOX --> INPUT["Input"]
        BOX --> EDITOR["Editor"]
        TUI --> OVERLAY["Overlay"]
        OVERLAY --> SELECT["SelectList"]
        OVERLAY --> SETTINGS["SettingsList"]
        TUI --> LOADER["Loader"]
        TUI --> IMG["Image"]
    end

    style TUI fill:#2db87f,color:#fff
```

**Key idea:** Differential rendering. Instead of clearing and redrawing the whole screen 60 times per second (which causes flicker), it compares the old screen with the new one and only updates the lines that changed.

---

### 3.5 `pi-mom` -- The Slack Bot

**One sentence:** A Slack bot that receives messages and delegates them to pi-coding-agent running in a sandbox.

```mermaid
sequenceDiagram
    participant Slack
    participant MOM as pi-mom
    participant Agent as pi-coding-agent
    participant Sandbox as Docker Sandbox

    Slack->>MOM: @pi-bot fix the deploy script
    MOM->>Agent: create session for #channel
    Agent->>Sandbox: bash: read deploy.sh
    Sandbox-->>Agent: file contents
    Agent->>Sandbox: bash: edit + test
    Sandbox-->>Agent: tests pass
    Agent-->>MOM: "Fixed! Here's what I changed..."
    MOM->>Slack: reply in thread
```

**Where to look:**
- `packages/mom/src/slack.ts` -- Slack Socket Mode connection
- `packages/mom/src/agent.ts` -- agent orchestration
- `packages/mom/src/sandbox.ts` -- Docker sandbox config

---

### 3.6 `pi-web-ui` -- Web Chat Components

**One sentence:** Drop-in web components (built with Lit) for AI chat interfaces, with artifact rendering (HTML, PDF, code, images).

```mermaid
graph TD
    subgraph "pi-web-ui"
        CHAT["ChatPanel"]
        CHAT --> MSGS["Message List"]
        CHAT --> INPUT["Input Editor"]
        CHAT --> ARTIFACTS["Artifacts Panel"]

        ARTIFACTS --> HTML["HTML Artifact"]
        ARTIFACTS --> SVG["SVG Artifact"]
        ARTIFACTS --> PDF["PDF Artifact"]
        ARTIFACTS --> DOCX["DOCX Artifact"]
        ARTIFACTS --> EXCEL["Excel Artifact"]
        ARTIFACTS --> IMG["Image Artifact"]
        ARTIFACTS --> MD["Markdown Artifact"]

        subgraph "Storage (IndexedDB)"
            SESS["Sessions"]
            SETT["Settings"]
            KEYS["Provider Keys"]
        end
    end

    style CHAT fill:#20c997,color:#fff
```

---

### 3.7 `pi-pods` -- GPU Pod Manager

**One sentence:** A CLI to deploy and manage vLLM (a fast open-source LLM server) on rented GPU machines.

```mermaid
graph LR
    CLI["pi-pods CLI"] -->|SSH| POD1["GPU Pod 1<br/>Qwen 72B"]
    CLI -->|SSH| POD2["GPU Pod 2<br/>DeepSeek"]
    CLI -->|SSH| POD3["GPU Pod 3<br/>Llama 3"]

    POD1 -->|OpenAI-compatible API| APP["Your App"]
    POD2 -->|OpenAI-compatible API| APP
    POD3 -->|OpenAI-compatible API| APP

    style CLI fill:#e599f7,color:#fff
```

---

## 4. Full System Architecture

Here's how everything fits together when the coding agent is running:

```mermaid
graph TB
    subgraph "User Layer"
        TERM["Terminal / IDE"]
        BROWSER["Browser"]
        SLACK["Slack"]
    end

    subgraph "Application Layer"
        CA["pi-coding-agent"]
        MOM["pi-mom"]
        WEBUI["pi-web-ui"]
        PODS["pi-pods"]
    end

    subgraph "Runtime Layer"
        AGENT["pi-agent-core<br/>(agent loop, tools, state)"]
        TUI["pi-tui<br/>(terminal rendering)"]
    end

    subgraph "Provider Layer"
        AI["pi-ai<br/>(unified LLM API)"]
    end

    subgraph "External Services"
        OPENAI["OpenAI"]
        ANTHROPIC["Anthropic"]
        GOOGLE["Google"]
        BEDROCK["AWS Bedrock"]
        LOCAL["Local (Ollama/vLLM)"]
        GPU["GPU Pods"]
    end

    subgraph "Local Machine"
        FS["File System"]
        SHELL["Shell / Bash"]
        GIT["Git"]
    end

    TERM --> CA
    BROWSER --> WEBUI
    SLACK --> MOM

    CA --> AGENT
    CA --> TUI
    MOM --> AGENT
    WEBUI --> AI
    PODS --> GPU

    AGENT --> AI
    AI --> OPENAI
    AI --> ANTHROPIC
    AI --> GOOGLE
    AI --> BEDROCK
    AI --> LOCAL

    AGENT --> FS
    AGENT --> SHELL
    AGENT --> GIT

    style AI fill:#4a9eff,color:#fff
    style AGENT fill:#7c5cfc,color:#fff
    style CA fill:#ff6b6b,color:#fff
```

---

## 5. Data Flow -- A Single Coding Request

What happens when you type `"add error handling to server.ts"` in the terminal:

```mermaid
flowchart TD
    A["You type a prompt"] --> B["CLI parses input"]
    B --> C["AgentSession adds message to history"]
    C --> D["Agent sends context to LLM via pi-ai"]
    D --> E{"LLM responds"}

    E -->|"Text response"| F["Display in TUI"]
    E -->|"Tool call"| G["Agent executes tool"]

    G --> H{"Which tool?"}
    H -->|read| I["Read file from disk"]
    H -->|bash| J["Execute shell command"]
    H -->|edit| K["Apply diff to file"]
    H -->|write| L["Write new file"]
    H -->|grep/find/ls| M["Search filesystem"]

    I --> N["Send result back to LLM"]
    J --> N
    K --> N
    L --> N
    M --> N

    N --> D
    F --> O["Wait for next prompt"]

    style A fill:#4a9eff,color:#fff
    style E fill:#7c5cfc,color:#fff
    style G fill:#ff6b6b,color:#fff
    style O fill:#2db87f,color:#fff
```

The loop (D -> E -> G -> N -> D) is the **agent loop**. It keeps going until the LLM decides it's done and responds with just text (no tool calls).

---

## 6. Security Architecture

### 6.1 Credential Management

```mermaid
graph TD
    subgraph "API Key Resolution (Priority Order)"
        P1["1. CLI flag: --api-key"]
        P2["2. auth.json file<br/>(chmod 600)"]
        P3["3. OAuth token<br/>(auto-refreshed)"]
        P4["4. Environment variable<br/>(OPENAI_API_KEY, etc.)"]
        P5["5. Custom resolver<br/>(extensions)"]
    end

    P1 --> P2 --> P3 --> P4 --> P5

    subgraph "OAuth Flows"
        OAUTH["OAuth Login"]
        OAUTH --> PKCE["PKCE Flow<br/>(Proof Key for Code Exchange)"]
        OAUTH --> DEVICE["Device Code Flow<br/>(for CLI apps)"]
    end

    subgraph "File Security"
        AUTH["~/.pi/auth.json"]
        LOCK["File Locking<br/>(proper-lockfile)"]
        PERM["Permissions: 0o600<br/>(owner read/write only)"]
    end

    style P1 fill:#ff6b6b,color:#fff
    style AUTH fill:#ffa94d,color:#fff
```

**What this means for you:**
- Credentials are stored in `~/.pi/auth.json` with strict file permissions (only your user can read it)
- File locking prevents race conditions when multiple agent instances run simultaneously
- OAuth uses PKCE -- the most secure flow for CLI apps (no secrets stored in the app itself)
- The system tries multiple sources in order, so you can set keys however you prefer

### 6.2 Command Execution Security

```mermaid
graph TD
    subgraph "Bash Tool Security Layers"
        CMD["User/LLM requests command"]
        CMD --> SPAWN["spawn() with detached process"]
        SPAWN --> TIMEOUT["Configurable timeout"]
        TIMEOUT --> ABORT["AbortSignal for cancellation"]
        ABORT --> SANITIZE["Output sanitization"]
        SANITIZE --> STRIP["Strip ANSI escapes"]
        SANITIZE --> BINARY["Remove binary garbage"]
        SANITIZE --> TRUNC["Truncate large output (10MB cap)"]
        TRUNC --> KILL["Process tree cleanup<br/>SIGTERM -> 5s -> SIGKILL"]
    end

    subgraph "pi-mom Sandbox"
        DOCKER["Docker Container"]
        DOCKER --> ISOLATED["Isolated filesystem"]
        DOCKER --> NETWORK["Controlled network"]
        DOCKER --> PERSIST["Persistent workspace per channel"]
    end

    style CMD fill:#ff6b6b,color:#fff
    style DOCKER fill:#4a9eff,color:#fff
```

**Key security measures:**

| Layer | What it does | Why it matters |
|-------|-------------|----------------|
| **Detached processes** | Bash commands run in their own process group | Can kill the whole tree if something hangs |
| **Timeouts** | Commands auto-kill after N seconds | Prevents infinite loops from eating resources |
| **Output sanitization** | Strips ANSI codes, binary data | Prevents terminal injection attacks |
| **Output truncation** | Caps at ~10MB | Prevents memory exhaustion |
| **Process tree kill** | SIGTERM then SIGKILL | No orphaned processes left behind |
| **Docker sandbox (mom)** | Slack bot runs in isolated containers | Untrusted Slack users can't escape to host |
| **File permissions** | auth.json is 0o600 | Other users on the machine can't read your keys |
| **Tool schema validation** | TypeBox + AJV validate all tool params | LLM can't pass malformed data to tools |

### 6.3 Input Validation Pipeline

```mermaid
flowchart LR
    LLM["LLM generates<br/>tool call JSON"] --> PARSE["partial-json<br/>parser"]
    PARSE --> SCHEMA["TypeBox schema<br/>validation"]
    SCHEMA --> AJV["AJV JSON<br/>schema validator"]
    AJV -->|Valid| EXEC["Execute tool"]
    AJV -->|Invalid| ERR["Return error<br/>to LLM"]

    style LLM fill:#7c5cfc,color:#fff
    style EXEC fill:#2db87f,color:#fff
    style ERR fill:#ff6b6b,color:#fff
```

---

## 7. Extension Architecture

One of the most powerful features for building your own agentic systems:

```mermaid
graph TD
    subgraph "Extension Loading"
        GLOBAL["~/.pi/extensions/"]
        PROJECT[".pi/extensions/"]
        PKG["npm packages"]
        CLI_FLAG["--extension flag"]
    end

    subgraph "What Extensions Can Do"
        TOOLS["Register custom tools"]
        COMMANDS["Add slash commands"]
        EVENTS["Subscribe to events<br/>(turn_start, turn_end, tool_execution)"]
        UI["Show dialogs, notifications"]
        KEYS["Add keyboard shortcuts"]
        WRAP["Wrap existing tools<br/>(intercept, modify, log)"]
        SYSTEM["Modify system prompt"]
    end

    GLOBAL --> LOADER["Extension Loader"]
    PROJECT --> LOADER
    PKG --> LOADER
    CLI_FLAG --> LOADER
    LOADER --> RUNNER["Extension Runner"]
    RUNNER --> TOOLS
    RUNNER --> COMMANDS
    RUNNER --> EVENTS
    RUNNER --> UI
    RUNNER --> KEYS
    RUNNER --> WRAP
    RUNNER --> SYSTEM

    style LOADER fill:#7c5cfc,color:#fff
    style RUNNER fill:#4a9eff,color:#fff
```

**This is your main entry point for building custom agentic systems.** You can:
1. Add new tools (web scraping, database queries, API calls, etc.)
2. Wrap existing tools (add logging, approval gates, rate limiting)
3. React to events (trigger actions when the agent reads a file, runs a command, etc.)
4. Build custom UIs (dialogs, settings panels)

---

## 8. Session & Memory Architecture

```mermaid
graph TD
    subgraph "Session Tree"
        ROOT["Root message"]
        ROOT --> B1["Branch 1"]
        ROOT --> B2["Branch 2 (fork)"]
        B1 --> B1A["Continue..."]
        B1 --> B1B["Fork from here"]
        B2 --> B2A["Different approach"]
    end

    subgraph "Persistence"
        JSONL["session.jsonl<br/>(message log)"]
        META["metadata.json<br/>(session info)"]
    end

    subgraph "Compaction"
        FULL["Full conversation<br/>(getting too long)"]
        FULL --> SUMMARIZE["LLM summarizes<br/>old messages"]
        SUMMARIZE --> COMPACT["Compacted context<br/>(summary + recent messages)"]
    end

    B1A --> JSONL
    COMPACT --> AGENT["Agent continues<br/>with fresh context window"]

    style ROOT fill:#4a9eff,color:#fff
    style COMPACT fill:#2db87f,color:#fff
```

**Why this matters:**
- **Branching** -- Try different approaches without losing your history (like git branches for conversations)
- **Compaction** -- When the conversation gets too long for the LLM's context window, older messages are summarized automatically so you can keep working indefinitely
- **Persistence** -- Sessions are saved as JSONL files, so you can resume any time

---

## 9. The Technology Stack (Glossary for Beginners)

```mermaid
mindmap
  root((Pi Stack))
    Language
      TypeScript
        Strict mode
        ES2022 target
        ESM modules
    Build
      tsgo (compiler)
      Biome (lint + format)
      npm workspaces (monorepo)
      Vitest (testing)
      Husky (git hooks)
    AI/LLM
      Tool calling / Function calling
      Streaming responses
      Extended thinking / Reasoning
      Multi-provider abstraction
      Token counting & cost tracking
    Terminal
      Differential rendering
      ANSI escape codes
      Kitty/iTerm2 image protocol
      IME support (CJK)
    Web
      Lit (web components)
      Tailwind CSS
      IndexedDB (browser storage)
    Infrastructure
      GitHub Actions (CI/CD)
      Docker (sandboxing)
      SSH (GPU pods)
      vLLM (model serving)
    Security
      OAuth PKCE
      File permission (0600)
      Schema validation
      Process isolation
      Output sanitization
```

| Term | Plain English |
|------|--------------|
| **TypeScript** | JavaScript with types -- catches bugs before you run the code |
| **ESM** | Modern `import/export` syntax (vs old `require()`) |
| **Monorepo** | All packages in one repo -- shared tooling, atomic changes |
| **npm workspaces** | npm's built-in way to manage multiple packages in one repo |
| **Biome** | Fast linter and formatter (replaces ESLint + Prettier) |
| **Vitest** | Test runner (like Jest but faster, built for ESM) |
| **Lit** | Google's library for building web components |
| **IndexedDB** | Browser database for storing sessions/settings |
| **vLLM** | Open-source server for running LLMs on GPUs efficiently |
| **PKCE** | Secure OAuth flow that doesn't require storing a client secret |
| **JSONL** | One JSON object per line -- easy to append, stream, and parse |
| **Tool calling** | LLM says "run this function" -- the framework runs it and sends results back |
| **Compaction** | Summarizing old conversation to fit in the context window |
| **TypeBox** | Define schemas in TypeScript that work at runtime for validation |

---

## 10. How to Build Your Own Agentic System on Pi

### Path 1: Extension (Simplest)

Write a TypeScript file, drop it in `.pi/extensions/`, and it runs inside pi.

```mermaid
flowchart LR
    A["Write extension.ts"] --> B["Drop in .pi/extensions/"]
    B --> C["pi loads it at startup"]
    C --> D["Your custom tools<br/>are available to the LLM"]

    style D fill:#2db87f,color:#fff
```

### Path 2: Custom Agent (Full Control)

Use `pi-agent-core` and `pi-ai` as libraries in your own app.

```mermaid
flowchart TD
    A["npm install @mariozechner/pi-ai @mariozechner/pi-agent-core"]
    A --> B["Create tools with AgentTool interface"]
    B --> C["Instantiate Agent with your tools"]
    C --> D["Call agent.run() with user messages"]
    D --> E["Agent loops: LLM <-> Tools"]
    E --> F["Your custom agentic application"]

    style F fill:#4a9eff,color:#fff
```

### Path 3: Full-Stack Agent App

Combine the web UI with a backend agent for a ChatGPT-like experience with custom tools.

```mermaid
graph TD
    subgraph "Frontend"
        WEBUI["pi-web-ui components"]
        WEBUI --> CHAT["ChatPanel"]
        WEBUI --> ART["Artifacts viewer"]
    end

    subgraph "Backend"
        PROXY["Agent Proxy (HTTP)"]
        PROXY --> AGENT["pi-agent-core"]
        AGENT --> AI["pi-ai"]
        AGENT --> CUSTOM["Your custom tools"]
    end

    CHAT <-->|"WebSocket/HTTP"| PROXY
    AI --> PROVIDERS["LLM Providers"]

    style WEBUI fill:#20c997,color:#fff
    style AGENT fill:#7c5cfc,color:#fff
```

---

## 11. Directory Layout Quick Reference

```
pi-mono/
|-- packages/
|   |-- ai/                  # LLM provider abstraction
|   |   |-- src/providers/   # One file per LLM provider
|   |   |-- src/stream.ts    # Unified streaming entry point
|   |   +-- src/types.ts     # Core types
|   |
|   |-- agent/               # Agent runtime
|   |   |-- src/agent.ts     # Agent class
|   |   +-- src/agent-loop.ts# The ask-tool-respond loop
|   |
|   |-- coding-agent/        # Terminal coding assistant
|   |   |-- src/core/tools/  # bash, read, write, edit, grep, find, ls
|   |   |-- src/core/extensions/ # Extension system
|   |   |-- src/modes/       # interactive, print, rpc
|   |   +-- docs/            # Extensive documentation
|   |
|   |-- tui/                 # Terminal UI components
|   |   +-- src/components/  # box, input, editor, markdown, etc.
|   |
|   |-- mom/                 # Slack bot
|   |   +-- src/slack.ts     # Socket Mode integration
|   |
|   |-- web-ui/              # Web chat components (Lit)
|   |   |-- src/components/  # ChatPanel, messages, artifacts
|   |   +-- src/storage/     # IndexedDB backend
|   |
|   +-- pods/                # GPU pod management
|       +-- src/commands/    # pods, models, prompt
|
|-- .github/workflows/       # CI/CD pipelines
|-- biome.json               # Linter/formatter config
|-- tsconfig.json            # Root TypeScript config
+-- package.json             # Workspace root
```

---

## 12. Contributing Checklist

```mermaid
flowchart TD
    START["Pick something to work on"] --> INSTALL["npm install"]
    INSTALL --> BUILD["npm run build"]
    BUILD --> CODE["Make your changes"]
    CODE --> CHECK["npm run check<br/>(lint + types)"]
    CHECK -->|Errors| CODE
    CHECK -->|Clean| TEST["Run relevant tests"]
    TEST -->|Failing| CODE
    TEST -->|Passing| COMMIT["Commit with descriptive message"]

    style START fill:#4a9eff,color:#fff
    style COMMIT fill:#2db87f,color:#fff
```

**Rules of the road:**
- No `any` types unless absolutely necessary
- No inline/dynamic imports -- always top-level `import` statements
- Run `npm run check` before committing (it catches lint, format, and type errors)
- Never run `npm run dev`, `npm run build`, or `npm test` (use specific test commands instead)
- Use `git add <specific-files>` not `git add .` (multiple agents may work in parallel)

---

## 13. Key Design Decisions

| Decision | Why |
|----------|-----|
| **npm workspaces (no Turborepo/Nx)** | Simplicity -- fewer tools, less config, npm handles it |
| **TypeBox for schemas** | Schemas defined in TypeScript, validated at runtime, used for LLM tool params |
| **File-based storage (no database)** | Sessions are JSON files -- no DB to set up, easy to inspect and debug |
| **ESM only** | Modern standard, tree-shakeable, faster startup |
| **Biome (not ESLint)** | Orders of magnitude faster, single tool for lint + format |
| **Detached process spawning** | Full control over process lifecycle, clean kills |
| **Differential TUI rendering** | No flicker, low CPU, works over slow SSH connections |
| **Web Components (Lit)** | Framework-agnostic -- works in React, Vue, Svelte, or vanilla HTML |

---

*This document was generated to help new contributors understand the Pi monorepo architecture. For detailed API docs, see the README.md in each package directory.*
