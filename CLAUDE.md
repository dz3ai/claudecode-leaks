# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **security research archive**, not a development environment. It contains a snapshot of Claude Code's source code that was publicly exposed on March 31, 2026 through a source map file leak in the npm distribution. The repository exists for educational study, security research, and software supply-chain analysis.

**Important context:**
- This is read-only research material extracted from an npm source map (`cli.js.map`, 60MB)
- No build process, package management, or development workflow exists
- The original code remains the property of Anthropic
- This is not affiliated with, endorsed by, or maintained by Anthropic

## Architecture Overview

Claude Code is a CLI tool built with:
- **Runtime**: Bun (JavaScript runtime)
- **Language**: TypeScript (strict mode)
- **Terminal UI**: React + Ink (terminal rendering)
- **CLI Parser**: Commander.js
- **Schema Validation**: Zod v4

The codebase follows a **tool-based agent architecture** where Claude interacts with the user's development environment through declarative tools (file operations, shell commands, web search, etc.).

## High-Level Structure

### Core Systems

**Tool System** (`src/tools/`, `src/tools.ts`, `src/Tool.ts`)
- Every capability (Bash, FileRead, Agent spawning) is implemented as a self-contained Tool
- Tools define: input schema (Zod), permission model, execution logic, progress state
- ~43 tool implementations, loaded conditionally via feature flags
- Tools are registered in `src/tools.ts` with dead-code elimination via `bun:bundle` feature flags

**Command System** (`src/commands/`, `src/commands.ts`)
- User-facing slash commands invoked with `/` prefix
- ~110 command implementations (`/commit`, `/review`, `/mcp`, `/config`, etc.)
- Commands use Commander.js for parsing and React/Ink for UI

**Query Engine** (`src/QueryEngine.ts` - 46KB, ~1,400 lines)
- Core engine for LLM API calls
- Handles streaming responses, tool-call loops, thinking mode, retry logic
- Single most complex file in the codebase

**Permission System** (`src/hooks/toolPermission/`)
- Checks permissions on every tool invocation
- Prompts user for approval/denial or auto-resolves based on permission mode
- Critical security boundary

### Key Directories

| Directory | Purpose |
|-----------|---------|
| `src/tools/` | Tool implementations (Bash, File ops, Agent, MCP, etc.) |
| `src/commands/` | Slash command implementations |
| `src/components/` | Ink UI components (~140 components) |
| `src/services/` | External service integrations (API, OAuth, MCP, LSP, analytics) |
| `src/bridge/` | IDE/remote bridge (bidirectional communication layer) |
| `src/coordinator/` | Multi-agent orchestration |
| `src/skills/` | Skill system (reusable workflows) |
| `src/plugins/` | Plugin loader |
| `src/screens/` | Full-screen UIs (Doctor, REPL, Resume) |
| `src/state/` | State management |
| `src/memdir/` | Persistent memory directory |
| `src/tasks/` | Task management |
| `src/vim/` | Vim mode implementation |
| `src/voice/` | Voice input |
| `src/remote/` | Remote sessions |
| `src/server/` | Server mode |

### Entry Points

**`src/main.tsx`** (786KB, ~24,000 lines)
- Application entry point
- Commander.js CLI parser setup
- React/Ink renderer initialization
- Performance optimization: parallel prefetch of MDM settings, keychain, API preconnect

**`src/QueryEngine.ts`** - Core LLM interaction engine

**`src/Tool.ts`** (29KB) - Base types and interfaces for all tools

**`src/commands.ts`** (25KB) - Command registry

## Architectural Patterns

### Feature Flags (Dead Code Elimination)

Bun's `bun:bundle` feature flags strip inactive code at build time:

```typescript
import { feature } from 'bun:bundle'

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

Notable flags: `PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `AGENT_TRIGGERS`, `MONITOR_TOOL`

### Tool Registration Pattern

Tools are conditionally imported based on feature flags and environment variables:

```typescript
// Ant-only tools
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool
  : null

// Feature-gated tools
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

### Service Layer Architecture

Each external service has a dedicated module under `src/services/`:
- `api/` - Anthropic API client, file API, bootstrap
- `mcp/` - Model Context Protocol server connection
- `oauth/` - OAuth 2.0 authentication flow
- `lsp/` - Language Server Protocol manager
- `analytics/` - GrowthBook-based feature flags

### Bridge System

`src/bridge/` provides bidirectional communication for:
- IDE extensions (VS Code, JetBrains)
- Remote control
- Session execution management

Key files:
- `bridgeMain.ts` - Bridge main loop
- `bridgeMessaging.ts` - Message protocol
- `jwtUtils.ts` - JWT-based authentication

## Tech Stack Details

| Category | Technology |
|----------|------------|
| Runtime | [Bun](https://bun.sh) |
| Language | TypeScript (strict) |
| Terminal UI | [React](https://react.dev) + [Ink](https://github.com/vadimdemedes/ink) |
| CLI Parsing | [Commander.js](https://github.com/tj/commander.js) |
| Schema Validation | [Zod v4](https://zod.dev) |
| Code Search | [ripgrep](https://github.com/BurntSushi/ripgrep) |
| Protocols | [MCP SDK](https://modelcontextprotocol.io), LSP |
| API | [Anthropic SDK](https://docs.anthropic.com) |
| Telemetry | OpenTelemetry + gRPC |
| Feature Flags | GrowthBook |
| Auth | OAuth 2.0, JWT, macOS Keychain |

## Key Files for Understanding

To understand the architecture quickly:
1. **Start with**: `src/Tool.ts` - Understand tool interface and permission model
2. **Then**: `src/QueryEngine.ts` - Core LLM interaction loop
3. **Pick a tool**: `src/tools/BashTool/BashTool.ts` - Simple tool example
4. **Pick a command**: `src/commands/commit.ts` - Command implementation example

## Scale

- **~1,906 files** in `src/` directory
- **~512,000+ lines of code**
- **~43 tools**
- **~110 commands**
- **~140 UI components**
- **Extracted from**: 60MB source map file (`cli.js.map`)

## Research Notes

This source snapshot reveals:
- Complete tool implementation details
- Permission system architecture
- Security mechanisms (sandboxing, approval flows)
- Integration patterns (MCP, LSP, OAuth)
- Telemetry and monitoring approach
- Multi-agent coordination patterns
- Plugin and skill system design

The codebase demonstrates production patterns for building agentic CLI tools with React-based terminal UIs, comprehensive permission systems, and extensible architecture.

## Disclaimer

This repository exists for **educational and security research purposes only**. It is not a development environment, and the code should not be modified or redistributed as an official Anthropic product.
