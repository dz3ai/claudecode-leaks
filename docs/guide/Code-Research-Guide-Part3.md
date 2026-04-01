# Claude Code Source Leak Research Guide - Part 3

> Part 3: Core Architecture Analysis
> Deep analysis based on actual source code

---

## 📑 Table of Contents

### 3.1 Tool System
- [3.1.1 Tool Interface Definition](#311-tool-interface-definition) - Core types and interfaces
- [3.1.2 Tool Implementation Pattern](#312-tool-implementation-pattern) - Tool definition structure
- [3.1.3 Permission Model Implementation](#313-permission-model-implementation) - Permission check flow
- [3.1.4 Core Tool Implementation Cases](#314-core-tool-implementation-cases) - BashTool, FileReadTool, AgentTool
- [3.1.5 Tool Registry](#315-tool-registry) - Tool loading and dead code elimination
- [3.1.6 Tool System Design Patterns Summary](#316-tool-system-design-patterns-summary)

### 3.2 Query Engine
- [3.2.1 Core Architecture](#321-core-architecture) - QueryEngine class structure and configuration
- [3.2.2 Query Loop Flow](#322-query-loop-flow) - Complete LLM interaction loop
- [3.2.3 Streaming Response Processing](#323-streaming-response-processing) - SSE event handling and real-time UI updates
- [3.2.4 Tool Call Coordination](#324-tool-call-coordination) - Tool execution and permission checks
- [3.2.5 Cost Tracking and Budget Control](#325-cost-tracking-and-budget-control) - Usage statistics and budget checks
- [3.2.6 Error Handling and Retry](#326-error-handling-and-retry) - Error categorization and retry logic
- [3.2.7 Thinking Mode](#327-thinking-mode) - Extended thinking functionality
- [3.2.8 Query Engine Design Patterns Summary](#328-query-engine-design-patterns-summary)

> 📌 **Continue reading:** [Part 3 (Continued)](./Code-Research-Guide-Part3-Continued.md) - 3.3 Command System, 3.4 Permission System, 3.5 Bridge System

---

## Part 3: Core Architecture Analysis

### 3.1 Tool System

The Tool System is the core abstraction of Claude Code, defining all capabilities for AI interaction with the development environment.

#### 3.1.1 Tool Interface Definition

**Core File:** `src/Tool.ts` (792 lines)

**Core Type Definitions:**

```typescript
// Tool Input Schema - using Zod v4 for type-safe schema definition
export type ToolInputJSONSchema = {
  [x: string]: unknown
  type: 'object'
  properties?: {
    [x: string]: unknown
  }
}

// Tool Use Context - contains all environment info needed for tool execution
export type ToolUseContext = {
  options: {
    commands: Command[]           // Available commands list
    debug: boolean                // Debug mode
    mainLoopModel: string         // Main loop model
    tools: Tools                  // Available tools collection
    verbose: boolean              // Verbose output
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]    // MCP server connections
    mcpResources: Record<string, ServerResource[]>
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
    customSystemPrompt?: string   // Custom system prompt
    appendSystemPrompt?: string   // Append system prompt
    querySource?: QuerySource
    refreshTools?: () => Tools    // Refresh tools callback
  }
  abortController: AbortController         // Abort controller
  readFileState: FileStateCache            // File read cache
  getAppState(): AppState                  // Get app state
  setAppState(f: (prev: AppState) => AppState): void
  setAppStateForTasks?: (f: (prev: AppState) => AppState): void
  handleElicitation?: (...) => Promise<ElicitResult>
  setToolJSX?: SetToolJSXFn                // Set tool JSX UI
  addNotification?: (notif: Notification) => void
  appendSystemMessage?: (msg: SystemMessage) => void
  sendOSNotification?: (opts: {...}) => void
  requestPrompt?: (...) => Promise<PromptResponse>
  messages: Message[]
  fileReadingLimits?: {...}
  globLimits?: {...}
  toolDecisions?: Map<...>
  queryTracking?: QueryChainTracking
  // ... more fields
}
```

**Tool Permission Context:**

```typescript
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode                    // Permission mode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource    // Always allow rules
  alwaysDenyRules: ToolPermissionRulesBySource     // Always deny rules
  alwaysAskRules: ToolPermissionRulesBySource      // Always ask rules
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean   // Avoid permission prompts
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}>
```

**Permission Modes:**

```typescript
export type PermissionMode =
  | 'default'      // Default: ask for every tool invocation
  | 'auto'         // Auto: auto-approve all tool invocations
  | 'plan'         // Plan mode: for planning phase
  | 'bypassPermissions'  // Bypass: no permission checks
  | 'foreground'   // Foreground: allow foreground tool calls
  | 'background'   // Background: allow background tool calls
```

#### 3.1.2 Tool Implementation Pattern

**Tool Definition Structure:**

Each tool follows a unified structure definition:

```typescript
// Example: simplified BashTool structure
const BashTool = buildTool({
  // Tool metadata
  name: BASH_TOOL_NAME,  // 'Bash'
  metadata: {
    description: 'Execute shell commands',
    uiMetadata: {
      title: 'Bash',
      category: 'system',
      icon: 'terminal',
    }
  },

  // Input validation schema (using Zod v4)
  inputSchema: lazySchema(() =>
    z.object({
      command: z.string().describe('The command to execute'),
      timeout_ms: z.number().optional(),
      background: z.boolean().optional(),
      // ... more parameters
    })
  ),

  // Permission check function
  permissionsCheck: async (
    input: ParsedInput<typeof inputSchema>,
    context: ToolUseContext
  ): Promise<PermissionResult> => {
    // Check if command is in allow list
    // Check if path is within working directory
    // Check if dangerous operation
    // Return PermissionResult
  },

  // Tool execution function
  fn: async (
    input: ParsedInput<typeof inputSchema>,
    context: ToolUseContext,
    progress?: ToolCallProgress<BashProgress>
  ): Promise<ToolResult<string>> => {
    // 1. Parse command
    // 2. Check sandbox requirements
    // 3. Execute command
    // 4. Process output
    // 5. Return result
  },

  // Optional: UI render function
  renderToolResult?: (result: string) => React.ReactNode
})
```

**Core Interface:**

```typescript
export interface Tool<
  TInput extends z.ZodTypeAny = z.ZodTypeAny,
  TData = unknown,
  TProgress extends ToolProgressData = ToolProgressData
> {
  readonly name: string
  readonly aliases?: readonly string[]      // Tool aliases
  readonly metadata?: ToolMetadata
  readonly inputSchema: TInput              // Zod schema
  readonly permissionsCheck?: PermissionsCheckFn<TInput>
  readonly fn: ToolFn<TInput, TData, TProgress>
  readonly renderToolResult?: (result: TData) => React.ReactNode
  readonly shouldSuppressSilentToolErrors?: boolean
  readonly experimental?: boolean
  readonly dangerous?: boolean              // Mark as dangerous tool
}
```

#### 3.1.3 Permission Model Implementation

**Permission Check Flow:**

```
Tool Invocation Request
    ↓
Check Permission Mode (PermissionMode)
    ↓
    ├─ bypassPermissions → Execute directly
    ├─ auto → Check alwaysAllowRules
    ├─ default → Enter user interaction flow
    └─ plan → Plan mode special handling
    ↓
Apply tool-specific rules (permissionsCheck)
    ↓
    ├─ alwaysAllowRules → Approve
    ├─ alwaysDenyRules → Deny
    └─ alwaysAskRules → Ask user
    ↓
Check working directory constraints
    ↓
Check dangerous operation mode
    ↓
Return PermissionResult (allow/deny)
```

**Permission Check Example (BashTool):**

File: `src/tools/BashTool/bashPermissions.ts`

```typescript
export function bashToolHasPermission(
  input: { command: string; ... },
  context: ToolUseContext
): PermissionResult {
  // 1. Check if command matches always-allow rules
  const alwaysAllow = context.permissionContext.alwaysAllowRules
  if (matchesRule(input.command, alwaysAllow)) {
    return { allow: true }
  }

  // 2. Check if command matches always-deny rules
  const alwaysDeny = context.permissionContext.alwaysDenyRules
  if (matchesRule(input.command, alwaysDeny)) {
    return {
      allow: false,
      reason: 'Command matches deny rule',
      errorCode: 403
    }
  }

  // 3. Check if attempting to access files outside working directory
  if (commandEscapesWorkingDirectory(input.command)) {
    return {
      allow: false,
      reason: 'Command attempts to access files outside working directory',
      errorCode: 403,
      requiresConfirmation: true
    }
  }

  // 4. Check if dangerous operation (delete, format, etc.)
  if (isDangerousCommand(input.command)) {
    return {
      allow: false,
      reason: 'Dangerous operation detected',
      errorCode: 403,
      requiresConfirmation: true
    }
  }

  // 5. Default behavior depends on permission mode
  switch (context.permissionContext.mode) {
    case 'auto':
      return { allow: true }
    case 'default':
      return { allow: false, requiresConfirmation: true }
    case 'bypassPermissions':
      return { allow: true }
    default:
      return { allow: false, requiresConfirmation: true }
  }
}
```

**Permission Rule Types:**

```typescript
export type ToolPermissionRulesBySource = {
  [sourceName: string]: ToolPermissionRule[]
}

export type ToolPermissionRule = {
  toolName: string          // Tool name (supports wildcards)
  toolInputSummary?: string // Tool input summary
  permission: 'allow' | 'deny' | 'ask'
  reason?: string
}
```

#### 3.1.4 Core Tool Implementation Examples

**Example 1: BashTool - Shell Command Execution**

File: `src/tools/BashTool/BashTool.tsx`

**Key Features:**
- Command parsing and security checks
- Sandbox execution (via `shouldUseSandbox()` check)
- Output truncation and preview generation
- Background task support
- Git operation tracking

**Core Logic Flow:**

```typescript
// 1. Command parsing and security check
const parsedCommand = parseForSecurity(input.command)

// 2. Check if sandbox is needed
const useSandbox = shouldUseSandbox(input.command, context)

// 3. Execute command
const result = await exec(input.command, {
  timeout: input.timeout_ms || getDefaultTimeoutMs(),
  cwd: getCwd(),
  env: process.env,
  sandbox: useSandbox ? new SandboxManager() : undefined
})

// 4. Process output
if (result.stdout.length > PREVIEW_SIZE_BYTES) {
  // Large output written to file
  const outputPath = getToolResultPath()
  await writeTextContent(outputPath, result.stdout)
  return {
    data: `Output too large. See ${outputPath}`,
    newMessages: [...]
  }
}

// 5. Return result
return {
  data: result.stdout,
  newMessages: [
    {
      type: 'assistant',
      content: renderToolResultMessage(result)
    }
  ]
}
```

**Example 2: FileReadTool - File Reading**

File: `src/tools/FileReadTool/FileReadTool.tsx`

**Key Features:**
- Supports multiple file formats (text, images, PDF, Jupyter notebooks)
- Caching mechanism (FileStateCache)
- LRU cache eviction
- Encoding detection (UTF-8, Latin-1, ASCII)
- Line ending detection

**Input Schema:**

```typescript
inputSchema: z.object({
  path: z.string().describe('Path to the file to read'),
  limit: z.number().optional().describe('Limit lines to read'),
  offset: z.number().optional().describe('Start from line'),
  pages: z.string().optional().describe('For PDFs: page ranges'),
  return_format: z.enum(['markdown', 'text']).optional()
})
```

**Example 3: AgentTool - Sub-agent Creation**

File: `src/tools/AgentTool/AgentTool.tsx`

**Key Features:**
- Dynamic sub-agent creation
- Independent context and state
- Communication mechanism with main agent
- Result aggregation

**Sub-agent Creation Flow:**

```typescript
// 1. Create sub-agent context
const subagentContext = createSubagentContext(parentContext, {
  agentId: generateAgentId(),
  messages: [],
  tools: filteredTools,  // Only tools available to sub-agent
  taskBudget: { total: input.maxTokens || 100000 }
})

// 2. Initialize sub-agent's QueryEngine
const subagentEngine = new QueryEngine(subagentContext)

// 3. Execute sub-agent task
const result = await subagentEngine.query(input.prompt)

// 4. Aggregate results
return {
  data: result.finalMessage,
  newMessages: result.messages
}
```

#### 3.1.5 Tool Registry

**File:** `src/tools.ts` (17KB, ~450 lines)

**Tool Registration Mechanism:**

```typescript
import { feature } from 'bun:bundle'
import { toolMatchesName, type Tool, type Tools } from './Tool.js'
import { AgentTool } from './tools/AgentTool/AgentTool.js'
import { BashTool } from './tools/BashTool/BashTool.js'
import { FileEditTool } from './tools/FileEditTool/FileEditTool.js'
// ... more imports

// Core (unconditionally loaded)
const coreTools: Tools = [
  BashTool,
  FileReadTool,
  FileWriteTool,
  FileEditTool,
  GlobTool,
  GrepTool,
  NotebookEditTool,
  WebFetchTool,
  AgentTool,
  SkillTool,
  TaskCreateTool,
  TaskUpdateTool,
  // ... more core tools
]

// Ant-only tools (conditionally loaded)
const antOnlyTools = process.env.USER_TYPE === 'ant'
  ? [
      REPLTool,
      SuggestBackgroundPRTool,
    ]
  : []

// Feature-flagged tools
const proactiveTools = (feature('PROACTIVE') || feature('KAIROS'))
  ? [SleepTool]
  : []

const cronTools = feature('AGENT_TRIGGERS')
  ? [CronCreateTool, CronDeleteTool, CronListTool]
  : []

const monitorTool = feature('MONITOR_TOOL')
  ? [MonitorTool]
  : []

// Export all tools
export const allTools: Tools = [
  ...coreTools,
  ...antOnlyTools,
  ...proactiveTools,
  ...cronTools,
  ...monitorTool,
]
```

**Dead Code Elimination:**

Using Bun's `bun:bundle` feature flags, inactive feature code is completely removed at build time:

```typescript
import { feature } from 'bun:bundle'

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null  // If VOICE_MODE not enabled, entire Voice module is not bundled
```

#### 3.1.6 Tool System Design Pattern Summary

**1. Unified Interface Pattern**
- All tools implement the same `Tool` interface
- Unified input validation (Zod schema)
- Unified permission check interface
- Unified progress reporting interface

**2. Composition Over Inheritance**
- Tools gain capabilities through composing `ToolUseContext`
- Not through inheriting from base classes
- Flexible feature composition

**3. Declarative Configuration**
- Tool metadata (name, description, category, icon)
- Input schema (Zod)
- Permission rules (declarative rule matching)

**4. Type Safety**
- Complete TypeScript type definitions
- Zod runtime type validation
- Compile-time and runtime dual guarantees

**5. Progressive Enhancement**
- Core tools unconditionally loaded
- Optional tools controlled by feature flags
- Ant-only tools controlled by environment variables

**6. Separation of Concerns**
- Tool logic is independent
- UI rendering is optional
- Permission checking is pluggable

### 3.2 Query Engine

The Query Engine is the core of Claude Code, responsible for LLM interaction loops, streaming response handling, and tool call coordination.

#### 3.2.1 Core Architecture

**File:** `src/QueryEngine.ts` (1,295 lines, ~40KB)

**Class Structure:**

```typescript
export class QueryEngine {
  private config: QueryEngineConfig
  private abortController: AbortController
  private messages: Message[]
  private usage: NonNullableUsage = EMPTY_USAGE
  private totalCost: number = 0
  private toolUseIDs: Set<string> = new Set()
  private queryChainTracking?: QueryChainTracking

  constructor(config: QueryEngineConfig) {
    this.config = config
    this.abortController = createAbortController()
    this.messages = config.initialMessages || []
    this.initialize()
  }

  // Main query entry point
  async query(userInput: string): Promise<QueryResult> {
    // Complete query loop
  }

  // Private methods
  private async initialize(): Promise<void> { ... }
  private async processStream(stream: AsyncIterable<...>): Promise<void> { ... }
  private async handleToolUse(toolUse: ToolUseBlock): Promise<void> { ... }
  private async callLLM(messages: Message[]): Promise<AsyncIterable<...>> { ... }
  private updateUsage(usage: Usage): void { ... }
  private shouldStop(): boolean { ... }
}
```

**Configuration Type:**

```typescript
export type QueryEngineConfig = {
  cwd: string                              // Working directory
  tools: Tools                             // Available tools
  commands: Command[]                      // Available commands
  mcpClients: MCPServerConnection[]        // MCP servers
  agents: AgentDefinition[]                // Sub-agent definitions
  canUseTool: CanUseToolFn                 // Permission check function
  getAppState: () => AppState              // Get app state
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]              // Initial messages
  readFileCache: FileStateCache            // File read cache
  customSystemPrompt?: string              // Custom system prompt
  appendSystemPrompt?: string              // Append system prompt
  userSpecifiedModel?: string              // User-specified model
  fallbackModel?: string                   // Fallback model
  thinkingConfig?: ThinkingConfig          // Thinking mode config
  maxTurns?: number                        // Maximum turns
  maxBudgetUsd?: number                    // Maximum budget (USD)
  taskBudget?: { total: number }           // Task budget (tokens)
  jsonSchema?: Record<string, unknown>     // JSON schema (structured output)
  verbose?: boolean                        // Verbose mode
}
```

#### 3.2.2 Query Loop Flow

```
User Input (userInput)
    ↓
1. Initialize (initialize)
   ├─ Load system prompt
   ├─ Load memory (loadMemoryPrompt)
   ├─ Initialize MCP clients
   └─ Build initial message list
    ↓
2. LLM Call Loop (query loop)
   │
   ├─→ 2.1 Prepare Messages
   │   ├─ Add user message
   │   ├─ Apply system prompt
   │   ├─ Add tool definitions
   │   └─ Serialize to API format
   │
   ├─→ 2.2 Call LLM API (callLLM)
   │   ├─ Select model
   │   ├─ Send request
   │   └─ Return streaming response
   │
   ├─→ 2.3 Process Streaming Response (processStream)
   │   ├─ Receive content_block_delta events
   │   ├─ Accumulate text content
   │   ├─ Update UI in real-time
   │   └─ Detect tool_use content blocks
   │
   ├─→ 2.4 Handle Tool Call (handleToolUse)
   │   │
   │   ├─ Parse tool name and parameters
   │   ├─ Find tool definition
   │   ├─ Permission check (canUseTool)
   │   │   ├─ Pass → Execute tool
   │   │   └─ Deny → Return error
   │   │
   │   ├─ Execute Tool (tool.fn)
   │   │   ├─ Pass ToolUseContext
   │   │   ├─ Pass progress callback
   │   │   └─ Wait for result
   │   │
   │   ├─ Process Tool Result
   │   │   ├─ Add to message list
   │   │   ├─ Update UI
   │   │   └─ Continue loop
   │   │
   │   └─ Special Tool Handling
   │       ├─ AgentTool → Create sub-agent
   │       ├─ SleepTool → Delay execution
   │       └─ TaskCreateTool → Task management
   │
   ├─→ 2.5 Update Usage (updateUsage)
   │   ├─ Accumulate token counts
   │   ├─ Calculate cost
   │   └─ Check budget limits
   │
   └─→ 2.6 Check Stop Conditions
       ├─ Reached maxTurns?
       ├─ Exceeded maxBudgetUsd?
       ├─ Received stop signal?
       ├─ User interrupt?
       └─ Detected completion signal?
    ↓
3. Return Result
   ├─ Final message list
   ├─ Total usage statistics
   └─ Total cost
```

#### 3.2.3 Streaming Response Handling

**Streaming Event Types:**

```typescript
type StreamEvent =
  | { type: 'message_start'; message: {...} }
  | { type: 'content_block_start'; index: number; content_block: {...} }
  | { type: 'content_block_delta'; index: number; delta: {...} }
  | { type: 'content_block_stop'; index: number }
  | { type: 'message_delta'; delta: {...}; usage: {...} }
  | { type: 'message_stop' }
```

**Streaming Processing Logic:**

```typescript
private async processStream(
  stream: AsyncIterable<StreamEvent>
): Promise<void> {
  let currentContentBlock: ContentBlock | null = null
  let accumulatedText = ''

  for await (const event of stream) {
    switch (event.type) {
      case 'message_start':
        // Message starts
        this.handleMessageStart(event.message)
        break

      case 'content_block_start':
        // Content block starts (text or tool call)
        currentContentBlock = event.content_block
        if (currentContentBlock.type === 'text') {
          accumulatedText = ''
        }
        break

      case 'content_block_delta':
        // Content delta (streaming text fragments)
        if (currentContentBlock?.type === 'text') {
          const textDelta = event.delta.text
          accumulatedText += textDelta

          // Real-time UI update
          this.setToolJSX?.({
            jsx: <StreamingText text={accumulatedText} />,
            shouldHidePromptInput: true,
            shouldContinueAnimation: true
          })

          // Update response length counter
          this.setResponseLength?.(prev => prev + textDelta.length)
        }
        break

      case 'content_block_stop':
        // Content block ends
        if (currentContentBlock?.type === 'text') {
          // Complete text content block
          this.messages.push({
            role: 'assistant',
            content: accumulatedText
          })
        } else if (currentContentBlock?.type === 'tool_use') {
          // Tool call content block
          await this.handleToolUse(currentContentBlock)
        }
        currentContentBlock = null
        break

      case 'message_delta':
        // Message metadata update (usage, etc.)
        if (event.usage) {
          this.updateUsage(event.usage)
        }
        break

      case 'message_stop':
        // Message ends
        return
    }
  }
}
```

#### 3.2.4 Tool Call Coordination

**Tool Call Flow:**

```typescript
private async handleToolUse(
  toolUse: ToolUseBlockParam
): Promise<void> {
  const toolName = toolUse.name
  const toolInput = toolUse.input
  const toolUseID = toolUse.id

  // 1. Find tool
  const tool = this.config.tools.find(t =>
    toolMatchesName(t, toolName)
  )

  if (!tool) {
    throw new Error(`Tool not found: ${toolName}`)
  }

  // 2. Permission check
  const permissionResult = await this.config.canUseTool(
    tool,
    toolInput,
    this.buildToolUseContext()
  )

  if (!permissionResult.allow) {
    // Permission denied
    this.messages.push({
      role: 'assistant',
      content: `[Tool ${toolName} was denied: ${permissionResult.reason}]`
    })
    return
  }

  // 3. Parse input (using Zod schema)
  const parsedInput = tool.inputSchema.parse(toolInput)

  // 4. Execute tool
  let toolResult: ToolResult<unknown>

  try {
    // Show tool execution UI
    this.setToolJSX?.({
      jsx: <ToolExecutionProgress toolName={toolName} />,
      shouldHidePromptInput: true,
      showSpinner: true
    })

    // Call tool function
    toolResult = await tool.fn(parsedInput, this.buildToolUseContext(), (progress) => {
      // Progress callback
      this.setToolJSX?.({
        jsx: <ToolProgress toolName={toolName} progress={progress} />,
        shouldHidePromptInput: true
      })
    })

    // Handle new messages returned by tool
    if (toolResult.newMessages) {
      this.messages.push(...toolResult.newMessages)
    }

    // Add tool result message
    this.messages.push({
      role: 'assistant',
      content: [{
        type: 'tool_result',
        tool_use_id: toolUseID,
        content: JSON.stringify(toolResult.data)
      }]
    })

  } catch (error) {
    // Tool execution error
    this.messages.push({
      role: 'assistant',
      content: `[Tool ${toolName} failed: ${error.message}]`
    })
  }

  // 5. Continue query loop
  // Tool results become context for next LLM call
}
```

#### 3.2.5 Cost Tracking and Budget Control

**Usage Type:**

```typescript
export type NonNullableUsage = {
  input_tokens: number
  output_tokens: number
  cache_creation_input_tokens?: number
  cache_read_input_tokens?: number
}
```

**Cost Calculation:**

```typescript
private updateUsage(usage: Usage): void {
  // Accumulate usage
  this.usage = accumulateUsage(this.usage, usage)

  // Calculate cost
  const modelCost = getModelCost(this.config.mainLoopModel)
  const turnCost = calculateCost(usage, modelCost)
  this.totalCost += turnCost

  // Check budget limits
  if (this.config.maxBudgetUsd && this.totalCost > this.config.maxBudgetUsd) {
    throw new Error(
      `Budget exceeded: ${this.totalCost.toFixed(2)} > ${this.config.maxBudgetUsd}`
    )
  }

  // Check task budget
  if (this.config.taskBudget) {
    const totalTokens = this.usage.input_tokens + this.usage.output_tokens
    if (totalTokens > this.config.taskBudget.total) {
      throw new Error(
        `Task token budget exceeded: ${totalTokens} > ${this.config.taskBudget.total}`
      )
    }
  }

  // Update UI display
  this.setAppState?.(prev => ({
    ...prev,
    totalCost: this.totalCost,
    totalTokens: this.usage.input_tokens + this.usage.output_tokens
  }))
}
```

#### 3.2.6 Error Handling and Retry

**Error Classification:**

```typescript
function categorizeRetryableAPIError(error: unknown): {
  retryable: boolean
  retryAfterMs?: number
} {
  if (error instanceof APIError) {
    switch (error.status) {
      case 429: // Rate Limit
        return {
          retryable: true,
          retryAfterMs: error.retryAfter || 5000
        }
      case 500: // Internal Server Error
      case 502: // Bad Gateway
      case 503: // Service Unavailable
        return {
          retryable: true,
          retryAfterMs: 1000
        }
      case 400: // Bad Request
        return { retryable: false }
      default:
        return { retryable: false }
    }
  }
  return { retryable: false }
}
```

**Retry Logic:**

```typescript
private async callLLMWithRetry(
  messages: Message[]
): Promise<AsyncIterable<StreamEvent>> {
  const maxRetries = 3
  let attempt = 0

  while (attempt < maxRetries) {
    try {
      return await this.callLLM(messages)
    } catch (error) {
      const { retryable, retryAfterMs } = categorizeRetryableAPIError(error)

      if (!retryable) {
        throw error  // Non-retryable, throw directly
      }

      attempt++

      if (attempt >= maxRetries) {
        throw new Error(`Max retries exceeded: ${error.message}`)
      }

      // Wait before retry
      await sleep(retryAfterMs || 1000 * attempt)
    }
  }

  throw new Error('Should not reach here')
}
```

#### 3.2.7 Thinking Mode

**Thinking Configuration:**

```typescript
export type ThinkingConfig = {
  enabled: boolean              // Whether thinking mode is enabled
  budgetTokens: number          // Thinking budget (tokens)
  thinkingInstructions?: string // Thinking instructions
}
```

**Thinking Processing:**

```typescript
private async processThinkingBlock(
  thinkingText: string
): Promise<void> {
  // Thinking content is not displayed to user
  // But is counted in token usage

  // Validate thinking budget
  const thinkingTokens = estimateTokens(thinkingText)
  if (thinkingTokens > this.config.thinkingConfig?.budgetTokens) {
    throw new Error('Thinking budget exceeded')
  }

  // Thinking content can be used for debugging and analysis
  if (this.config.verbose) {
    console.log('[Thinking]', thinkingText)
  }

  // Thinking content is not added to user-visible message list
  // But is added to internal logs
}
```

#### 3.2.8 Query Engine Design Pattern Summary

**1. State Machine Pattern**
- Query loop is a state machine
- States: Initialize → LLM Call → Stream Processing → Tool Call → Complete
- Each state has clear entry and exit conditions

**2. Iterator Pattern**
- Streaming response is AsyncIterable
- Event-by-event processing, non-blocking
- Supports real-time UI updates

**3. Strategy Pattern**
- Different permission modes have different handling strategies
- Different tools have different execution strategies
- Strategies are pluggable

**4. Observer Pattern**
- Progress callbacks
- UI update callbacks
- State change listeners

**5. Chain of Responsibility**
- Permission check chain
- Error handling chain
- Message processing chain

**6. Decorator Pattern**
- ToolUseContext decorates tool execution
- Logging decorators
- Caching decorators

---

**Document Status:** 🚧 In Progress (3.1 and 3.2 completed)
**Expected Completion:** 2026-04-01
**Next Preview:** 3.3 Command System, 3.4 Permission System, 3.5 Bridge System

---

*Continuing...*
