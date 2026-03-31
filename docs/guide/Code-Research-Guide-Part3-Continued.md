# Claude Code Source Leak Research Guide - Part 3 (Continued)

> Part 3: Core Architecture Analysis (Continued)
> Deep analysis based on actual source code

---

### 3.3 Command System

The command system defines all commands that users can invoke via the slash (`/`) prefix.

#### 3.3.1 Command Interface Definition

**Core File:** `src/commands.ts` (754 lines, ~20KB)

**Command Type Definitions:**

```typescript
export type Command =
  | PromptCommand      // Prompt-based commands (LLM-driven)
  | BuiltInCommand     // Built-in commands (programmatic execution)
  | HybridCommand      // Hybrid commands

// Prompt Command - Uses LLM to execute tasks
export type PromptCommand = {
  type: 'prompt'
  name: string                    // Command name
  description: string             // Command description
  allowedTools: string[]          // Allowed tools list
  contentLength: number           // Prompt content length
  progressMessage: string         // Progress message
  source: 'builtin' | 'user'      // Command source
  getPromptForCommand: (
    args: string[],
    context: CommandContext
  ) => Promise<string>            // Generate prompt content
}

// Built-in Command - Direct code execution
export type BuiltInCommand = {
  type: 'builtin'
  name: string
  description: string
  fn: (
    args: string[],
    context: CommandContext
  ) => Promise<void> | void       // Command execution function
}

// Command Context
export type CommandContext = {
  cwd: string
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  messages: Message[]
  // ... other context info
}
```

#### 3.3.2 Command Implementation Patterns

**Pattern 1: PromptCommand**

Example: `/commit` command

File: `src/commands/commit.ts`

```typescript
const ALLOWED_TOOLS = [
  'Bash(git add:*)',
  'Bash(git status:*)',
  'Bash(git commit:*)',
]

function getPromptContent(): string {
  const { commit: commitAttribution } = getAttributionTexts()

  return `## Context

- Current git status: !\`git status\`
- Current git diff: !\`git diff HEAD\`
- Current branch: !\`git branch --show-current\`
- Recent commits: !\`git log --oneline -10\`

## Git Safety Protocol

- NEVER update the git config
- NEVER skip hooks (--no-verify)
- CRITICAL: ALWAYS create NEW commits
- Do not commit files with secrets
- If no changes, do not create empty commit
- Never use -i flag (interactive)

## Your task

Based on the above changes, create a single git commit:

1. Analyze all staged changes
2. Draft a commit message
3. Stage relevant files
4. Create commit using HEREDOC syntax
`
}

const command: PromptCommand = {
  type: 'prompt',
  name: 'commit',
  description: 'Create a git commit',
  allowedTools: ALLOWED_TOOLS,
  contentLength: 0, // Dynamic
  progressMessage: 'creating commit',
  source: 'builtin',

  async getPromptForCommand(_args, context) {
    const promptContent = getPromptContent()

    // Execute embedded shell commands
    const finalContent = await executeShellCommandsInPrompt(
      promptContent,
      {
        ...context,
        getAppState() {
          const appState = context.getAppState()
          return {
            ...appState,
            toolPermissionContext: {
              ...appState.toolPermissionContext,
              alwaysAllowRules: {
                ...appState.toolPermissionContext.alwaysAllowRules,
                command: ALLOWED_TOOLS,
              },
            },
          }
        },
      }
    )

    return finalContent
  }
}
```

**Workflow:**

```
User Input: /commit
    ↓
1. Parse command arguments (if any)
    ↓
2. Call getPromptForCommand()
    ├─ Generate prompt template
    ├─ Execute embedded shell commands (git status, git diff, etc.)
    ├─ Inject actual output into prompt
    └─ Return complete prompt
    ↓
3. Send prompt as user message to LLM
    ↓
4. LLM generates response
    ↓
5. Execute git commands suggested by LLM
    ├─ git add (based on LLM judgment)
    └─ git commit (using LLM-generated message)
    ↓
6. Display result to user
```

**Pattern 2: BuiltInCommand**

Example: `/config` command

```typescript
const command: BuiltInCommand = {
  type: 'builtin',
  name: 'config',
  description: 'Manage settings',
  fn: async (args, context) => {
    const subcommand = args[0]

    switch (subcommand) {
      case 'get':
        return configGet(args[1], context)
      case 'set':
        return configSet(args[1], args[2], context)
      case 'list':
        return configList(context)
      case 'unset':
        return configUnset(args[1], context)
      default:
        throw new Error(`Unknown subcommand: ${subcommand}`)
    }
  }
}
```

#### 3.3.3 Core Command Examples

**Example 1: `/commit` - Git Commit**

- **Type:** PromptCommand
- **Function:** Create git commit
- **Tool Restrictions:** Only git add/status/commit
- **Safety Mechanisms:** Git safety protocol, prevents accidental operations

**Example 2: `/review` - Code Review**

```typescript
const reviewCommand: PromptCommand = {
  type: 'prompt',
  name: 'review',
  description: 'Review recent changes',
  allowedTools: [
    'Bash(git diff:*)',
    'Bash(git log:*)',
    'FileRead(*)',
    'Grep(*)',
  ],
  progressMessage: 'reviewing code',
  // ...
}
```

**Example 3: `/mcp` - MCP Server Management**

```typescript
const mcpCommand: BuiltInCommand = {
  type: 'builtin',
  name: 'mcp',
  description: 'Manage MCP servers',
  fn: async (args, context) => {
    const subcommand = args[0]

    switch (subcommand) {
      case 'list':
        return listMcpServers(context)
      case 'add':
        return addMcpServer(args[1], context)
      case 'remove':
        return removeMcpServer(args[1], context)
      case 'restart':
        return restartMcpServer(args[1], context)
      default:
        return showMcpHelp()
    }
  }
}
```

**Example 4: `/compact` - Conversation Compaction**

- **Type:** PromptCommand
- **Function:** Compact conversation history to save tokens
- **Implementation:** Analyze messages, identify key info, generate summary

#### 3.3.4 Command Registry

**File:** `src/commands.ts`

**Unconditionally Loaded Commands:**

```typescript
// Core commands
export const coreCommands: Command[] = [
  addDir,
  autofixPr,
  backfillSessions,
  btw,
  clear,
  color,
  commit,
  commitPushPr,
  compact,
  config,
  context,
  cost,
  diff,
  ctx_viz,
  doctor,
  memory,
  help,
  ide,
  init,
  initVerifiers,
  keybindings,
  login,
  logout,
  issue,
  feedback,
  mcp,
  pr_comments,
  releaseNotes,
  rename,
  resume,
  review,
  ultrareview,
  session,
  share,
  skills,
  status,
  tasks,
  teleport,
  usage,
  theme,
  vim,
  // ... more
]
```

**Conditionally Loaded Commands:**

```typescript
// Ant-only commands
const antOnlyCommands = process.env.USER_TYPE === 'ant'
  ? [agentsPlatform]
  : []

// Feature-flagged commands
const proactiveCommand = feature('PROACTIVE') || feature('KAIROS')
  ? [proactive]
  : []

const briefCommand = feature('KAIROS') || feature('KAIROS_BRIEF')
  ? [briefCommand]
  : []

const assistantCommand = feature('KAIROS')
  ? [assistantCommand]
  : []

const bridgeCommand = feature('BRIDGE_MODE')
  ? [bridge]
  : []

const remoteControlServerCommand =
  feature('DAEMON') && feature('BRIDGE_MODE')
    ? [remoteControlServerCommand]
    : []

const voiceCommand = feature('VOICE_MODE')
  ? [voiceCommand]
  : []

// Export all commands
export const allCommands: Command[] = [
  ...coreCommands,
  ...antOnlyCommands,
  ...proactiveCommand,
  ...briefCommand,
  ...assistantCommand,
  ...bridgeCommand,
  ...remoteControlServerCommand,
  ...voiceCommand,
  // ... more
]
```

#### 3.3.5 Command System Design Patterns

**1. Command Pattern**
- Each command encapsulates an operation
- Unified execution interface
- Supports parameter passing

**2. Strategy Pattern**
- PromptCommand vs BuiltInCommand
- Different execution strategies
- Extensible command types

**3. Template Method Pattern**
- PromptCommand's getPromptForCommand
- Defines prompt generation skeleton
- Subcommands implement details

**4. Dependency Injection**
- CommandContext provides dependencies
- Commands don't access global state directly
- Facilitates testing and reuse

**5. Permission Control**
- allowedTools restricts tool usage
- Fine-grained permission control
- Prevents accidental operations

### 3.4 Permission System

The permission system is the security boundary of Claude Code, controlling execution permissions for all tools and commands.

#### 3.4.1 Permission Check Flow

**Core Files:**
- `src/hooks/toolPermission/` - Permission system implementation (1,386 lines)
- `src/hooks/useCanUseTool.ts` - Permission check hook

**Permission Check Layers:**

```
Tool Invocation Request
    ↓
┌─────────────────────────────────────┐
│  Layer 1: Global Permission Mode    │
│  - bypassPermissions → Allow        │
│  - auto → Enter auto-approval flow  │
│  - default → Enter user interaction │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  Layer 2: Tool-Specific Checks      │
│  - Tool's permissionsCheck function │
│  - Runtime validation (path, args)  │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  Layer 3: Rule Matching             │
│  - alwaysAllowRules                 │
│  - alwaysDenyRules                  │
│  - alwaysAskRules                   │
│  - Supports wildcards and prefixes  │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  Layer 4: Context Constraints      │
│  - Working directory boundaries    │
│  - Additional working directories  │
│  - Dangerous operation markers     │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  Layer 5: User Interaction          │
│  - Display permission prompt        │
│  - User chooses allow/deny          │
│  - Remember decision (cache)        │
└─────────────────────────────────────┘
    ↓
Return PermissionResult
```

#### 3.4.2 Permission Rule Matching

**Rule Types:**

```typescript
export type ToolPermissionRule = {
  toolName: string          // Tool name (supports wildcards)
  toolInputSummary?: string // Tool input summary (optional)
  permission: 'allow' | 'deny' | 'ask'
  reason?: string
}

export type ToolPermissionRulesBySource = {
  [sourceName: string]: ToolPermissionRule[]
}
```

**Rule Matching Examples:**

```typescript
// Rule 1: Allow all git operations
{
  toolName: 'Bash',
  toolInputSummary: 'git *',
  permission: 'allow',
  reason: 'Git operations are safe'
}

// Rule 2: Deny deleting node_modules
{
  toolName: 'Bash',
  toolInputSummary: 'rm -rf node_modules',
  permission: 'deny',
  reason: 'Use package manager instead'
}

// Rule 3: Ask for all write operations
{
  toolName: 'FileWrite',
  permission: 'ask',
  reason: 'Confirm file writes'
}

// Rule 4: Allow reading any file
{
  toolName: 'FileRead',
  toolInputSummary: '*',
  permission: 'allow'
}
```

**Matching Algorithm:**

```typescript
function matchesRule(
  toolCall: { toolName: string; input: unknown },
  rule: ToolPermissionRule
): boolean {
  // 1. Check tool name (supports wildcards)
  const nameMatches = matchWildcardPattern(
    rule.toolName,
    toolCall.toolName
  )

  if (!nameMatches) {
    return false
  }

  // 2. If rule has no inputSummary, it matches
  if (!rule.toolInputSummary) {
    return true
  }

  // 3. Check input summary (prefix match)
  const inputSummary = extractToolInputSummary(
    toolCall.toolName,
    toolCall.input
  )

  return inputSummary.startsWith(rule.toolInputSummary)
}
```

#### 3.4.3 User Interaction Handling

**Permission Prompt Component:**

File: `src/hooks/toolPermission/PermissionPrompt.tsx`

```typescript
export function PermissionPrompt({
  toolName,
  input,
  onAllow,
  onDeny,
  onAlwaysAllow,
  onAlwaysDeny,
}: PermissionPromptProps) {
  const tool = getToolByName(toolName)
  const inputSummary = formatToolInput(tool, input)

  return (
    <Box flexDirection="column">
      <Text bold>Tool Permission Request</Text>

      <Box marginTop={1}>
        <Text>{tool.metadata.description}</Text>
      </Box>

      <Box marginTop={1}>
        <Text dim>Input: </Text>
        <Text>{inputSummary}</Text>
      </Box>

      {/* Dangerous operation warning */}
      {isDangerousOperation(tool, input) && (
        <Box marginTop={1}>
          <Text color="red">⚠️ Dangerous operation detected!</Text>
        </Box>
      )}

      {/* Action buttons */}
      <Box marginTop={1} flexDirection="row">
        <TextButton onPress={onAllow}>Allow</TextButton>
        <Text dim> | </Text>
        <TextButton onPress={onDeny}>Deny</TextButton>
        <Text dim> | </Text>
        <TextButton onPress={onAlwaysAllow}>
          Always Allow
        </TextButton>
        <Text dim> | </Text>
        <TextButton onPress={onAlwaysDeny}>
          Always Deny
        </TextButton>
      </Box>

      {/* Keyboard shortcuts */}
      <Box marginTop={1}>
        <Text dim>
          Shortcuts: y=allow, n=deny, A=always allow, D=always deny
        </Text>
      </Box>
    </Box>
  )
}
```

**Permission Result Handling:**

```typescript
export type PermissionResult =
  | { allow: true }
  | {
      allow: false
      reason: string
      errorCode?: number
      requiresConfirmation?: boolean
    }

// Handle permission result
function handlePermissionResult(
  result: PermissionResult,
  context: ToolUseContext
): void {
  if (result.allow) {
    // Execute tool
    executeTool(context)
  } else {
    // Deny execution
    if (result.requiresConfirmation) {
      // Show confirmation prompt
      showConfirmationPrompt(result.reason)
    } else {
      // Show denial message
      showDenialMessage(result.reason)
    }
  }
}
```

#### 3.4.4 Sandbox Mechanism

**Sandbox Decision:**

File: `src/tools/BashTool/shouldUseSandbox.ts`

```typescript
export function shouldUseSandbox(
  command: string,
  context: ToolUseContext
): boolean {
  // 1. Check user config
  const userConfig = context.getAppState().sandboxConfig
  if (userConfig.mode === 'always') {
    return true
  }
  if (userConfig.mode === 'never') {
    return false
  }

  // 2. Check command type
  if (isReadOnlyCommand(command)) {
    return false  // Read-only commands don't need sandbox
  }

  if (isNetworkCommand(command)) {
    return true   // Network commands need sandbox
  }

  if (isSystemCommand(command)) {
    return true   // System commands need sandbox
  }

  // 3. Check path
  const cwd = getCwd()
  if (commandEscapesDirectory(command, cwd)) {
    return true  // Escaping commands need sandbox
  }

  // 4. Default behavior
  return userConfig.mode === 'auto'
}
```

**Sandbox Execution:**

```typescript
class SandboxManager {
  private containerId?: string
  private tempDir?: string

  async execute(command: string): Promise<ExecResult> {
    // 1. Create temporary directory
    this.tempDir = await mkdtemp(join(tmpdir(), 'claude-sandbox-'))

    // 2. Start container (if using Docker)
    if (this.useDocker) {
      this.containerId = await this.startDockerContainer()
    }

    // 3. Execute command in isolated environment
    const result = await this.executeInIsolation(command)

    // 4. Cleanup
    await this.cleanup()

    return result
  }

  private async executeInIsolation(
    command: string
  ): Promise<ExecResult> {
    if (this.useDocker) {
      return this.executeInDocker(command)
    } else if (this.useFirejail) {
      return this.executeInFirejail(command)
    } else {
      // Use chroot isolation
      return this.executeInChroot(command)
    }
  }
}
```

#### 3.4.5 Permission System Design Patterns

**1. Onion Model (Layered Security)**
- Multiple permission check layers
- Each layer validates independently
- Fail-fast returns

**2. Rule Engine**
- Declarative permission rules
- Wildcard matching
- Priority handling

**3. Confirmation Pattern**
- Double confirmation for dangerous operations
- Remember user choices
- Keyboard shortcut support

**4. Sandbox Pattern**
- Isolated execution environment
- Restricted resource access
- Automatic cleanup

**5. Audit Logging**
- Record all permission decisions
- Support post-hoc review
- Configurable verbosity

### 3.5 Bridge System

The bridge system implements bidirectional communication between IDE extensions (VS Code, JetBrains) and the Claude Code CLI.

#### 3.5.1 Bridge Architecture

**Core Files:**
- `src/bridge/bridgeMain.ts` - Bridge main loop
- `src/bridge/bridgeMessaging.ts` - Message protocol
- `src/bridge/jwtUtils.ts` - JWT authentication
- `src/bridge/sessionRunner.ts` - Session execution management

**Architecture Diagram:**

```
┌─────────────────┐
│   IDE Extension │
│  (VS Code/Jet)  │
└────────┬────────┘
         │ WebSocket / HTTP
         ↓
┌─────────────────────────────────┐
│     Bridge Server               │
│  - API Endpoints                │
│  - WebSocket Connection         │
│  - JWT Authentication           │
└────────┬────────────────────────┘
         │
         ↓
┌─────────────────────────────────┐
│     Claude Code CLI             │
│  - QueryEngine                  │
│  - Tool Execution               │
│  - Message Streaming            │
└─────────────────────────────────┘
```

**Core Types:**

```typescript
export type BridgeConfig = {
  apiUrl: string              // Bridge API URL
  wsUrl: string               // WebSocket URL
  bridgeId: string            // Bridge ID
  authToken: string           // JWT auth token
  pollIntervalMs: number      // Poll interval
  sessionTimeoutMs: number    // Session timeout
}

export type BridgeSession = {
  sessionId: string           // Unique session ID
  status: 'pending' | 'running' | 'completed' | 'failed'
  messages: Message[]         // Session messages
  createdAt: number           // Creation timestamp
  updatedAt: number           // Last update timestamp
}

export type BridgeMessage =
  | { type: 'user_message'; content: string }
  | { type: 'assistant_message'; content: string }
  | { type: 'tool_use'; tool: string; input: unknown }
  | { type: 'tool_result'; result: unknown }
  | { type: 'status'; status: string }
  | { type: 'error'; error: string }
```

#### 3.5.2 Message Protocol

**Message Format:**

```typescript
// IDE → CLI
export type IDEToCLIMessage =
  | {
      type: 'start_session'
      sessionId: string
      initialMessage: string
      options: SessionOptions
    }
  | {
      type: 'send_message'
      sessionId: string
      content: string
    }
  | {
      type: 'interrupt'
      sessionId: string
    }
  | {
      type: 'stop_session'
      sessionId: string
    }

// CLI → IDE
export type CLIToIDEMessage =
  | {
      type: 'session_started'
      sessionId: string
    }
  | {
      type: 'message_delta'
      sessionId: string
      delta: {
        type: 'text' | 'tool_use' | 'tool_result'
        content: string
      }
    }
  | {
      type: 'message_complete'
      sessionId: string
      message: Message
    }
  | {
      type: 'session_complete'
      sessionId: string
      finalState: SessionState
    }
  | {
      type: 'error'
      sessionId: string
      error: {
        code: string
        message: string
      }
    }
```

**Message Processing:**

```typescript
class BridgeMessageHandler {
  async handleMessage(
    message: IDEToCLIMessage,
    context: BridgeContext
  ): Promise<CLIToIDEMessage> {
    switch (message.type) {
      case 'start_session':
        return await this.handleStartSession(message, context)

      case 'send_message':
        return await this.handleSendMessage(message, context)

      case 'interrupt':
        return await this.handleInterrupt(message, context)

      case 'stop_session':
        return await this.handleStopSession(message, context)

      default:
        throw new Error(`Unknown message type: ${message.type}`)
    }
  }

  private async handleStartSession(
    message: IDEToCLIMessage & { type: 'start_session' },
    context: BridgeContext
  ): Promise<CLIToIDEMessage> {
    // 1. Create session
    const session = await this.createSession(message.sessionId)

    // 2. Initialize QueryEngine
    const engine = new QueryEngine({
      ...message.options,
      initialMessages: [],
    })

    // 3. Send initial message
    const result = await engine.query(message.initialMessage)

    // 4. Return session started confirmation
    return {
      type: 'session_started',
      sessionId: message.sessionId,
    }
  }

  private async handleSendMessage(
    message: IDEToCLIMessage & { type: 'send_message' },
    context: BridgeContext
  ): Promise<CLIToIDEMessage> {
    // 1. Get session
    const session = context.getSession(message.sessionId)

    // 2. Send message to QueryEngine
    const response = await session.engine.query(message.content)

    // 3. Return response (streaming)
    return {
      type: 'message_complete',
      sessionId: message.sessionId,
      message: response.finalMessage,
    }
  }
}
```

#### 3.5.3 JWT Authentication

**JWT Token Generation:**

File: `src/bridge/jwtUtils.ts`

```typescript
import { sign } from 'jsonwebtoken'

export function generateBridgeToken(
  bridgeId: string,
  userId: string,
  secret: string
): string {
  const payload = {
    bridgeId,
    userId,
    iat: Math.floor(Date.now() / 1000),
    exp: Math.floor(Date.now() / 1000) + (60 * 60), // 1 hour
    nonce: randomUUID(),
  }

  return sign(payload, secret, {
    algorithm: 'HS256',
  })
}

export function verifyBridgeToken(
  token: string,
  secret: string
): { valid: boolean; payload?: BridgeTokenPayload } {
  try {
    const payload = verify(token, secret, {
      algorithms: ['HS256'],
    })

    return {
      valid: true,
      payload: payload as BridgeTokenPayload,
    }
  } catch (error) {
    return {
      valid: false,
    }
  }
}
```

**Token Refresh:**

```typescript
class TokenRefreshScheduler {
  private refreshTimer?: NodeJS.Timeout

  startRefreshScheduler(
    token: string,
    refreshIntervalMs: number
  ): void {
    this.stopRefreshScheduler()

    this.refreshTimer = setInterval(async () => {
      try {
        // Refresh 5 minutes before expiration
        const payload = decode(token)
        const expiresAt = payload.exp * 1000
        const now = Date.now()

        if (expiresAt - now < 5 * 60 * 1000) {
          await this.refreshToken(token)
        }
      } catch (error) {
        console.error('Token refresh failed:', error)
      }
    }, refreshIntervalMs)
  }

  stopRefreshScheduler(): void {
    if (this.refreshTimer) {
      clearInterval(this.refreshTimer)
      this.refreshTimer = undefined
    }
  }

  private async refreshToken(oldToken: string): Promise<string> {
    // Call refresh API
    const response = await fetch('/api/bridge/refresh', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${oldToken}`,
      },
    })

    const { token } = await response.json()
    return token
  }
}
```

#### 3.5.4 IDE Integration

**VS Code Extension Integration:**

```typescript
// VS Code extension side code (conceptual)

class ClaudeCodeBridge {
  private ws: WebSocket
  private sessionId: string

  async connect(apiUrl: string, authToken: string): Promise<void> {
    this.ws = new WebSocket(`${apiUrl}/ws?token=${authToken}`)

    this.ws.onmessage = (event) => {
      const message = JSON.parse(event.data)
      this.handleMessage(message)
    }
  }

  async startSession(
    initialMessage: string
  ): Promise<string> {
    const response = await fetch('/api/sessions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.authToken}`,
      },
      body: JSON.stringify({
        initialMessage,
        options: this.getSessionOptions(),
      }),
    })

    const { sessionId } = await response.json()
    this.sessionId = sessionId

    return sessionId
  }

  async sendMessage(content: string): Promise<void> {
    await fetch(`/api/sessions/${this.sessionId}/messages`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.authToken}`,
      },
      body: JSON.stringify({ content }),
    })
  }

  private handleMessage(message: CLIToIDEMessage): void {
    switch (message.type) {
      case 'message_delta':
        this.showStreamingResponse(message.delta)
        break

      case 'message_complete':
        this.finalizeResponse(message.message)
        break

      case 'session_complete':
        this.handleSessionComplete(message.finalState)
        break

      case 'error':
        this.showError(message.error)
        break
    }
  }

  private showStreamingResponse(delta: MessageDelta): void {
    // Display streaming response in VS Code
    this.outputChannel.append(delta.content)
  }

  private finalizeResponse(message: Message): void {
    // Finalize response display
    this.updateVirtualDocument(message)
  }
}
```

**JetBrains Plugin Integration:**

```typescript
// JetBrains plugin side code (conceptual)

class ClaudeCodeBridgeService : Service {
  private val client = HttpClient()

  fun startSession(initialMessage: String): String {
    val request = HttpRequest.newBuilder()
      .uri(URI.create("$apiUrl/api/sessions"))
      .header("Content-Type", "application/json")
      .header("Authorization", "Bearer $authToken")
      .POST(HttpRequest.BodyPublishers.ofString(
        JSONObject()
          .put("initialMessage", initialMessage)
          .put("options", getSessionOptions())
          .toString()
      ))
      .build()

    val response = client.send(request, BodyHandlers.ofString())
    val json = JSONObject(response.body())
    return json.getString("sessionId")
  }

  fun sendMessage(sessionId: String, content: String) {
    val request = HttpRequest.newBuilder()
      .uri(URI.create("$apiUrl/api/sessions/$sessionId/messages"))
      .header("Content-Type", "application/json")
      .header("Authorization", "Bearer $authToken")
      .POST(HttpRequest.BodyPublishers.ofString(
        JSONObject()
          .put("content", content)
          .toString()
      ))
      .build()

    client.send(request, BodyHandlers.ofString())
  }
}
```

#### 3.5.5 Bridge System Design Patterns

**1. Client-Server Pattern**
- IDE as client
- Bridge Server as intermediary
- CLI as backend service

**2. Message Passing Pattern**
- Asynchronous message communication
- Type-safe message protocol
- Streaming message support

**3. Session Management**
- Session lifecycle management
- Session isolation
- Timeout handling

**4. Authentication & Authorization**
- JWT token authentication
- Token refresh mechanism
- Permission checking

**5. Observer Pattern**
- IDE subscribes to session events
- Real-time message push
- WebSocket long connection

**6. Adapter Pattern**
- Unified Bridge API
- Adapt to different IDEs
- Protocol conversion

---

## Part 3 Completion Summary

### Completed Core System Analysis

✅ **3.1 Tool System**
- Tool interface definition
- Permission model implementation
- Core tool examples
- Tool registry
- 6 design patterns

✅ **3.2 QueryEngine**
- Core architecture
- Query loop flow
- Streaming response handling
- Tool call coordination
- Cost tracking and budget control
- Error handling and retry
- Thinking Mode
- 6 design patterns

✅ **3.3 Command System**
- Command interface definition
- Command implementation patterns (PromptCommand vs BuiltInCommand)
- Core command examples (/commit, /review, /mcp)
- Command registry
- 5 design patterns

✅ **3.4 Permission System**
- Permission check flow (5-layer checks)
- Permission rule matching
- User interaction handling
- Sandbox mechanism
- 5 design patterns

✅ **3.5 Bridge System**
- Bridge architecture
- Message protocol
- JWT authentication
- IDE integration
- 6 design patterns

### Statistics

| System | Core Files | Lines of Code | Design Patterns |
|--------|-----------|---------------|-----------------|
| Tool System | src/Tool.ts, src/tools.ts | ~2,500 | 6 |
| QueryEngine | src/QueryEngine.ts | ~1,300 | 6 |
| Command System | src/commands.ts | ~750 | 5 |
| Permission System | src/hooks/toolPermission/ | ~1,400 | 5 |
| Bridge System | src/bridge/ | ~2,000 | 6 |
| **Total** | **5 Core Systems** | **~8,000 Lines** | **28 Patterns** |

---

**Document Status:** ✅ Part 3 Complete!
**Next Preview:** Part 4: Key Technical Topics
**Last Updated:** 2026-03-31

---

*Continuing...*
