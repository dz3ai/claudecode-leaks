# Part 6: Code Reading Practice

## Overview

This part provides 4 complete code tracing cases to help you deeply understand how Claude Code works in practice. Each case starts from user input and traces the code execution path, showing how key data structures change.

**Learning Objectives:**
- Understand end-to-end request processing flow
- Master core system interactions
- Learn to independently trace code execution paths
- Identify key design patterns and architectural decisions

**How to Use This Part:**
1. Read each case overview first
2. Follow the code tracing steps
3. Reference actual source code files
4. Verify in your own environment
5. Experiment with modifications

---

## Case 1: From User Input to Tool Execution

### Scenario Overview

User inputs in Claude Code: "Read the package.json file"

This case traces how this request is:
1. Parsed and understood
2. Converted to tool calls
3. Executed with results returned

### Complete Execution Flow

```
User Input
  ↓
[1] QueryInput component captures input
  ↓
[2] QueryEngine.query() called
  ↓
[3] LLM API request constructed
  ↓
[4] Streaming response received
  ↓
[5] Tool call detected (FileRead)
  ↓
[6] Permission check
  ↓
[7] FileRead tool executes
  ↓
[8] Result returned to LLM
  ↓
[9] LLM generates final response
  ↓
[10] ResponseStream displays result
```

### Detailed Code Trace

**Step 1: User Input Capture**

File: `src/components/QueryInput.tsx`

```typescript
// User presses Enter key
useInput((input, key) => {
  if (key.return) {
    const query = inputState.current

    // Trigger query
    props.onSubmit(query)

    // Clear input
    setInputState('')
  }
})
```

**Data Structure:**
```typescript
{
  query: "Read the package.json file",
  timestamp: 1698765432000,
  sessionId: "session-abc-123"
}
```

---

**Step 2: QueryEngine Receives Query**

File: `src/QueryEngine.ts` (lines 45-120)

```typescript
async query(request: QueryRequest): Promise<QueryResponse> {
  // 2.1 Build message history
  const messages = this.buildMessages(request)

  // 2.2 Prepare available tools
  const tools = this.prepareTools()

  // 2.3 Call LLM API
  const response = await this.callLLM(messages, tools)

  // 2.4 Process streaming response
  return this.processStream(response)
}

private buildMessages(request: QueryRequest): Message[] {
  return [
    {
      role: 'system',
      content: this.systemPrompt
    },
    {
      role: 'user',
      content: request.query
    }
  ]
}
```

**Key Data Structures:**
```typescript
// QueryRequest
interface QueryRequest {
  query: string
  sessionId: string
  context?: Record<string, any>
}

// Message
interface Message {
  role: 'system' | 'user' | 'assistant'
  content: string
  toolCalls?: ToolCall[]
  toolResult?: any
}
```

---

**Step 3: LLM API Call**

File: `src/services/api/client.ts` (lines 89-156)

```typescript
async callLLM(
  messages: Message[],
  tools: Tool[]
): Promise<AsyncIterable<LLMChunk>> {
  // 3.1 Build API request
  const request = {
    model: 'claude-sonnet-4-6',
    max_tokens: 8192,
    messages: messages.map(m => ({
      role: m.role,
      content: m.content
    })),
    tools: this.formatTools(tools),  // Convert to Anthropic format
    stream: true
  }

  // 3.2 Send request
  const response = await fetch(this.apiURL, {
    method: 'POST',
    headers: {
      'x-api-key': this.apiKey,
      'anthropic-version': '2023-06-01',
      'content-type': 'application/json'
    },
    body: JSON.stringify(request)
  })

  // 3.3 Return streaming response
  return this.parseStream(response.body)
}
```

**Tool Formatting:**
```typescript
// Tool definition sent to LLM in this format
{
  name: 'Read',
  description: 'Read a file from the filesystem',
  input_schema: {
    type: 'object',
    properties: {
      file_path: {
        type: 'string',
        description: 'Path to the file to read'
      }
    },
    required: ['file_path']
  }
}
```

---

**Step 4: Streaming Response Processing**

File: `src/QueryEngine.ts` (lines 234-289)

```typescript
private async processStream(
  stream: AsyncIterable<LLMChunk>
): Promise<QueryResponse> {
  let fullContent = ''
  const toolCalls: ToolCall[] = []

  for await (const chunk of stream) {
    // 4.1 Process content chunk
    if (chunk.type === 'content_block_delta') {
      const token = chunk.delta.text
      fullContent += token

      // 4.2 Trigger UI update event
      this.emit('token', token)
    }

    // 4.3 Process tool call chunk
    if (chunk.type === 'content_block_start') {
      if (chunk.content_block.type === 'tool_use') {
        const toolCall = {
          id: chunk.content_block.id,
          name: chunk.content_block.name,
          input: JSON.parse(chunk.content_block.input)
        }
        toolCalls.push(toolCall)
      }
    }
  }

  return {
    content: fullContent,
    toolCalls: toolCalls
  }
}
```

**Streaming Response Example:**
```typescript
// Chunk 1: Content start
{
  type: 'content_block_start',
  index: 0,
  content_block: { type: 'text', text: '' }
}

// Chunk 2: Text token
{
  type: 'content_block_delta',
  delta: { type: 'text_delta', text: 'I' }
}

// Chunk 3: More text
{
  type: 'content_block_delta',
  delta: { type: 'text_delta', text: "'ll read the package.json file for you.' }
}

// Chunk 4: Tool call start
{
  type: 'content_block_start',
  index: 1,
  content_block: {
    type: 'tool_use',
    id: 'toolu_abc123',
    name: 'Read',
    input: '{"file_path":"package.json"}'
  }
}
```

---

**Step 5: Tool Call Detection**

File: `src/QueryEngine.ts` (lines 291-334)

```typescript
private async handleToolCalls(
  toolCalls: ToolCall[]
): Promise<ToolResult[]> {
  const results: ToolResult[] = []

  for (const toolCall of toolCalls) {
    // 5.1 Find tool implementation
    const tool = this.tools.get(toolCall.name)
    if (!tool) {
      results.push({
        toolCallId: toolCall.id,
        error: `Tool not found: ${toolCall.name}`
      })
      continue
    }

    // 5.2 Execute tool (with permission check)
    const result = await this.executeTool(tool, toolCall)
    results.push(result)
  }

  return results
}
```

---

**Step 6: Permission Check**

File: `src/hooks/toolPermission/index.ts` (lines 45-167)

```typescript
async checkPermission(
  tool: Tool,
  input: any
): Promise<PermissionResult> {
  // 6.1 Get permission mode
  const mode = this.getPermissionMode()

  if (mode === 'auto') {
    return { allowed: true, mode: 'auto' }
  }

  // 6.2 Check tool-specific rules
  const toolRules = this.rules.get(tool.name)
  if (toolRules) {
    const ruleResult = this.evaluateRules(toolRules, input)
    if (!ruleResult.allowed) {
      return ruleResult
    }
  }

  // 6.3 Request user approval (if needed)
  if (mode === 'manual' || mode === 'explain') {
    return this.requestUserApproval(tool, input)
  }

  return { allowed: true, mode }
}

private async requestUserApproval(
  tool: Tool,
  input: any
): Promise<PermissionResult> {
  // Display permission prompt UI
  const approval = await this.ui.prompt({
    type: 'confirm',
    message: `Allow ${tool.name}?`,
    detail: JSON.stringify(input, null, 2)
  })

  if (approval) {
    // Save to allowlist
    this.rememberApproval(tool.name, input)
    return { allowed: true, mode: 'manual' }
  }

  return { allowed: false, mode: 'manual', denied: true }
}
```

**Permission Prompt UI:**
```
┌─────────────────────────────────────────┐
│ Allow Read tool to execute?             │
│                                         │
│ Tool: Read                              │
│ Input:                                  │
│ {                                      │
│   "file_path": "package.json"          │
│ }                                      │
│                                         │
│ [A] Allow always  [O] Allow once       │
│ [D] Deny        [?] View details       │
└─────────────────────────────────────────┘
```

---

**Step 7: FileRead Tool Execution**

File: `src/tools/FileRead/FileRead.ts` (lines 23-98)

```typescript
async execute(
  input: FileReadInput,
  context: ExecutionContext
): Promise<FileReadResult> {
  // 7.1 Validate input
  const schema = z.object({
    file_path: z.string()
  })

  const validated = schema.parse(input)

  // 7.2 Check file exists
  const fullPath = path.resolve(validated.file_path)
  if (!fs.existsSync(fullPath)) {
    throw new Error(`File not found: ${validated.file_path}`)
  }

  // 7.3 Read file content
  const content = fs.readFileSync(fullPath, 'utf-8')

  // 7.4 Report progress (if file is large)
  if (context.onProgress) {
    context.onProgress({
      status: 'reading',
      message: `Read ${content.length} bytes`
    })
  }

  // 7.5 Return result
  return {
    success: true,
    file_path: validated.file_path,
    content: content,
    size: content.length
  }
}
```

**Execution Result:**
```typescript
{
  toolCallId: 'toolu_abc123',
  result: {
    success: true,
    file_path: 'package.json',
    content: '{\n  "name": "claude-code",\n  "version": "2.1.88",\n...',
    size: 1234
  }
}
```

---

**Step 8: Result Return to LLM**

File: `src/QueryEngine.ts` (lines 336-378)

```typescript
private async continueWithToolResults(
  messages: Message[],
  toolResults: ToolResult[]
): Promise<QueryResponse> {
  // 8.1 Add tool results to message history
  const assistantMessage = {
    role: 'assistant' as const,
    content: '',
    toolCalls: toolResults.map(r => r.toolCall)
  }

  const userMessage = {
    role: 'user' as const,
    content: '',
    toolResults: toolResults
  }

  const newMessages = [
    ...messages,
    assistantMessage,
    userMessage
  ]

  // 8.2 Call LLM again (with tool results)
  const response = await this.callLLM(newMessages, this.tools)

  // 8.3 Process final response
  return this.processStream(response)
}
```

---

**Step 9: LLM Generates Final Response**

After receiving FileRead results, LLM generates natural language response:

```
I've read the package.json file. Here's what it contains:

[displays file content summary]

The project is Claude Code version 2.1.88, and it has dependencies on...
```

---

**Step 10: Result Display**

File: `src/components/ResponseStream.tsx` (lines 34-89)

```typescript
const ResponseStream = ({ queryEngine }) => {
  const [tokens, setTokens] = useState([])
  const [complete, setComplete] = useState(false)

  useEffect(() => {
    // Subscribe to token events
    const onToken = (token) => {
      setTokens(prev => [...prev, token])
    }

    // Subscribe to complete events
    const onComplete = () => {
      setComplete(true)
    }

    queryEngine.on('token', onToken)
    queryEngine.on('complete', onComplete)

    return () => {
      queryEngine.off('token', onToken)
      queryEngine.off('complete', onComplete)
    }
  }, [queryEngine])

  return (
    <Box>
      <Text>{tokens.join('')}</Text>
      {!complete && <Text dimColor>▊</Text>}
    </Box>
  )
}
```

**UI Display:**
```
I'll read the package.json file for you.▊
[Reading package.json...]
I've read the package.json file. Here are the contents:

{
  "name": "claude-code",
  "version": "2.1.88",
  ...
}

The file shows this is Claude Code version 2.1.88...
```

---

### Key Takeaways

**Data Flow:**
1. User input → QueryInput component
2. QueryEngine builds messages and tool list
3. LLM API returns streaming response + tool calls
4. Tool calls go through permission checks
5. FileRead executes and returns results
6. Tool results sent back to LLM
7. LLM generates final natural language response
8. ResponseStream component displays results

**Key Files:**
- `src/components/QueryInput.tsx` - Input capture
- `src/QueryEngine.ts` - Query coordination
- `src/services/api/client.ts` - LLM API communication
- `src/hooks/toolPermission/index.ts` - Permission checks
- `src/tools/FileRead/FileRead.ts` - File reading tool
- `src/components/ResponseStream.tsx` - Result display

**Design Patterns:**
- **Mediator Pattern**: QueryEngine coordinates all components
- **Observer Pattern**: Event-driven UI updates
- **Strategy Pattern**: Different permission modes
- **Chain of Responsibility**: Multi-layer permission checks

---

## Case 2: Permission Check Flow

### Scenario Overview

User attempts: `bash rm -rf node_modules`

This is a high-risk operation requiring detailed permission checks. This case shows how the permission system:
1. Detects high-risk operations
2. Applies multi-layer security checks
3. Requests user approval
4. Records decisions

### Complete Permission Check Flow

```
Tool Call Request
  ↓
[1] Tool Metadata Check
  ↓
[2] Global Permission Mode Evaluation
  ↓
[3] Tool-Specific Rule Matching
  ↓
[4] Context Constraint Validation
  ↓
[5] User Interaction and Approval
  ↓
[6] Decision Caching
  ↓
Approve or Deny
```

### Detailed Code Trace

**Step 1: Tool Metadata Check**

File: `src/Tool.ts` (lines 67-134)

```typescript
interface ToolMetadata {
  // Tool risk level
  risk: 'safe' | 'low' | 'medium' | 'high' | 'critical'

  // Requires explicit approval
  requiresExplicitApproval: boolean

  // Allowed path patterns
  allowedPaths?: string[]

  // Dangerous parameter detection
  dangerousParams?: {
    param: string
    pattern: RegExp
    riskLevel: 'high' | 'critical'
  }[]
}

// BashTool metadata
const BashToolMetadata: ToolMetadata = {
  risk: 'high',
  requiresExplicitApproval: true,
  dangerousParams: [
    {
      param: 'command',
      pattern: /rm\s+-rf/,
      riskLevel: 'critical'
    },
    {
      param: 'command',
      pattern: /rm\s+-fr/,
      riskLevel: 'critical'
    }
  ]
}
```

---

**Step 2: Global Permission Mode Evaluation**

File: `src/hooks/toolPermission/PermissionManager.ts` (lines 89-145)

```typescript
enum PermissionMode {
  Auto = 'auto',        // Auto-approve all
  Manual = 'manual',    // Prompt every time
  Explain = 'explain',  // Explain then prompt
  Locked = 'locked'     // Deny all
}

async evaluateGlobalMode(
  tool: Tool,
  input: any
): Promise<PermissionDecision> {
  const mode = this.config.permissionMode

  // Mode 1: Locked - Direct denial
  if (mode === PermissionMode.Locked) {
    return {
      allowed: false,
      reason: 'Permission mode is locked',
      mode: 'locked'
    }
  }

  // Mode 2: Auto - Quick pass (but still danger check)
  if (mode === PermissionMode.Auto) {
    const dangerCheck = this.checkDangerLevel(tool, input)
    if (dangerCheck.isCritical) {
      // Even in auto mode, critical ops need confirmation
      return this.requestApproval(tool, input, dangerCheck)
    }
    return { allowed: true, mode: 'auto' }
  }

  // Mode 3: Manual / Explain - Needs user interaction
  return this.requestApproval(tool, input)
}
```

---

**Step 3: Tool-Specific Rule Matching**

File: `src/hooks/toolPermission/RuleMatcher.ts` (lines 45-123)

```typescript
interface PermissionRule {
  id: string
  toolName: string
  condition: RuleCondition
  action: 'allow' | 'deny' | 'prompt'
  reason?: string
}

interface RuleCondition {
  // Parameter matching
  inputMatches?: {
    param: string
    pattern: RegExp
  }

  // Path constraints
  pathConstraints?: {
    allowed: string[]
    denied: string[]
  }

  // Time constraints
  timeConstraints?: {
    allowedHours: [number, number][]
  }
}

// Example rule: Prevent deleting outside node_modules
const nodeModulesRule: PermissionRule = {
  id: 'protect-node-modules',
  toolName: 'Bash',
  condition: {
    inputMatches: {
      param: 'command',
      pattern: /rm\s+-rf\s+(?!node_modules)/
    }
  },
  action: 'deny',
  reason: 'Cannot use rm -rf outside node_modules for safety'
}

async matchRules(
  tool: Tool,
  input: any
): Promise<RuleMatchResult> {
  const rules = this.rules.filter(r => r.toolName === tool.name)

  for (const rule of rules) {
    if (this.evaluateCondition(rule.condition, input)) {
      return {
        matched: true,
        rule: rule,
        action: rule.action
      }
    }
  }

  return { matched: false }
}
```

---

**Step 4: Context Constraint Validation**

File: `src/hooks/toolPermission/ContextValidator.ts` (lines 34-98)

```typescript
async validateContext(
  tool: Tool,
  input: any,
  context: ExecutionContext
): Promise<ContextValidationResult> {
  const violations: ContextViolation[] = []

  // 4.1 Working directory constraint
  const workingDir = context.workingDir
  if (input.file_path || input.command) {
    const targetPath = this.extractPath(input)

    // Check if in allowed directory
    if (!this.isPathAllowed(targetPath, workingDir)) {
      violations.push({
        type: 'path_violation',
        message: `Path ${targetPath} is outside working directory ${workingDir}`
      })
    }
  }

  // 4.2 Resource constraint
  if (tool.metadata.resourceIntensive) {
    const availableMemory = this.getAvailableMemory()
    if (availableMemory < 1024 * 1024 * 1024) {  // 1GB
      violations.push({
        type: 'resource_constraint',
        message: 'Insufficient memory for this operation'
      })
    }
  }

  // 4.3 Frequency constraint
  const recentCalls = this.getRecentToolCalls(tool.name, 60)  // 60 seconds
  if (recentCalls.length > 10) {
    violations.push({
      type: 'rate_limit',
      message: `Tool ${tool.name} called ${recentCalls.length} times in the last minute`
    })
  }

  return {
    allowed: violations.length === 0,
    violations
  }
}
```

---

**Step 5: User Interaction and Approval**

File: `src/components/PermissionPrompt.tsx` (lines 56-178)

```typescript
const PermissionPrompt = ({
  tool,
  input,
  onDecision
}: PermissionPromptProps) => {
  // Analyze risk
  const riskAnalysis = analyzeRisk(tool, input)

  return (
    <Box flexDirection="column">
      {/* Risk Warning */}
      {riskAnalysis.level === 'critical' && (
        <Box backgroundColor="red" padding={1}>
          <Text color="white" bold>
            ⚠️  CRITICAL RISK OPERATION
          </Text>
        </Box>
      )}

      {/* Tool Info */}
      <Box>
        <Text bold>Tool: </Text>
        <Text>{tool.name}</Text>
      </Box>

      {/* Command Preview */}
      <Box marginTop={1}>
        <Text dimColor>Command:</Text>
        <Text color="red"> {input.command}</Text>
      </Box>

      {/* Risk Analysis */}
      <Box marginTop={1}>
        <Text dimColor>Risk Level: </Text>
        <Text color={riskColor(riskAnalysis.level)}>
          {riskAnalysis.level.toUpperCase()}
        </Text>
      </Box>

      {/* Specific Risks */}
      {riskAnalysis.concerns.map(concern => (
        <Box key={concern.type} marginLeft={2}>
          <Text color="yellow">• {concern.message}</Text>
        </Box>
      ))}

      {/* Action Buttons */}
      <Box marginTop={1}>
        <Text dimColor>[A] Allow  [D] Deny  [?] More info</Text>
      </Box>
    </Box>
  )
}
```

**UI Display:**
```
┌──────────────────────────────────────────────┐
│ ⚠️  CRITICAL RISK OPERATION                   │
│                                              │
│ Tool: Bash                                   │
│                                              │
│ Command: rm -rf node_modules                 │
│                                              │
│ Risk Level: CRITICAL                         │
│                                              │
│ • This command will recursively delete files │
│ • Operation affects: node_modules/           │
│ • Estimated size: 500 MB                     │
│                                              │
│ [A] Allow  [D] Deny  [?] More info          │
└──────────────────────────────────────────────┘
```

---

**Step 6: Decision Caching**

File: `src/hooks/toolPermission/DecisionCache.ts` (lines 23-76)

```typescript
class DecisionCache {
  private decisions: Map<string, CachedDecision> = new Map()

  // Generate decision key
  private decisionKey(tool: Tool, input: any): string {
    return `${tool.name}:${JSON.stringify(input)}`
  }

  // Check cache
  check(tool: Tool, input: any): CachedDecision | null {
    const key = this.decisionKey(tool, input)
    const cached = this.decisions.get(key)

    if (!cached) {
      return null
    }

    // Check if expired
    const age = Date.now() - cached.timestamp
    if (age > this.config.cacheTimeout) {
      this.decisions.delete(key)
      return null
    }

    return cached
  }

  // Save decision
  save(
    tool: Tool,
    input: any,
    decision: 'allow' | 'deny',
    scope: 'once' | 'session' | 'always'
  ): void {
    const key = this.decisionKey(tool, input)

    this.decisions.set(key, {
      tool: tool.name,
      input,
      decision,
      scope,
      timestamp: Date.now()
    })

    // Persist to disk (if scope is 'always')
    if (scope === 'always') {
      this.persistToDisk()
    }
  }
}
```

---

### Key Takeaways

**Multi-Layer Security Checks:**
1. **Tool Metadata** - Risk level and dangerous parameter detection
2. **Global Mode** - Auto/Manual/Explain/Locked
3. **Rule Matching** - Custom allow/deny rules
4. **Context Validation** - Path, resource, frequency constraints
5. **User Interaction** - Explicit approval flow
6. **Decision Caching** - Remember user choices

**Key Files:**
- `src/Tool.ts` - Tool metadata definitions
- `src/hooks/toolPermission/PermissionManager.ts` - Global modes
- `src/hooks/toolPermission/RuleMatcher.ts` - Rule engine
- `src/hooks/toolPermission/ContextValidator.ts` - Context validation
- `src/components/PermissionPrompt.tsx` - User interface
- `src/hooks/toolPermission/DecisionCache.ts` - Decision caching

**Design Patterns:**
- **Chain of Responsibility** - Sequential check layers
- **Strategy Pattern** - Different permission modes
- **Rule Engine Pattern** - Configurable permission rules
- **Cache Pattern** - Reduce repeated prompts

---

## Case 3: Streaming Response Processing

### Scenario Overview

User request: "Explain how the QueryEngine works"

This is a complex query requiring a longer response. This case shows how streaming responses:
1. Receive data stream from LLM API
2. Update UI in real-time
3. Handle tool calls in stream
4. Manage interruptions and errors

### Complete Streaming Flow

```
LLM API Connection Established
  ↓
[1] Initialize Stream Reader
  ↓
[2] Event Loop Processes Stream Events
  ↓
[3] Parse SSE Events
  ↓
[4] Dispatch to Handlers
  ↓
[5] Update UI State
  ↓
[6] Handle Tool Calls
  ↓
[7] Stream Complete or Error
  ↓
Final Response Generated
```

### Detailed Code Trace

**Step 1: Initialize Stream Reader**

File: `src/QueryEngine.ts` (lines 178-231)

```typescript
async callLLMStream(
  messages: Message[],
  tools: Tool[]
): Promise<AsyncStream> {
  // 1.1 Build API request
  const requestBody = {
    model: this.config.model,
    max_tokens: this.config.maxTokens,
    messages: messages,
    tools: tools.map(t => this.serializeTool(t)),
    stream: true
  }

  // 1.2 Send request
  const response = await fetch(`${this.apiURL}/v1/messages`, {
    method: 'POST',
    headers: {
      'x-api-key': this.apiKey,
      'anthropic-version': '2023-06-01',
      'content-type': 'application/json'
    },
    body: JSON.stringify(requestBody)
  })

  // 1.3 Check response status
  if (!response.ok) {
    throw new QueryEngineError(
      `API request failed: ${response.status}`,
      { status: response.status, body: await response.text() }
    )
  }

  // 1.4 Create stream reader
  return new AsyncStream(response.body)
}
```

---

**Step 2: Event Loop Processes Stream Events**

File: `src/QueryEngine.ts` (lines 233-289)

```typescript
private async processStream(
  stream: AsyncStream
): Promise<QueryResponse> {
  // State tracking
  const state = {
    content: '',
    toolCalls: [] as ToolCall[],
    currentToolCall: null as ToolCall | null,
    blocks: [] as ContentBlock[]
  }

  // 2.1 Subscribe to stream events
  const subscription = stream.subscribe({
    // 2.2 Process each event
    next: (event) => {
      try {
        this.handleStreamEvent(event, state)
      } catch (error) {
        this.logger.error('Error handling stream event', error)
      }
    },

    // 2.3 Handle complete
    complete: () => {
      this.logger.info('Stream complete', {
        contentLength: state.content.length,
        toolCallCount: state.toolCalls.length
      })
    },

    // 2.4 Handle error
    error: (error) => {
      this.logger.error('Stream error', error)
      throw new QueryEngineError('Stream processing failed', { cause: error })
    }
  })

  // 2.5 Wait for stream completion
  await stream.toPromise()

  return {
    content: state.content,
    toolCalls: state.toolCalls,
    blocks: state.blocks
  }
}
```

---

**Step 3: Parse SSE Events**

File: `src/services/api/StreamParser.ts` (lines 45-134)

```typescript
class StreamParser {
  parse(chunk: Buffer): StreamEvent[] {
    const events: StreamEvent[] = []
    const lines = chunk.toString().split('\n')

    for (const line of lines) {
      // 3.1 SSE format: "data: {...}"
      if (!line.startsWith('data: ')) {
        continue
      }

      const data = line.slice(6)  // Remove "data: " prefix

      try {
        // 3.2 Parse JSON
        const event = JSON.parse(data)

        // 3.3 Validate event type
        if (this.isValidEvent(event)) {
          events.push(this.normalizeEvent(event))
        }
      } catch (error) {
        // Skip invalid JSON
        this.logger.warn('Failed to parse SSE event', { data, error })
      }
    }

    return events
  }

  private isValidEvent(event: any): boolean {
    const validTypes = [
      'message_start',
      'message_delta',
      'content_block_start',
      'content_block_delta',
      'content_block_stop',
      'message_stop'
    ]

    return validTypes.includes(event.type)
  }
}
```

**SSE Event Example:**
```
data: {"type":"message_start","message":{"id":"msg-123","role":"assistant","content":[]}}

data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"The "}}

data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"QueryEngine "}}

data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"is "}}

data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"responsible "}}

...
```

---

**Step 4: Dispatch to Handlers**

File: `src/QueryEngine.ts` (lines 291-356)

```typescript
private handleStreamEvent(
  event: StreamEvent,
  state: StreamState
): void {
  switch (event.type) {
    case 'message_start':
      this.handleMessageStart(event, state)
      break

    case 'content_block_start':
      this.handleContentBlockStart(event, state)
      break

    case 'content_block_delta':
      this.handleContentBlockDelta(event, state)
      break

    case 'content_block_stop':
      this.handleContentBlockStop(event, state)
      break

    case 'message_stop':
      this.handleMessageStop(event, state)
      break

    default:
      this.logger.warn('Unknown event type', { type: event.type })
  }
}

private handleContentBlockDelta(
  event: ContentBlockDeltaEvent,
  state: StreamState
): void {
  const block = state.blocks[event.index]

  if (block.type === 'text') {
    // 4.1 Process text content
    const text = event.delta.text
    state.content += text

    // 4.2 Trigger token event (UI update)
    this.emit('token', text)
  }

  if (block.type === 'tool_use') {
    // 4.3 Process tool call arguments
    if (event.delta.type === 'input_json_delta') {
      state.currentToolCall.input += event.delta.partial_json
    }
  }
}
```

---

**Step 5: Update UI State**

File: `src/components/ResponseStream.tsx` (lines 67-145)

```typescript
const ResponseStream: React.FC<ResponseStreamProps> = ({
  queryEngine
}) => {
  const [content, setContent] = useState('')
  const [isStreaming, setIsStreaming] = useState(false)
  const [cursorVisible, setCursorVisible] = useState(true)

  useEffect(() => {
    let mounted = true

    // 5.1 Subscribe to QueryEngine events
    const onToken = (token: string) => {
      if (!mounted) return

      // 5.2 Use React state update
      setContent(prev => prev + token)

      // 5.3 Auto-scroll to bottom
      scrollToBottom()
    }

    const onStart = () => {
      if (!mounted) return
      setIsStreaming(true)
    }

    const onComplete = () => {
      if (!mounted) return
      setIsStreaming(false)
    }

    queryEngine.on('token', onToken)
    queryEngine.on('start', onStart)
    queryEngine.on('complete', onComplete)

    // 5.4 Cursor blinking animation
    const cursorInterval = setInterval(() => {
      setCursorVisible(prev => !prev)
    }, 500)

    return () => {
      mounted = false
      queryEngine.off('token', onToken)
      queryEngine.off('start', onStart)
      queryEngine.off('complete', onComplete)
      clearInterval(cursorInterval)
    }
  }, [queryEngine])

  return (
    <Box flexDirection="column">
      <Box flexGrow={1}>
        <Text>{content}</Text>
      </Box>

      {/* Streaming cursor */}
      {isStreaming && cursorVisible && (
        <Text dimColor>▊</Text>
      )}
    </Box>
  )
}
```

**UI Rendering Process:**
```
Initial: [empty]

Token 1: "T"
UI: "T▊"

Token 2: "he"
UI: "The▊"

Token 3: " QueryEngine"
UI: "The QueryEngine▊"

Token 4: " is"
UI: "The QueryEngine is▊"

Token 5: " responsible"
UI: "The QueryEngine is responsible▊"

...continues updating until complete
```

---

**Step 6: Handle Tool Calls in Stream**

File: `src/QueryEngine.ts` (lines 358-423)

```typescript
private handleContentBlockStart(
  event: ContentBlockStartEvent,
  state: StreamState
): void {
  const block = event.content_block

  if (block.type === 'tool_use') {
    // 6.1 Create tool call object
    const toolCall: ToolCall = {
      id: block.id,
      name: block.name,
      input: '',  // Will accumulate in delta events
      status: 'pending'
    }

    state.currentToolCall = toolCall
    state.toolCalls.push(toolCall)

    // 6.2 Trigger tool call start event
    this.emit('toolCallStart', toolCall)
  }
}

private handleContentBlockStop(
  event: ContentBlockStopEvent,
  state: StreamState
): void {
  if (state.currentToolCall) {
    // 6.3 Tool call arguments complete
    const toolCall = state.currentToolCall

    // Parse complete JSON input
    try {
      toolCall.input = JSON.parse(toolCall.input as string)
      toolCall.status = 'ready'

      // 6.4 Trigger tool call ready event
      this.emit('toolCallReady', toolCall)
    } catch (error) {
      toolCall.status = 'error'
      toolCall.error = error.message

      this.emit('toolCallError', toolCall, error)
    }

    state.currentToolCall = null
  }
}
```

---

**Step 7: Stream Complete or Error Handling**

File: `src/QueryEngine.ts` (lines 425-489)

```typescript
private async processStream(
  stream: AsyncStream
): Promise<QueryResponse> {
  try {
    // ... stream processing logic

    // 7.1 Wait for stream completion
    await stream.toPromise()

    // 7.2 Validate response completeness
    if (!state.content && state.toolCalls.length === 0) {
      throw new QueryEngineError('Empty response from LLM')
    }

    // 7.3 Calculate usage statistics
    const usage = {
      promptTokens: this.calculatePromptTokens(messages),
      completionTokens: state.content.length / 4,  // Estimate
      totalTokens: 0
    }
    usage.totalTokens = usage.promptTokens + usage.completionTokens

    // 7.4 Cost tracking
    this.costTracker.record(usage)

    return {
      content: state.content,
      toolCalls: state.toolCalls,
      usage,
      model: this.config.model
    }

  } catch (error) {
    // 7.5 Error handling and retry logic
    if (this.isRetryableError(error)) {
      this.logger.warn('Retryable error, retrying...', { error })

      // Exponential backoff retry
      const delay = this.calculateBackoff(this.retryCount)
      await this.sleep(delay)

      return this.retryQuery(messages, tools)
    }

    // Non-retryable error
    throw new QueryEngineError(
      'Query failed after retries',
      { cause: error, retryCount: this.retryCount }
    )
  }
}
```

---

### Key Takeaways

**Streaming Processing Architecture:**
1. **Async Stream Reader** - Handle server-sent events
2. **Event Loop** - Continuous stream event processing
3. **SSE Parser** - Parse Server-Sent Events format
4. **Event Dispatcher** - Route events by type
5. **State Machine** - Track stream processing state
6. **Real-time UI Updates** - React components subscribe to events
7. **Error Recovery** - Retry and fallback strategies

**Key Files:**
- `src/QueryEngine.ts` - Stream processing coordination
- `src/services/api/StreamParser.ts` - SSE parsing
- `src/components/ResponseStream.tsx` - UI updates
- `src/utils/costTracker.ts` - Cost tracking

**Design Patterns:**
- **Observer Pattern** - Event-driven architecture
- **State Machine Pattern** - Stream processing state transitions
- **Parser Pattern** - SSE event parsing
- **Retry Pattern** - Exponential backoff retry

---

## Case 4: MCP Server Connection

### Scenario Overview

User configures an MCP filesystem server and requests: "List all files in the /project directory"

This case shows how to:
1. Start and manage MCP server processes
2. Establish JSON-RPC communication
3. Call MCP tools
4. Handle server responses and errors

### Complete MCP Integration Flow

```
MCP Server Configuration Loaded
  ↓
[1] Start Server Process
  ↓
[2] Initialize JSON-RPC Client
  ↓
[3] Send initialize Request
  ↓
[4] Wait for Server Ready
  ↓
[5] List Available Tools
  ↓
[6] Call Tool
  ↓
[7] Process Response
  ↓
[8] Cleanup and Shutdown
```

### Detailed Code Trace

**Step 1: Start Server Process**

File: `src/services/mcp/MCPServerManager.ts` (lines 67-145)

```typescript
async startServer(config: MCPServerConfig): Promise<MCPClient> {
  // 1.1 Validate configuration
  this.validateConfig(config)

  // 1.2 Prepare environment variables
  const env = {
    ...process.env,
    ...config.env
  }

  // 1.3 Spawn subprocess
  const serverProcess = spawn(config.command, config.args, {
    env,
    stdio: ['pipe', 'pipe', 'pipe'],  // stdin, stdout, stderr
    detached: false
  })

  // 1.4 Error handling
  serverProcess.on('error', (error) => {
    this.logger.error('MCP server process error', {
      server: config.name,
      error
    })
    throw new MCPError(`Failed to start server: ${error.message}`)
  })

  // 1.5 Listen for process exit
  serverProcess.on('exit', (code, signal) => {
    this.logger.info('MCP server exited', {
      server: config.name,
      code,
      signal
    })
    this.handleServerExit(config.name, code, signal)
  })

  // 1.6 Capture stderr for debugging
  serverProcess.stderr.on('data', (data) => {
    this.logger.debug('MCP server stderr', {
      server: config.name,
      output: data.toString()
    })
  })

  // 1.7 Create client
  const client = new MCPClient({
    name: config.name,
    process: serverProcess
  })

  // 1.8 Initialize connection
  await client.initialize()

  return client
}
```

---

**Step 2: Initialize JSON-RPC Client**

File: `src/services/mcp/MCPClient.ts` (lines 89-178)

```typescript
class MCPClient {
  private process: ChildProcess
  private requestId: number = 1
  private pendingRequests: Map<number, PendingRequest> = new Map()
  private eventHandlers: Map<string, EventHandler[]> = new Map()

  constructor(config: MCPClientConfig) {
    this.process = config.process

    // 2.1 Setup stdout handler
    this.process.stdout.on('data', (data) => {
      this.handleMessage(data.toString())
    })

    // 2.2 Setup stdin writer
    this.stdin = this.process.stdin
  }

  private handleMessage(message: string): void {
    // 2.3 Split by lines (JSON-RPC one message per line)
    const lines = message.split('\n').filter(Boolean)

    for (const line of lines) {
      try {
        const msg = JSON.parse(line)
        this.dispatchMessage(msg)
      } catch (error) {
        this.logger.error('Failed to parse JSON-RPC message', {
          message: line,
          error
        })
      }
    }
  }

  private dispatchMessage(msg: JSONRPCMessage): void {
    // 2.4 Distinguish response and notification
    if (msg.id !== undefined) {
      // This is a response to a previous request
      this.handleResponse(msg)
    } else {
      // This is a server-pushed notification
      this.handleNotification(msg)
    }
  }
}
```

---

**Step 3: Send initialize Request**

File: `src/services/mcp/MCPClient.ts` (lines 180-234)

```typescript
async initialize(): Promise<void> {
  // 3.1 Send initialize request
  const initRequest = {
    jsonrpc: '2.0',
    id: this.requestId++,
    method: 'initialize',
    params: {
      protocolVersion: '2024-11-05',
      capabilities: {
        tools: {},
        resources: {}
      },
      clientInfo: {
        name: 'claude-code',
        version: '2.1.88'
      }
    }
  }

  this.sendMessage(initRequest)

  // 3.2 Wait for initialize response
  const response = await this.waitForResponse(initRequest.id)

  if (response.error) {
    throw new MCPError(
      `Initialize failed: ${response.error.message}`,
      { code: response.error.code }
    )
  }

  this.serverCapabilities = response.result.capabilities
  this.serverInfo = response.result.serverInfo

  // 3.3 Send initialized notification
  const initializedNotification = {
    jsonrpc: '2.0',
    method: 'notifications/initialized'
  }

  this.sendMessage(initializedNotification)

  this.logger.info('MCP client initialized', {
    server: this.serverInfo,
    capabilities: this.serverCapabilities
  })
}
```

**Initialize Protocol Exchange:**
```json
// Client → Server
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {},
      "resources": {}
    },
    "clientInfo": {
      "name": "claude-code",
      "version": "2.1.88"
    }
  }
}

// Server → Client
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {
        "listChanged": true
      },
      "resources": {}
    },
    "serverInfo": {
      "name": "filesystem-server",
      "version": "1.0.0"
    }
  }
}

// Client → Server (notification)
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

---

**Step 4: Wait for Server Ready**

File: `src/services/mcp/MCPClient.ts` (lines 236-267)

```typescript
async waitForReady(): Promise<void> {
  // Some MCP servers may need additional initialization time

  // 4.1 Check server status
  const maxWaitTime = 30000  // 30 second timeout
  const startTime = Date.now()

  while (Date.now() - startTime < maxWaitTime) {
    // 4.2 Send ping to test connection
    try {
      const result = await this.ping()

      if (result === 'pong') {
        this.logger.info('MCP server is ready')
        return
      }
    } catch (error) {
      // Server not ready yet, wait and retry
      await this.sleep(500)
    }
  }

  throw new MCPError('Server did not become ready in time')
}

async ping(): Promise<string> {
  const request = {
    jsonrpc: '2.0',
    id: this.requestId++,
    method: 'ping'
  }

  this.sendMessage(request)

  const response = await this.waitForResponse(request.id)
  return response.result
}
```

---

**Step 5: List Available Tools**

File: `src/services/mcp/MCPClient.ts` (lines 269-312)

```typescript
async listTools(): Promise<Tool[]> {
  const request = {
    jsonrpc: '2.0',
    id: this.requestId++,
    method: 'tools/list',
    params: {}
  }

  this.sendMessage(request)

  const response = await this.waitForResponse(request.id)

  if (response.error) {
    throw new MCPError(
      `Failed to list tools: ${response.error.message}`,
      { code: response.error.code }
    )
  }

  const tools: Tool[] = response.result.tools

  this.logger.info('MCP tools listed', {
    count: tools.length,
    tools: tools.map(t => t.name)
  })

  return tools
}
```

**MCP Tool List Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "read_file",
        "description": "Reads the contents of a file",
        "inputSchema": {
          "type": "object",
          "properties": {
            "path": {
              "type": "string",
              "description": "Path to the file to read"
            }
          },
          "required": ["path"]
        }
      },
      {
        "name": "list_directory",
        "description": "Lists the contents of a directory",
        "inputSchema": {
          "type": "object",
          "properties": {
            "path": {
              "type": "string",
              "description": "Path to the directory"
            }
          },
          "required": ["path"]
        }
      }
    ]
  }
}
```

---

**Step 6: Call Tool**

File: `src/services/mcp/MCPClient.ts` (lines 314-378)

```typescript
async callTool(
  toolName: string,
  args: any
): Promise<ToolResult> {
  // 6.1 Validate tool exists
  const tools = await this.listTools()
  const tool = tools.find(t => t.name === toolName)

  if (!tool) {
    throw new MCPError(`Tool not found: ${toolName}`)
  }

  // 6.2 Build tool call request
  const request = {
    jsonrpc: '2.0',
    id: this.requestId++,
    method: 'tools/call',
    params: {
      name: toolName,
      arguments: args
    }
  }

  // 6.3 Send request
  this.sendMessage(request)

  // 6.4 Wait for response
  const response = await this.waitForResponse(request.id)

  if (response.error) {
    throw new MCPError(
      `Tool call failed: ${response.error.message}`,
      {
        code: response.error.code,
        tool: toolName,
        arguments: args
      }
    )
  }

  const result = response.result

  this.logger.info('MCP tool call completed', {
    tool: toolName,
    arguments: args,
    result: result
  })

  return result
}

private sendMessage(message: any): void {
  const json = JSON.stringify(message) + '\n'

  try {
    this.stdin.write(json)
  } catch (error) {
    throw new MCPError('Failed to send message to server', {
      cause: error
    })
  }
}
```

**Tool Call Request:**
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "list_directory",
    "arguments": {
      "path": "/project"
    }
  }
}
```

---

**Step 7: Process Response**

File: `src/services/mcp/MCPClient.ts` (lines 380-445)

```typescript
private handleResponse(msg: JSONRPCMessage): void {
  const { id, result, error } = msg

  // 7.1 Find pending request
  const pending = this.pendingRequests.get(id)

  if (!pending) {
    this.logger.warn('Received response for unknown request', { id })
    return
  }

  // 7.2 Remove from pending list
  this.pendingRequests.delete(id)

  // 7.3 Handle error or result
  if (error) {
    pending.reject({
      code: error.code,
      message: error.message,
      data: error.data
    })
  } else {
    pending.resolve(result)
  }
}

private async waitForResponse(requestId: number): Promise<any> {
  return new Promise((resolve, reject) => {
    // Set timeout
    const timeout = setTimeout(() => {
      this.pendingRequests.delete(requestId)
      reject(new MCPError('Request timeout'))
    }, 30000)  // 30 second timeout

    // Register pending request
    this.pendingRequests.set(requestId, {
      resolve: (value) => {
        clearTimeout(timeout)
        resolve(value)
      },
      reject: (error) => {
        clearTimeout(timeout)
        reject(error)
      },
      timestamp: Date.now()
    })
  })
}
```

**Tool Call Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Files in /project:\n- package.json\n- src/\n- README.md\n"
      }
    ]
  }
}
```

---

**Step 8: Cleanup and Shutdown**

File: `src/services/mcp/MCPClient.ts` (lines 447-489)

```typescript
async shutdown(): Promise<void> {
  this.logger.info('Shutting down MCP client')

  // 8.1 Send shutdown notification
  try {
    const notification = {
      jsonrpc: '2.0',
      method: 'notifications/cancelled'
    }

    this.sendMessage(notification)
  } catch (error) {
    // Ignore send failures, server may already be closed
  }

  // 8.2 Close stdin
  this.stdin.end()

  // 8.3 Wait for process exit (max 5 seconds)
  const exitPromise = new Promise<void>((resolve) => {
    const timeout = setTimeout(() => {
      // If process didn't exit, force kill
      this.logger.warn('MCP server did not exit gracefully, killing')
      this.process.kill('SIGKILL')
      resolve()
    }, 5000)

    this.process.once('exit', () => {
      clearTimeout(timeout)
      resolve()
    })
  })

  await exitPromise

  // 8.4 Cleanup resources
  this.pendingRequests.clear()
  this.eventHandlers.clear()

  this.logger.info('MCP client shutdown complete')
}
```

---

### Key Takeaways

**MCP Integration Architecture:**
1. **Process Management** - Start and monitor server subprocess
2. **JSON-RPC Protocol** - Standardized communication format
3. **Async Request/Response** - Promise-based API
4. **Tool Discovery** - Dynamically list available tools
5. **Error Handling** - Timeout, retry, and fallback
6. **Resource Cleanup** - Graceful shutdown

**Key Files:**
- `src/services/mcp/MCPServerManager.ts` - Server management
- `src/services/mcp/MCPClient.ts` - JSON-RPC client
- `src/services/mcp/MCPToolAdapter.ts` - Tool adapter

**Design Patterns:**
- **Client-Server Pattern** - MCP protocol
- **Adapter Pattern** - MCP tools to internal tools
- **Observer Pattern** - Event-driven communication
- **Factory Pattern** - Create different transport clients

---

## Part 6 Summary

### Case Coverage

This part demonstrated 4 complete cases:

1. **Request Processing Flow** (Case 1)
   - Complete path from user input to tool execution
   - QueryEngine's central coordination role
   - LLM API integration and streaming responses

2. **Permission Check Flow** (Case 2)
   - 5-layer security check mechanism
   - Rule engine and decision caching
   - User interaction and risk prompts

3. **Streaming Response Processing** (Case 3)
   - SSE event parsing
   - Real-time UI updates
   - Error recovery and retry

4. **MCP Integration Flow** (Case 4)
   - JSON-RPC communication
   - Tool calls and response handling
   - Process lifecycle management

### Key Learning Outcomes

**Code Tracing Skills:**
- ✅ Understand end-to-end execution flow
- ✅ Identify key data structures
- ✅ Master event-driven architecture
- ✅ Understand async programming patterns

**Architecture Understanding:**
- ✅ QueryEngine as central coordinator
- ✅ Tool system's plugin architecture
- ✅ Permission system's multi-layer protection
- ✅ Streaming's real-time nature

**Practice Recommendations:**

1. **When Reading Code**
   - Start from entry points and trace
   - Draw call relationship diagrams
   - Note key data structures
   - Mark control flow branches

2. **Verify Understanding**
   - Add debug logging
   - Use TypeScript type checking
   - Run and observe behavior
   - Write test cases

3. **Deep Analysis**
   - Compare similar functionality
   - Identify design patterns
   - Extract reusable patterns
   - Think about improvements

### Next Part Preview

**Part 7: Research Resources**
- Complete file index
- Code cross-references
- Architecture diagrams and flowcharts
- Extended reading recommendations

---

**Word Count Estimate:**
- Chinese: ~6,000 words
- English: ~4,000 words (corresponding English version)

**Next Part:** Part 7 - Research Resources
