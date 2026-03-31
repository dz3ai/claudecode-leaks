# Claude Code Source Leak Research Guide

> Full Edition
> Authors: Claude Code Security Research Community
> Date: 2026-03-31
> Version: 1.0.0

---

## Part 1: Incident Background

### 1.1 Timeline of Events

**First Leak (February 2025)**
- February 2025: Researchers first discover source map file in Claude Code npm package
- File location: `cli.js.map` (~60MB)
- Exposed content: Complete TypeScript source code
- Resolution: Not fully fixed

**Second Leak (March 31, 2026)**
- March 31, 2026: [@Fried_rice](https://x.com/Fried_rice) publicly highlights the issue on Twitter
- Tweet: https://x.com/Fried_rice/status/2038894956459290963
- Affected versions: Multiple versions including latest v2.1.88
- Leak scale: 1,906 TypeScript files, ~512,000+ lines of code

**Key Timeline:**
```
2025-02-XX  First source map leak discovered
2025-03-XX  Partial fix, but not complete removal
2026-03-31  @Fried_rice public disclosure, widespread attention
2026-03-31  Multiple research mirror repositories created
2026-04-01  Security community deep-dive analysis, supply chain security discussions
```

### 1.2 Leak Mechanism Explained

#### Source Map Principles

Source maps are mapping files that reference compiled/minified code back to original source code for debugging purposes.

**Normal Flow:**
```
TypeScript Source → Compile → JavaScript + Source Map
                           ↓
                   DevTools loads Source Map
                           ↓
              Browser/IDE shows original source
```

**Leak Flow:**
```
1. Claude Code build process generates Source Map
2. Source Map file (cli.js.map) is packaged into npm bundle
3. Source Map contains URLs pointing to original source code
4. Original source hosted on Anthropic's R2 storage bucket
5. R2 bucket configured for public access
         ↓
    Complete source code publicly downloadable
```

#### Technical Details

**Source Map File Content Example:**
```json
{
  "version": 3,
  "sources": [
    "src/QueryEngine.ts",
    "src/Tool.ts",
    "src/main.tsx",
    ... 1,900+ files
  ],
  "sourcesContent": [
    "export class QueryEngine { ... }",
    "export interface Tool { ... }",
    ...
  ],
  "mappings": "...",
  "sourceRoot": "https://anthropic-r2-bucket.com/src/"
}
```

**Root Causes:**
1. **Build configuration error**: Production build did not disable source map generation
2. **Packaging oversight**: `.npmignore` did not exclude `.map` files
3. **Storage misconfiguration**: Source code storage bucket allowed public access
4. **Release process gap**: Lack of pre-publish security checks

### 1.3 Impact Assessment

#### Direct Impact

**Exposed Content:**
- ✅ Complete TypeScript source code (1,906 files)
- ✅ Core architecture implementation (Tool System, Query Engine, Permission System)
- ✅ API integration logic (Anthropic API, OAuth, MCP)
- ✅ Authentication implementation (JWT, OAuth 2.0 flows)
- ✅ Telemetry and monitoring logic (OpenTelemetry)
- ✅ UI component code (React + Ink)
- ✅ Configuration and metadata

**NOT Exposed:**
- ❌ API keys and credentials
- ❌ User data
- ❌ Model weights
- ❌ Private encryption keys
- ❌ Internal infrastructure configuration

#### Stakeholder Analysis

**1. Anthropic (Developer)**
- **IP Exposure**: Core algorithms and architecture design publicly revealed
- **Competitive Advantage**: Competitors can study implementation details
- **Security Risks**: Potential vulnerabilities may be discovered and exploited
- **Brand Reputation**: Supply chain security practices questioned

**2. Users (Low Direct Risk)**
- ✅ No direct security risk (no sensitive data exposed)
- ⚠️ Indirect risk: Attackers may study code to find vulnerabilities
- ⚠️ Privacy concerns: Telemetry logic transparency may reveal data collection practices

**3. Competitors**
- ✅ Opportunity to learn Claude Code's architecture design
- ✅ Can borrow best practices and design patterns
- ⚠️ Must avoid direct copying (legal risk)

**4. Open Source Community**
- ✅ Learning opportunity for modern AI tool architecture
- ✅ Case study for supply chain security
- ✅ Resource for understanding agent system implementation

**5. AI Agent Ecosystem**
- 🔍 Competitive landscape changes: Implementation details transparent
- 🔍 Security trust impact: Users may question product security
- 🔍 Open source alternatives opportunity: Can create open source implementations based on leaked code

#### Risk Matrix

| Dimension | Risk Level | Description |
|-----------|------------|-------------|
| User Data Security | 🟢 Low | No user data exposed |
| IP Protection | 🔴 High | Complete source code publicly available |
| Competitive Advantage | 🟡 Medium | Architecture learnable, but models and brand remain protected |
| System Security | 🟡 Medium | Potential vulnerabilities discoverable, but no direct exploitation path |
| Supply Chain Security | 🔴 High | Exposes serious supply chain security process failures |

### 1.4 Research Value and Ethical Considerations

#### Academic Research Value

**1. Software Supply Chain Security**
- Real-world supply chain leak case study
- Complete demonstration of source map risks
- Practical example of CI/CD security failures

**2. AI Tool Architecture**
- Implementation reference for modern AI agent systems
- Best practices for tool system design
- Real-world multi-agent coordination applications

**3. Terminal UI Development**
- React + Ink application in large-scale CLI tools
- Architecture patterns for complex terminal applications
- Streaming response UI implementation

**4. Distributed Systems**
- MCP protocol practical integration
- LSP protocol server-side implementation
- Bridge system communication patterns

#### Ethical and Legal Boundaries

**✅ Permitted Research Activities:**
- Academic research and educational purposes
- Security research and vulnerability analysis (responsible disclosure)
- Learning architecture design and programming patterns
- Writing research and analysis documentation
- Creating non-commercial open source tools (with attribution)

**❌ Prohibited Activities:**
- Claiming code as your own development
- Commercial use or resale
- Direct copy-paste to create competing products (legal risk)
- Using exposed information for attacks
- Infringing on Anthropic's intellectual property

**Research Ethics Principles:**
1. **Respect Intellectual Property**: Clearly attribute code ownership, do not claim originality
2. **Responsible Disclosure**: Contact vendor first when discovering security issues
3. **Educational Purpose**: Research should serve community learning and security improvement
4. **Transparency and Integrity**: Cite sources, explain research methodology

#### Usage Recommendations

**For Security Researchers:**
- Focus on supply chain security lessons extraction
- Study build and release process improvements
- Responsibly disclose discovered issues

**For Developers:**
- Learn architecture design and programming patterns
- Understand modern AI tool implementation
- Apply learned best practices to your own projects

**For Academic Researchers:**
- Analyze AI agent system design patterns
- Study CLI implementation of human-AI interaction
- Explore software supply chain security frameworks

**For Enterprises and Organizations:**
- Evaluate your own supply chain security practices
- Establish code review and release verification mechanisms
- Learn from Claude Code's architecture design (without copying code)

---

## Part 2: Quick Navigation

### 2.1 Navigate by Research Topic

#### Tool System and Permission Research
**Core Files:**
- `src/Tool.ts` - Tool interface definitions (29KB)
- `src/tools/` - 43 tool implementations
- `src/hooks/toolPermission/` - Permission checking system

**Research Focus:**
- Tool interface design patterns
- Permission granularity control
- User interaction flows
- Dangerous operation protection mechanisms

**Recommended Reading Order:**
1. `src/Tool.ts` - Understand tool type system
2. `src/tools/BashTool/BashTool.ts` - Simple tool example
3. `src/tools/AgentTool/AgentTool.ts` - Complex tool example
4. `src/hooks/toolPermission/` - Permission implementation details

#### LLM Interaction and Query Engine Research
**Core Files:**
- `src/QueryEngine.ts` - Query engine core (46KB)
- `src/query.ts` - Query pipeline (68KB)
- `src/context.ts` - Context management

**Research Focus:**
- Streaming response handling
- Tool call loops
- Retry and error handling
- Token counting and cost tracking

**Recommended Reading Order:**
1. `src/QueryEngine.ts` (lines 1-500) - Core interfaces and initialization
2. `src/QueryEngine.ts` (lines 500-1000) - Streaming response handling
3. `src/QueryEngine.ts` (lines 1000+) - Tool call coordination
4. `src/cost-tracker.ts` - Cost tracking implementation

#### Supply Chain Security Research
**Core Files:**
- `package.json` - Package configuration analysis
- `.npmignore` - Packaging exclusion configuration
- Build configuration files (infer from source)

**Research Focus:**
- Source map generation configuration
- Dependency management and locking
- Build process security
- Release verification mechanisms

**Recommended Reading Order:**
1. Analyze build scripts in `package.json`
2. Check for `.npmignore` or `.gitignore`
3. Infer build tool (Bun's bundling configuration)
4. Compare with best practices documentation

#### UI/UX Research and Terminal Interaction
**Core Files:**
- `src/main.tsx` - Application entry point (786KB)
- `src/components/` - 140+ UI components
- `src/ink/` - Ink framework wrapper

**Research Focus:**
- React in terminal environment
- Component design for complex UIs
- Streaming text rendering optimization
- Keyboard interaction handling

**Recommended Reading Order:**
1. `src/ink.ts` - Ink initialization
2. `src/components/` - Select typical components
3. `src/screens/` - Complete screen implementations
4. `src/main.tsx` - Application assembly logic

#### Integration and Protocol Research
**Core Directories:**
- `src/services/mcp/` - MCP protocol integration
- `src/services/lsp/` - LSP protocol integration
- `src/bridge/` - IDE bridge system

**Research Focus:**
- MCP protocol server-side implementation
- LSP protocol client-side implementation
- Bidirectional communication mechanisms
- Authentication and authorization

**Recommended Reading Order:**
1. `src/services/mcp/` - Understand MCP architecture
2. `src/services/lsp/` - Understand LSP integration
3. `src/bridge/bridgeMessaging.ts` - Communication protocol
4. `src/bridge/jwtUtils.ts` - Security authentication

### 2.2 Navigate by Tech Stack

#### TypeScript Experts
**Key Patterns:**
- Strict type system application
- Advanced type techniques (conditional types, mapped types)
- Generic constraints and inference
- Modularity and dependency injection

**Learning Focus:**
- Zod schema validation integration
- Type-safe tool definitions
- Generic utility function design

**Recommended Files:**
- `src/Tool.ts` - Type definition exemplar
- `src/types/` - Type definition collection
- `src/schemas/` - Zod schemas

#### React/Ink Developers
**Key Patterns:**
- Deep hooks usage
- Component composition patterns
- State management strategies
- Performance optimization techniques

**Learning Focus:**
- React adaptation for terminal environment
- Streaming data rendering
- Keyboard event handling
- Layout system design

**Recommended Files:**
- `src/ink/hooks/` - Custom hooks
- `src/components/` - Component implementations
- `src/screens/` - Complete UIs

#### System Architects
**Key Patterns:**
- Plugin system design
- Event-driven architecture
- Service layering
- Dependency injection

**Learning Focus:**
- Tool registration mechanisms
- Command dispatch system
- Service abstraction layer
- Configuration management

**Recommended Files:**
- `src/plugins/` - Plugin system
- `src/coordinator/` - Coordinator
- `src/services/` - Service layer
- `src/state/` - State management

#### Security Researchers
**Key Patterns:**
- Permission checking system
- Sandbox mechanisms
- Input validation
- Audit logging

**Learning Focus:**
- Permission model design
- Dangerous operation protection
- Credential management
- Telemetry and monitoring

**Recommended Files:**
- `src/hooks/toolPermission/` - Permission system
- `src/services/oauth/` - Authentication flow
- `src/diagnosticTracking.ts` - Telemetry
- `src/internalLogging.ts` - Logging system

### 2.3 Recommended Reading Paths

#### Path A: Beginner Path (4-6 hours)

**Goal:** Understand Claude Code's basic architecture and working principles

**Step 1: Background (30 minutes)**
- Read Part 1 of this guide (Incident Background)
- Review README.md for project overview
- Browse CLAUDE.md architecture overview

**Step 2: Core Concepts (1.5 hours)**
- Read `src/Tool.ts` (29KB)
  - Understand Tool interface
  - Learn permission model
  - Review type definitions
- Read `src/tools/BashTool/BashTool.ts`
  - Understand simple tool implementation
  - Learn Zod schema usage
  - Review permission check flow

**Step 3: Query Engine (1.5 hours)**
- Read `src/QueryEngine.ts` first 500 lines
  - Understand core interface
  - Learn initialization flow
  - Study streaming response concepts
- Skip complex implementation details
- Focus on comments and interface definitions

**Step 4: Code Practice (1-2 hours)**
- Follow Part 6, Case 1 (User input to tool execution)
- Try tracing a simple command execution
- Understand the complete call chain

**Expected Outcomes:**
- ✅ Understand basic Tool System concepts
- ✅ Learn core LLM interaction flow
- ✅ Ability to read and trace simple code execution
- ✅ Establish overall architecture understanding

#### Path B: Advanced Path (8-12 hours)

**Goal:** Deep understanding of core systems, ability to analyze architectural design

**Step 1: Complete Tool System Understanding (2 hours)**
- Deep read of `src/Tool.ts`
- Read 5-10 different tool implementations:
  - BashTool (file operations)
  - FileReadTool (file reading)
  - AgentTool (sub-agent)
  - MCPTool (MCP integration)
  - TaskCreateTool (task management)
- Summarize tool design patterns and best practices

**Step 2: Deep Query Engine (3 hours)**
- Complete read of `src/QueryEngine.ts` (46KB, ~1,400 lines)
- Focus on understanding:
  - Streaming response handling mechanisms
  - Tool call loops
  - Retry logic
  - Error handling
- Read `src/query.ts` (query pipeline)

**Step 3: Permissions and Security (2 hours)**
- Deep dive into `src/hooks/toolPermission/`
- Understand permission modes:
  - default (ask by default)
  - auto (auto-approve)
  - plan (plan mode)
- Learn dangerous operation protection
- Study sandbox mechanisms

**Step 4: Choose 2-3 Technical Topics (3-4 hours)**
Recommended topics:
- React + Ink Terminal UI
- MCP Protocol Integration
- Multi-Agent Coordination

**Step 5: Code Practice (2 hours)**
- Follow all 4 cases in Part 6
- Try tracing code execution independently
- Draw call flow diagrams

**Expected Outcomes:**
- ✅ Deep understanding of core system architecture
- ✅ Master 2-3 technical topics
- ✅ Ability to analyze design patterns and trade-offs
- ✅ Can learn and apply to your own projects

#### Path C: Expert Path (16-20 hours)

**Goal:** Comprehensive system design mastery, extract reusable architectural patterns

**Step 1: Core Architecture Deep Analysis (6 hours)**
- Complete read of all core systems:
  - Tool System (all 43 tools)
  - Command System (all 110 commands)
  - Query Engine (complete implementation)
  - Permission System (all modes)
  - Bridge System
- Summarize design patterns for each system
- Identify architectural trade-offs and design decisions

**Step 2: Technical Topics Deep Dive (6 hours)**
Choose and deeply study:
- React + Ink Terminal UI (all 140+ components)
- Bun Features (feature flags, bundle)
- MCP Protocol (complete integration)
- LSP Protocol (complete integration)
- OAuth 2.0 + JWT
- OpenTelemetry
- Multi-Agent Coordination

**Step 3: Comparative Study (4 hours)**
- Compare with other AI Agent architectures:
  - OpenAI Code Interpreter
  - GitHub Copilot
  - Cursor
  - Continue.dev
- Identify common patterns
- Analyze differentiated designs

**Step 4: Pattern Extraction (2 hours)**
- Extract reusable design patterns
- Summarize best practices
- Identify anti-patterns (designs to avoid)
- Write architecture design documentation

**Step 5: Security Research (2 hours)**
- Deep analysis of supply chain security issues
- Evaluate all security boundaries
- Identify potential vulnerabilities
- Propose improvements

**Expected Outcomes:**
- ✅ Comprehensive mastery of Claude Code architecture
- ✅ Ability to design similar agent systems
- ✅ Extract reusable architectural patterns
- ✅ Deep understanding of supply chain security
- ✅ Can guide others in learning

---

## Part 3: Core Architecture Analysis

> **Coming Soon...**
>
> Part 3 will provide in-depth analysis of Claude Code's core architecture, including:
> - 3.1 Tool System
> - 3.2 Query Engine
> - 3.3 Command System
> - 3.4 Permission System
> - 3.5 Bridge System
>
> Expected completion: 2026-04-01

---

**Document Status:** 🚧 In Progress (Parts 1-2 completed)
**Last Updated:** 2026-03-31
**Contributors:** PRs welcome to improve this document
