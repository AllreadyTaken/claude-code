# Claude Code - arc42 Architecture Documentation

**Version:** 2.1.88
**Date:** 2026-04-01
**Status:** Extracted from source map analysis

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

Claude Code is an interactive CLI tool that provides AI-assisted software engineering capabilities. It enables users to:

- Execute natural language commands for code manipulation
- Read, search, edit, and create files through an AI agent
- Run shell commands with permission controls
- Use skills and plugins for extensibility
- Operate in interactive (REPL) or headless (SDK) modes
- Support multi-agent/swarm architectures
- Remote control via bridge/web interface

### 1.2 Quality Goals

| Priority | Quality Goal | Motivation |
|----------|-------------|------------|
| High | Responsiveness | Interactive CLI requires low-latency streaming |
| High | Security | File system and command execution require strict permissions |
| High | Extensibility | Plugin, skill, and tool systems for customization |
| Medium | Reliability | Session recovery, state persistence |
| Medium | Performance | Efficient context window management, lazy loading |

### 1.3 Stakeholders

| Role | Contact | Expectations |
|------|---------|-------------|
| End Users (Developers) | - | Fast, reliable AI coding assistant |
| Anthropic (Vendor) | - | IP protection, usage analytics |
| Plugin/Skill Developers | - | Extensible APIs |
| SDK Consumers | - | Headless API, TypeScript/Python SDKs |

---

## 2. Architecture Constraints

### 2.1 Technical Constraints

- **Node.js >= 18** as minimum runtime
- **Zero runtime dependencies** - all code bundled into single `cli.js`
- **Bun** used as bundler with feature flag dead code elimination
- **Source map extraction** - this documentation is based on reverse-engineered code
- **Single binary distribution** via npm package

### 2.2 Organizational Constraints

- Code extracted from npm package source map (not official source)
- No access to original build tooling or development environment
- Documentation based on static analysis only

### 2.3 Conventions

- TypeScript throughout (compiled to JavaScript)
- React/Ink for terminal UI
- Async generators for streaming
- Centralized immutable state store
- Feature flags via `bun:bundle`

---

## 3. Context and Scope

### 3.1 Business Context

```
┌─────────────────────────────────────────────────────┐
│                      User                            │
│         (Developer using CLI or SDK)                 │
└──────────────────┬──────────────────────────────────┘
                   │ Commands / Prompts
                   ▼
┌─────────────────────────────────────────────────────┐
│                  Claude Code                         │
│  ┌─────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │   CLI   │  │   SDK    │  │   Bridge/Remote  │   │
│  │  (REPL) │  │(Headless)│  │  (Web/Mobile)    │   │
│  └────┬────┘  └────┬─────┘  └────────┬─────────┘   │
│       └────────────┼─────────────────┘              │
│                    ▼                                 │
│          ┌─────────────────┐                         │
│          │  Query Engine   │                         │
│          └────────┬────────┘                         │
└───────────────────┼─────────────────────────────────┘
                    │ API Calls
                    ▼
┌─────────────────────────────────────────────────────┐
│              Anthropic API (Claude)                  │
└─────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────┐
│           External Systems (Optional)                │
│  ┌────────┐  ┌─────┐  ┌─────┐  ┌────────┐          │
│  │  MCP   │  │ LSP │  │ Web │  │  Git   │          │
│  │Servers │  │     │  │Search│  │        │          │
│  └────────┘  └─────┘  └─────┘  └────────┘          │
└─────────────────────────────────────────────────────┘
```

### 3.2 Technical Context

| Component | Technology | Purpose |
|-----------|-----------|---------|
| CLI Framework | Commander.js | Argument parsing, subcommands |
| Terminal UI | Ink (React-based) | Interactive terminal rendering |
| State Management | Custom immutable store | Centralized app state |
| API Client | @anthropic-ai/sdk | Communication with Claude API |
| Validation | Zod v4 | Tool input schema validation |
| Protocol | MCP SDK | Model Context Protocol integration |
| Telemetry | OpenTelemetry | Metrics and tracing |
| Feature Flags | GrowthBook + bun:bundle | Runtime and compile-time flags |

---

## 4. Solution Strategy

### 4.1 Core Architecture Patterns

1. **Async Generator Query Loop** - Streaming API responses with stateful turn management
2. **Tool Interface Pattern** - Polymorphic tool execution with permissions, rendering, and validation
3. **Command System** - Three-tier command types (prompt, local, local-jsx)
4. **Centralized Immutable State** - DeepImmutable<AppState> with selector pattern
5. **Feature Flag Gating** - Compile-time dead code elimination via bun:bundle
6. **Lazy/Dynamic Imports** - Deferred module loading for startup performance
7. **Custom Terminal UI Framework** - Ink-based reconciler with ANSI parsing

### 4.2 Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Single bundled binary | Simplifies distribution, zero dependency conflicts |
| Async generators for query loop | Enables streaming while maintaining state |
| Immutable state store | Predictable state transitions, easy debugging |
| React for terminal UI | Component composition, ecosystem |
| Feature flags at compile time | Zero runtime overhead for disabled features |
| Permission-first tool execution | Security by design |

---

## 5. Building Block View

### 5.1 Whitebox Overall System

```
┌──────────────────────────────────────────────────────────────┐
│                       Claude Code                             │
│                                                               │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────────┐   │
│  │   Entry    │  │    CLI       │  │    Query Engine    │   │
│  │  Points    │──│  Commands    │──│   (Core Loop)      │   │
│  │            │  │   (70+)      │  │                    │   │
│  └────────────┘  └──────────────┘  └─────────┬──────────┘   │
│                                                │              │
│  ┌────────────┐  ┌──────────────┐  ┌─────────▼──────────┐   │
│  │   State    │  │     UI       │  │      Tools         │   │
│  │   Store    │──│  (Ink/React) │  │    (43 tools)      │   │
│  │            │  │  (144 comps) │  │                    │   │
│  └────────────┘  └──────────────┘  └────────────────────┘   │
│                                                               │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────────┐   │
│  │  Services  │  │   Bridge     │  │      Agent         │   │
│  │  (36 mods) │  │  (Remote)    │  │    (Swarm)         │   │
│  └────────────┘  └──────────────┘  └────────────────────┘   │
│                                                               │
│  ┌────────────┐  ┌──────────────┐                            │
│  │  Plugins   │  │   Skills     │                            │
│  │  System    │  │   System     │                            │
│  └────────────┘  └──────────────┘                            │
└──────────────────────────────────────────────────────────────┘
```

### 5.2 Level 2 - Core Domains

#### 5.2.1 Entry and CLI Domain
- **main.tsx** - Application bootstrap, profiling, client detection
- **entrypoints/** - CLI, init, MCP, SDK entry points
- **commands.ts** - Command registry
- **commands/** - 70+ slash command implementations

#### 5.2.2 Query and AI Domain
- **query.ts** - Core async generator query loop
- **QueryEngine.ts** - Headless/SDK query wrapper
- **services/api/** - Anthropic API communication
- **services/compact/** - Context compaction strategies

#### 5.2.3 Tool Domain
- **Tool.ts** - Tool interface definition (792 lines)
- **tools.ts** - Tool registry and assembly
- **tools/** - 43 tool implementations

#### 5.2.4 UI Domain
- **ink/** - Custom Ink terminal framework (48 files)
- **components/** - 144 React components
- **screens/** - REPL, Doctor, ResumeConversation
- **hooks/** - 85 React hooks

#### 5.2.5 State Domain
- **state/AppStateStore.ts** - Central state type and store
- **state/store.ts** - Generic store implementation
- **context/** - React context providers

#### 5.2.6 Bridge/Remote Domain
- **bridge/** - Remote control bridge (31 files)
- **remote/** - Remote session management
- **server/** - Direct connect session server

#### 5.2.7 Agent/Swarm Domain
- **tools/AgentTool/** - Sub-agent spawning
- **coordinator/** - Multi-agent coordination
- **utils/swarm/** - Swarm utilities

#### 5.2.8 Infrastructure Domain
- **services/mcp/** - MCP server management
- **services/analytics/** - Analytics, GrowthBook, telemetry
- **services/lsp/** - LSP integration
- **services/plugins/** - Plugin management

---

## 6. Runtime View

### 6.1 Interactive REPL Flow

```
User Input
    │
    ▼
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│   CLI       │────▶│   setup()    │────▶│  launchRepl()│
│  Parser     │     │              │     │              │
└─────────────┘     └──────────────┘     └──────┬───────┘
                                                 │
                                                 ▼
                                        ┌────────────────┐
                                        │   App (Ink)    │
                                        │   Component    │
                                        └───────┬────────┘
                                                │
                                                ▼
                                       ┌─────────────────┐
                                       │   REPL Screen   │
                                       └────────┬────────┘
                                                │
                                                ▼
                                       ┌─────────────────┐
                                       │   query()       │
                                       │  (async gen)    │
                                       └────────┬────────┘
                                                │
                                    ┌───────────┼───────────┐
                                    ▼           ▼           ▼
                              ┌──────────┐ ┌────────┐ ┌─────────┐
                              │  API     │ │ Tools  │ │Compact   │
                              │ Stream   │ │Execute │ │Context   │
                              └──────────┘ └────────┘ └─────────┘
```

### 6.2 Tool Execution Flow

```
Assistant Message (tool_use)
    │
    ▼
┌──────────────┐    ┌───────────────┐    ┌──────────────┐
│  Parse Tool  │───▶│  Check        │───▶│  Execute     │
│  Call        │    │  Permissions  │    │  Tool        │
└──────────────┘    └───────────────┘    └──────┬───────┘
                                                 │
                                                 ▼
                                        ┌────────────────┐
                                        │  Render Result │
                                        │  (UI/Text)     │
                                        └────────┬───────┘
                                                 │
                                                 ▼
                                        ┌────────────────┐
                                        │  Return to     │
                                        │  Query Loop    │
                                        └────────────────┘
```

### 6.3 Remote Bridge Flow

```
Web/Mobile Client
    │
    ▼
┌──────────────┐    ┌───────────────┐    ┌──────────────┐
│  WebSocket   │───▶│  Bridge       │───▶│  Permission  │
│  Transport   │    │  Session      │    │  Callback    │
└──────────────┘    └───────────────┘    └──────────────┘
```

---

## 7. Deployment View

### 7.1 Distribution

```
┌─────────────────────────────────────┐
│         npm Package                  │
│  @anthropic-ai/claude-code@2.1.88   │
│                                      │
│  cli.js          (bundled binary)   │
│  cli.js.map      (source map)       │
│  package.json    (manifest)         │
│  README.md       (documentation)    │
│  vendor/         (native modules)   │
└─────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────┐
│         Installation                 │
│  npm install -g @anthropic-ai/      │
│    claude-code                      │
└─────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────┐
│         Runtime                      │
│  Node.js >= 18 or Bun               │
│  Single process, TTY required       │
│  (for interactive mode)             │
└─────────────────────────────────────┘
```

### 7.2 Native Modules

| Module | Platform | Purpose |
|--------|----------|---------|
| @img/sharp-darwin-arm64 | macOS ARM | Image processing |
| @img/sharp-darwin-x64 | macOS x64 | Image processing |
| @img/sharp-linux-arm | Linux ARM | Image processing |
| @img/sharp-linux-x64 | Linux x64 | Image processing |
| @img/sharp-win32-x64 | Windows x64 | Image processing |

---

## 8. Crosscutting Concepts

### 8.1 Feature Flags

| Flag | Domain | Purpose |
|------|--------|---------|
| COORDINATOR_MODE | Agent | Multi-agent coordination |
| KAIROS | Assistant | Assistant mode |
| BRIDGE_MODE | Remote | Remote control bridge |
| VOICE_MODE | Voice | Voice interaction |
| HISTORY_SNIP | Query | History snipping |
| CONTEXT_COLLAPSE | Query | Granular context collapsing |
| WORKFLOW_SCRIPTS | Infrastructure | Workflow automation |
| SSH_REMOTE | Remote | SSH-based remote |
| DIRECT_CONNECT | Remote | Direct session server |
| BUDDY | Buddy | Companion feature |
| WEB_BROWSER_TOOL | Tools | Web browsing capability |
| FORK_SUBAGENT | Agent | Sub-agent forking |
| PROACTIVE | Agent | Proactive assistance |
| AGENT_TRIGGERS | Agent | Agent trigger system |

### 8.2 Permission System

**Permission Modes:**
- `default` - Standard permission flow
- `plan` - Planning mode
- `acceptEdits` - Auto-accept edits
- `bypassPermissions` - Skip permission checks
- `dontAsk` - Never ask for permission
- `auto` - Automatic decisions
- `bubble` - Bubble up to parent

**Permission Rules:**
- `alwaysAllow` - Auto-approve
- `alwaysDeny` - Auto-reject
- `alwaysAsk` - Always prompt user

**Rule Sources:** user, project, local, policy, CLI, command, session

### 8.3 Context Compaction Strategies

| Strategy | Trigger | Description |
|----------|---------|-------------|
| Auto-compact | Context threshold | Automatic window management |
| Reactive compact | Prompt too long | Compact on error |
| Micro-compact | Cached editing | Granular compaction |
| Snip | Feature flag | History snipping |
| Context collapse | Feature flag | Granular collapsing |

### 8.4 Command Types

| Type | Description | Example |
|------|-------------|---------|
| `prompt` | Expands to prompt sent to model | Skills |
| `local` | Executes locally, returns text | /clear, /help |
| `local-jsx` | Renders React UI in terminal | /config, /model |

---

## 9. Architecture Decisions

See `/Documentation/adrs/` for detailed Architecture Decision Records organized by domain:

| Domain | ADRs |
|--------|------|
| Core AI/Query | ADR-0001 to ADR-0003 |
| Tools | ADR-0004 to ADR-0006 |
| Commands | ADR-0007 to ADR-0008 |
| UI/Terminal | ADR-0009 to ADR-0011 |
| State Management | ADR-0012 to ADR-0013 |
| Permissions/Security | ADR-0014 to ADR-0015 |
| Bridge/Remote | ADR-0016 to ADR-0017 |
| Agent/Swarm | ADR-0018 to ADR-0019 |
| Infrastructure | ADR-0020 to ADR-0021 |
| Configuration | ADR-0022 to ADR-0023 |

---

## 10. Quality Requirements

### 10.1 Performance

- **Startup time:** Minimized via lazy/dynamic imports and parallel prefetch
- **Streaming latency:** Async generator enables immediate response display
- **Memory:** Context compaction prevents unbounded growth

### 10.2 Security

- **Permission system:** Multi-layer with modes, rules, and classifiers
- **YOLO classifier:** Auto-approval of safe commands
- **Sandboxing:** VM context for REPL tool

### 10.3 Maintainability

- **TypeScript:** Full type coverage
- **Modular domains:** Clear separation of concerns
- **Feature flags:** Compile-time elimination of unused code

### 10.4 Extensibility

- **Plugin system:** Runtime plugin loading
- **Skill system:** Bundled and directory-based skills
- **Tool interface:** Polymorphic tool addition
- **MCP integration:** External tool servers

---

## 11. Risks and Technical Debt

| Risk | Impact | Mitigation |
|------|--------|------------|
| Source map extraction | Incomplete understanding | Cross-reference with runtime behavior |
| Zero dependencies in package.json | Hidden bundled deps | Runtime analysis required |
| Feature flag coupling | Compile-time decisions | Document all flags and their effects |
| Single binary | Debugging difficulty | Source maps, telemetry |
| Proprietary code | Legal considerations | This is analysis-only documentation |

---

## 12. Glossary

| Term | Definition |
|------|-----------|
| **Ink** | React-based terminal UI framework |
| **MCP** | Model Context Protocol |
| **LSP** | Language Server Protocol |
| **Query Loop** | Core async generator for API interaction |
| **Tool** | Executable capability exposed to the AI model |
| **Skill** | Pre-packaged prompt expansion or capability |
| **Bridge** | Remote control mechanism for web/mobile |
| **Swarm** | Multi-agent architecture |
| **Compaction** | Context window management strategy |
| **YOLO** | Auto-approval classifier for safe commands |
