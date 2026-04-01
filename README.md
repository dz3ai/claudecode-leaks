# Claude Code 源代码泄露研究仓库
# Claude Code Source Leak Research Repository

> **⚠️ 重要提示 / Important Notice**:
> 本仓库仅用于**教育和安全研究目的**。This repository is for **educational and security research purposes only**.
> 原始代码版权归 Anthropic 所有。Original code is the property of Anthropic.

---

## 📋 概述 / Overview

本仓库包含了 2026 年 3 月 31 日通过 npm 包 source map 泄露的 **Claude Code 完整源代码快照**。这是一个用于软件供应链安全研究、AI 工具架构分析和防御性安全实践的学术研究项目。

This repository contains a **complete source code snapshot of Claude Code** leaked through a source map exposure in the npm package on March 31, 2026. This is an academic research project for software supply-chain security analysis, AI tool architecture study, and defensive security practices.

### 🔍 泄露事件 / The Leak Incident

- **披露时间 / Disclosure**: 2026-03-31
- **发现者 / Discovered by**: [Chaofan Shou (@Fried_rice)](https://x.com/Fried_rice)
- **泄露机制 / Leak Mechanism**: npm 包中的 source map 文件（`cli.js.map`）暴露了完整源码
- **影响版本 / Affected Versions**: 包括 v2.1.88 在内的多个版本
- **代码规模 / Code Scale**: ~1,906 个文件，512,000+ 行 TypeScript 代码

---

## 🎯 研究目的 / Research Objectives

### 主要研究方向 / Primary Research Areas

1. **软件供应链安全 / Software Supply-Chain Security**
   - Source map 风险分析
   - 构建配置最佳实践
   - CI/CD 安全检查点
   - 包发布验证流程

2. **AI 工具架构 / AI Tool Architecture**
   - LLM 驱动的 CLI 工具设计
   - 工具系统实现模式
   - 多代理协调机制
   - 权限系统设计

3. **终端 UI 开发 / Terminal UI Development**
   - React + Ink 在复杂 CLI 中的应用
   - 流式响应处理
   - 键盘交互设计

4. **分布式系统 / Distributed Systems**
   - MCP（Model Context Protocol）集成
   - LSP（Language Server Protocol）实现
   - 桥接系统架构

---

## 📚 研究指南 / Research Guide

我们提供了**双语（中/英）深度研究指南**，帮助您系统地理解泄露的源代码：

We provide **bilingual (CN/EN) in-depth research guides** to help you systematically understand the leaked source code:

### 📖 指南导航 / Guide Navigation

| 指南 / Guide | 描述 / Description | 状态 / Status |
|-------------|-------------------|--------------|
| **[中文指南](docs/guide/代码研究指南-第一部分.md)** | 完整的中文研究指南（7 部分） | ✅ 已完成 / Complete (100%) |
| **[English Guide](docs/guide/Code-Research-Guide-Part1.md)** | Complete English research guide (7 parts) | ✅ Complete (100%) |

### 📊 内容结构 / Content Structure

#### 第一部分：事件背景 / Part 1: Incident Background
- 泄露时间线 / Timeline
- 泄露机制详解 / Leak Mechanism
- 影响范围评估 / Impact Assessment
- 研究价值与伦理 / Research Value & Ethics

#### 第二部分：快速导航 / Part 2: Quick Navigation
- 按研究主题导航 / Navigate by Topic
- 按技术栈导航 / Navigate by Tech Stack
- 推荐阅读路径 / Recommended Reading Paths

#### 第三部分：核心架构解析 / Part 3: Core Architecture Analysis
- ✅ 3.1 工具系统（Tool System）
- ✅ 3.2 查询引擎（Query Engine）
- ✅ 3.3 命令系统（Command System）
- ✅ 3.4 权限系统（Permission System）
- ✅ 3.5 桥接系统（Bridge System）

**已完成 / Completed**: ~20,000 字（中文）/ ~13,500 词（英文）

#### 第四部分：关键技术专题 / Part 4: Key Technical Topics ✅
- React + Ink 终端 UI 架构 / React + Ink Terminal UI Architecture
- Bun 特性使用 / Bun Features
- MCP 协议集成 / MCP Protocol Integration
- 多代理协调 / Multi-Agent Coordination

**已完成 / Completed**: ~6,000 字（中文）/ ~4,000 词（英文）

#### 第五部分：供应链安全 Lessons / Part 5: Supply Chain Security Lessons ✅
- Source Map 风险详解 / Source Map Risk Analysis
- 构建配置最佳实践 / Build Configuration Best Practices
- CI/CD 安全检查点 / CI/CD Security Checkpoints
- 包发布验证流程 / Package Release Verification

**已完成 / Completed**: ~4,000 字（中文）/ ~2,500 词（英文）

#### 第六部分：代码阅读实战 / Part 6: Code Reading Practice ✅
- 案例 1：从用户输入到工具执行 / Case 1: User Input to Tool Execution
- 案例 2：权限检查流程 / Case 2: Permission Check Flow
- 案例 3：流式响应处理 / Case 3: Streaming Response Processing
- 案例 4：MCP 服务器连接 / Case 4: MCP Server Connection

**已完成 / Completed**: ~6,000 字（中文）/ ~4,000 词（英文）

#### 第七部分：研究资源 / Part 7: Research Resources ✅
- 关键文件索引 / Key File Index
- 代码交叉引用 / Code Cross-References
- 架构图解 / Architecture Diagrams
- 扩展阅读建议 / Extended Reading

**已完成 / Completed**: ~3,000 字（中文）/ ~2,000 词（英文）

---

## 📊 完成统计 / Completion Statistics

| 部分 | 中文 | 英文 | 完成度 |
|------|------|------|--------|
| 第一部分 | 3,500 字 | 2,300 词 | ✅ 100% |
| 第二部分 | 4,500 字 | 3,000 词 | ✅ 100% |
| 第三部分 | 20,000 字 | 13,500 词 | ✅ 100% |
| 第四部分 | 6,000 字 | 4,000 词 | ✅ 100% |
| 第五部分 | 4,000 字 | 2,500 词 | ✅ 100% |
| 第六部分 | 6,000 字 | 4,000 词 | ✅ 100% |
| 第七部分 | 3,000 字 | 2,000 词 | ✅ 100% |
| **总计** | **47,000 字** | **31,300 词** | **100%** |

**成就 / Achievement**: ✅ 超过预期目标！所有 7 个部分已完成（超过原计划 17%）

---

## 🗂️ 项目结构 / Repository Structure

### 核心源代码 / Core Source Code

```
src/
├── main.tsx                 # 应用入口点（786KB，~24,000 行）
├── QueryEngine.ts           # LLM 查询引擎核心（46KB，~1,300 行）
├── Tool.ts                  # 工具类型定义（29KB，~800 行）
├── commands.ts              # 命令注册表（25KB，~750 行）
├── tools.ts                 # 工具注册表（17KB，~450 行）
│
├── commands/                # 斜杠命令实现（~110 个命令）
├── tools/                   # Agent 工具实现（~43 个工具）
├── components/              # Ink UI 组件（~140 个组件）
├── hooks/                   # React Hooks
├── services/                # 外部服务集成
│   ├── api/                 # Anthropic API 客户端
│   ├── mcp/                 # MCP 协议集成
│   ├── lsp/                 # LSP 协议集成
│   ├── oauth/               # OAuth 2.0 认证
│   └── analytics/           # GrowthBook 特性标志
│
├── bridge/                  # IDE 桥接系统
├── coordinator/             # 多代理协调器
├── plugins/                 # 插件系统
├── skills/                  # 技能系统
├── state/                   # 状态管理
├── tasks/                   # 任务管理
└── ...                      # 更多模块
```

### 文档目录 / Documentation Directory

```
docs/
├── guide/                   # 研究指南
│   ├── README.md            # 指南导航（双语）
│   ├── PROGRESS.md          # 编写进度
│   ├── 代码研究指南-第一部分.md      # 中文版 Part 1-2
│   ├── 代码研究指南-第三部分.md      # 中文版 Part 3.1-3.2
│   ├── 代码研究指南-第三部分续.md    # 中文版 Part 3.3-3.5
│   ├── Code-Research-Guide-Part1.md # English Part 1-2
│   ├── Code-Research-Guide-Part3.md # English Part 3.1-3.2
│   └── Code-Research-Guide-Part3-Continued.md # English Part 3.3-3.5
│
└── diagrams/                # 架构图和流程图（待创建）
```

---

## 🔑 核心系统概览 / Core Systems Overview

### 1. 工具系统 / Tool System (`src/tools/`)

**43 个工具**构成了 Claude Code 的核心能力。

**43 tools** form the core capabilities of Claude Code.

| 工具类别 / Tool Category | 代表工具 / Examples |
|-----------------------|-------------------|
| 文件操作 / File Ops | FileRead, FileWrite, FileEdit, Glob, Grep |
| Shell 执行 / Shell | Bash, Shell |
| Agent 协作 / Agent | Agent, SendMessage |
| 集成 / Integration | MCP, LSP, NotebookEdit |
| 任务管理 / Tasks | TaskCreate, TaskUpdate, CronCreate |

**关键特性 / Key Features**:
- 统一的工具接口 / Unified tool interface
- Zod schema 验证 / Schema validation
- 细粒度权限控制 / Fine-grained permissions
- 进度报告 / Progress reporting

### 2. 查询引擎 / Query Engine (`src/QueryEngine.ts`)

**核心循环 / Core Loop**:
```
用户输入 → LLM API → 流式响应 → 工具调用 → 结果返回 → 循环
User Input → LLM API → Streaming → Tool Call → Result → Loop
```

**关键能力 / Key Capabilities**:
- 流式响应处理 / Streaming response handling
- 工具调用协调 / Tool call coordination
- 错误重试机制 / Error retry logic
- 成本追踪 / Cost tracking
- Thinking Mode 支持

### 3. 命令系统 / Command System (`src/commands/`)

**110 个斜杠命令** / **110 Slash Commands**

| 类型 / Type | 命令 / Commands |
|-----------|---------------|
| Git 操作 / Git | `/commit`, `/review`, `/diff` |
| 配置 / Config | `/config`, `/skills`, `/memory` |
| 诊断 / Diagnostics | `/doctor`, `/context`, `/cost` |
| MCP / MCP | `/mcp` |
| 会话 / Session | `/compact`, `/resume`, `/share` |

### 4. 权限系统 / Permission System (`src/hooks/toolPermission/`)

**5 层安全检查 / 5-Layer Security Checks**:

1. 全局权限模式 / Global permission mode
2. 工具特定检查 / Tool-specific checks
3. 规则匹配 / Rule matching
4. 上下文约束 / Context constraints
5. 用户交互 / User interaction

### 5. 桥接系统 / Bridge System (`src/bridge/`)

**IDE 集成架构 / IDE Integration Architecture**:

```
IDE Extension ←→ Bridge Server ←→ Claude Code CLI
(VS Code/Jet)   (WebSocket)     (Tool Execution)
```

**关键组件 / Key Components**:
- JWT 认证 / JWT authentication
- 消息协议 / Message protocol
- 会话管理 / Session management
- WebSocket 通信 / WebSocket communication

---

## 🛠️ 技术栈 / Tech Stack

| 类别 / Category | 技术 / Technology |
|---------------|-----------------|
| **运行时 / Runtime** | [Bun](https://bun.sh) |
| **语言 / Language** | TypeScript (strict mode) |
| **终端 UI / Terminal UI** | [React](https://react.dev) + [Ink](https://github.com/vadimdemedes/ink) |
| **CLI 解析 / CLI Parsing** | [Commander.js](https://github.com/tj/commander.js) |
| **Schema 验证 / Schema Validation** | [Zod v4](https://zod.dev) |
| **代码搜索 / Code Search** | [ripgrep](https://github.com/BurntSushi/ripgrep) |
| **协议 / Protocols** | [MCP SDK](https://modelcontextprotocol.io), LSP |
| **API** | [Anthropic SDK](https://docs.anthropic.com) |
| **遥测 / Telemetry** | OpenTelemetry + gRPC |
| **特性标志 / Feature Flags** | GrowthBook |
| **认证 / Auth** | OAuth 2.0, JWT, macOS Keychain |

---

## 📖 快速开始 / Quick Start

### 推荐学习路径 / Recommended Learning Paths

#### 🌱 初学者路径 / Beginner Path (4-6 hours)

1. 阅读**第一部分**：了解泄露事件背景 / Read Part 1: Incident background
2. 阅读**第二部分**：选择适合的学习路径 / Read Part 2: Choose your path
3. 学习**3.1 工具系统**基础概念 / Study 3.1 Tool System basics
4. 跟随**第六部分**案例实践 / Follow Part 6 case studies

**预期收获 / Expected Outcomes**:
- ✅ 理解工具系统基本概念 / Understand basic tool system concepts
- ✅ 了解 LLM 交互核心流程 / Learn core LLM interaction flow
- ✅ 能够追踪简单代码执行 / Able to trace simple code execution

#### 🚀 进阶路径 / Advanced Path (8-12 hours)

1. 完整阅读**第三部分**：5 大核心系统 / Complete Part 3: 5 core systems
2. 深入研究 **2-3 个技术专题** / Dive into 2-3 technical topics
3. 实践所有**4 个代码案例** / Practice all 4 code cases
4. 学习供应链安全 lessons / Learn supply chain security lessons

**预期收获 / Expected Outcomes**:
- ✅ 深入理解核心架构 / Deep understanding of core architecture
- ✅ 掌握多个技术专题 / Master multiple technical topics
- ✅ 能够分析设计模式 / Able to analyze design patterns

#### 🔬 专家路径 / Expert Path (16-20 hours)

1. 完整阅读**所有章节** / Complete all chapters
2. 分析所有**43 个工具**实现 / Analyze all 43 tool implementations
3. 研究所有**110 个命令** / Study all 110 commands
4. 对比其他 AI Agent 架构 / Compare with other AI agent architectures
5. 提炼可复用的设计模式 / Extract reusable design patterns

**预期收获 / Expected Outcomes**:
- ✅ 全面掌握 Claude Code 架构 / Complete mastery of Claude Code architecture
- ✅ 能够设计类似的 agent 系统 / Able to design similar agent systems
- ✅ 深入理解供应链安全 / Deep understanding of supply chain security

---

## 🔬 研究价值 / Research Value

### 对安全研究者 / For Security Researchers

- ✅ 真实的供应链泄露案例研究 / Real-world supply-chain leak case study
- ✅ Source map 风险完整演示 / Complete source map risk demonstration
- ✅ CI/CD 安全失败实际案例 / Practical CI/CD security failure examples

### 对开发者 / For Developers

- ✅ 学习现代 AI 工具架构 / Learn modern AI tool architecture
- ✅ 理解工具系统设计模式 / Understand tool system design patterns
- ✅ 研究多代理协调实现 / Study multi-agent coordination implementation

### 对学术界 / For Academia

- ✅ AI agent 系统设计模式研究 / AI agent system design pattern research
- ✅ 人机交互 CLI 实现分析 / Human-AI interaction CLI implementation analysis
- ✅ 软件供应链安全框架探讨 / Software supply-chain security framework discussion

### 对企业和组织 / For Enterprises & Organizations

- ✅ 评估自身供应链安全实践 / Evaluate your own supply-chain security practices
- ✅ 建立代码审查和发布检查机制 / Establish code review and release verification
- ✅ 借鉴架构设计（不抄袭代码） / Learn from architecture (without copying code)

---

## ⚖️ 法律与道德声明 / Legal & Ethical Disclaimer

### ✅ 允许的行为 / Permitted Activities

- 学术研究和教育目的 / Academic research and educational purposes
- 安全研究和漏洞分析（负责任披露） / Security research and vulnerability analysis (responsible disclosure)
- 学习架构设计和编程模式 / Learning architecture design and programming patterns
- 编写研究和分析文档 / Writing research and analysis documentation
- 创建非商业性质的开源工具（注明来源） / Creating non-commercial open source tools (with attribution)

### ❌ 禁止的行为 / Prohibited Activities

- 声称代码为自己开发 / Claiming code as your own development
- 商业化使用或转售 / Commercial use or resale
- 直接复制粘贴创建竞品（法律风险） / Direct copy-paste to create competing products (legal risk)
- 利用暴露信息进行攻击 / Using exposed information for attacks
- 侵犯 Anthropic 的知识产权 / Infringing on Anthropic's intellectual property

### 🔐 研究伦理原则 / Research Ethics Principles

1. **尊重知识产权 / Respect Intellectual Property**: 明确代码归属 / Clearly attribute code ownership
2. **负责任披露 / Responsible Disclosure**: 发现安全问题先联系厂商 / Contact vendor first when discovering security issues
3. **教育目的 / Educational Purpose**: 研究应服务于社区学习和安全改进 / Research should serve community learning and security improvement
4. **透明诚信 / Transparency and Integrity**: 引用来源，说明研究方法 / Cite sources, explain research methodology

---

## 📊 项目统计 / Project Statistics

| 指标 / Metric | 数值 / Value |
|------------|-----------|
| **源代码文件 / Source Files** | ~1,906 个 files |
| **代码行数 / Lines of Code** | ~512,000+ lines |
| **工具数量 / Tools** | 43 个 tools |
| **命令数量 / Commands** | 110 个 commands |
| **UI 组件 / UI Components** | ~140 个 components |
| **编程语言 / Language** | TypeScript (strict) |
| **运行时 / Runtime** | Bun |

---

## 📝 文献引用 / Citation

如果您在研究中使用本仓库，请引用 / If you use this repository in your research, please cite:

```bibtex
@misc{claudecode-leak-2026,
  title={Claude Code Source Leak Research Repository},
  author={Research Community},
  year={2026},
  month={March},
  day={31},
  howpublished={GitHub},
  url={https://github.com/dz3ai/claudecode-leaks},
  note={Security research archive for educational purposes}
}
```

---

## 🔗 相关链接 / Related Links

### 官方资源 / Official Resources
- [Claude Documentation](https://docs.anthropic.com)
- [Claude API Reference](https://docs.anthropic.com/api)
- [MCP Protocol](https://modelcontextprotocol.io)

### 社区与讨论 / Community & Discussion
- [GitHub Repository](https://github.com/dz3ai/claudecode-leaks)
- [Issues & Bug Reports](https://github.com/dz3ai/claudecode-leaks/issues)
- [Pull Requests Welcome](https://github.com/dz3ai/claudecode-leaks/pulls)

### 相关研究 / Related Research
- [SLSA Framework](https://slsa.dev) - Supply-chain Levels for Software Artifacts
- [OWASP Software Supply Chain Security](https://owasp.org/www-project-software-supply-chain-security/)
- [CISA Open Source Software Security](https://www.cisa.gov/open-source-software-security)

---

## 🤝 贡献指南 / Contributing

我们欢迎改进研究指南的贡献！/ We welcome contributions to improve the research guide!

### 如何贡献 / How to Contribute

1. Fork 本仓库 / Fork this repository
2. 创建特性分支 / Create a feature branch (`git checkout -b feature/amazing-guide`)
3. 提交更改 / Commit your changes (`git commit -m 'Add amazing guide section'`)
4. 推送到分支 / Push to the branch (`git push origin feature/amazing-guide`)
5. 开启 Pull Request / Open a Pull Request

### 贡献领域 / Contribution Areas

- 📖 改进研究指南 / Improve research guides
- 🎨 创建架构图和流程图 / Create architecture diagrams
- 🐛 报告问题或建议 / Report issues or suggestions
- 📝 补充文档 / Supplement documentation
- 🔍 深度技术分析 / Deep technical analysis

---

## 📜 许可与声明 / License & Disclaimer

### 重要声明 / Important Notice

- 本仓库是**教育和安全研究档案**，由研究者维护 / This repository is an **educational and security research archive** maintained by researchers
- 原始 Claude Code 源代码版权归 **Anthropic** 所有 / The original Claude Code source code is the property of **Anthropic**
- 本仓库**非 Anthropic 官方仓库**，不受其认可或维护 / This repository is **not an official Anthropic repository**, nor is it endorsed or maintained by Anthropic
- 本仓库存在是为了研究源码泄露、打包失败和现代 agentic CLI 系统架构 / This repository exists to study source exposure, packaging failures, and modern agentic CLI system architecture
- 不得将此仓库中的任何内容用于恶意目的 / Any content in this repository must NOT be used for malicious purposes

### 使用条款 / Terms of Use

访问和使用本仓库即表示您同意：/ By accessing and using this repository, you agree to:

1. 仅将此材料用于教育和安全研究目的 / Use this material ONLY for educational and security research purposes
2. 不侵犯 Anthropic 的知识产权 / Not infringe on Anthropic's intellectual property
3. 遵守所有适用的法律和法规 / Comply with all applicable laws and regulations
4. 对您使用此仓库承担全部责任 / Assume full responsibility for your use of this repository

---

## 👥 维护者 / Maintainers

本仓库由安全研究社区维护。/ This repository is maintained by the security research community.

- **项目维护 / Project Maintenance**: [GitHub Issues](https://github.com/dz3ai/claudecode-leaks/issues)
- **安全报告 / Security Reports**: 请通过 GitHub Issues 私密报告 / Please report privately via GitHub Issues

---

## 📧 联系方式 / Contact

如有疑问或建议，欢迎通过以下方式联系：/ For questions or suggestions:

- 📮 [GitHub Issues](https://github.com/dz3ai/claudecode-leaks/issues)
- 💬 [Discussions](https://github.com/dz3ai/claudecode-leaks/discussions)

---

**最后更新 / Last Updated**: 2026-04-01
**版本 / Version**: 2.0.0
**状态 / Status**: ✅ 活跃维护中 / Actively Maintained

---

<p align="center">
  <sub>⚠️ 再次提醒：本仓库仅供学习和研究使用</sub><br>
  <sub>⚠️ Reminder: This repository is for learning and research purposes only</sub>
</p>
