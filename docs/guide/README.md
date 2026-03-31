# Claude Code 源代码泄露研究指南
# Claude Code Source Leak Research Guide

> 双语代码研究指南 | Bilingual Code Research Guide

## 快速开始 | Quick Start

**中文读者：** 请查看 [代码研究指南.md](./代码研究指南.md)
**English Readers:** Please see [Code-Research-Guide.md](./Code-Research-Guide.md)

## 按主题导航 | Navigate by Topic

### 核心架构 | Core Architecture
- [工具系统 | Tool System](./代码研究指南.md#31-工具系统-tool-system)
- [查询引擎 | Query Engine](./代码研究指南.md#32-查询引擎-queryengine)
- [命令系统 | Command System](./代码研究指南.md#33-命令系统-command-system)
- [权限系统 | Permission System](./代码研究指南.md#34-权限系统-permission-system)
- [桥接系统 | Bridge System](./代码研究指南.md#35-桥接系统-bridge-system)

### 技术专题 | Technical Topics
- [React + Ink 终端 UI](./代码研究指南.md#41-react--ink-终端-ui-架构)
- [Bun 特性使用](./代码研究指南.md#42-bun-特性使用feature-flags-bundle)
- [MCP 协议集成](./代码研究指南.md#43-mcp-协议集成)
- [LSP 集成架构](./代码研究指南.md#44-lsp-集成架构)
- [OAuth 2.0 + JWT 认证](./代码研究指南.md#45-oauth-20--jwt-认证流程)
- [OpenTelemetry 遥测](./代码研究指南.md#46-opentelemetry-遥测)
- [多代理协调](./代码研究指南.md#47-多代理协调multi-agent)

### 安全研究 | Security Research
- [Source Map 风险详解](./代码研究指南.md#51-source-map-风险详解)
- [构建配置最佳实践](./代码研究指南.md#52-构建配置最佳实践)
- [CI/CD 安全检查点](./代码研究指南.md#53-cicd-安全检查点)
- [包发布验证流程](./代码研究指南.md#54-包发布验证流程)

## 按阅读路径导航 | Navigate by Reading Path

### 初学者路径 | Beginner Path (4-6 hours)
1. 事件背景 → 了解泄露机制
2. 快速导航 → 建立整体认知
3. 工具系统 → 理解核心概念
4. 案例实践 → 跟随代码阅读

### 进阶路径 | Advanced Path (8-12 hours)
1. 完整阅读核心架构部分
2. 深入研究 2-3 个技术专题
3. 实践所有 4 个代码案例
4. 学习供应链安全 lessons

### 专家路径 | Expert Path (16-20 hours)
1. 完整阅读所有章节
2. 分析所有工具实现（43 个工具）
3. 研究所有命令实现（110 个命令）
4. 对比其他 AI Agent 架构
5. 提炼可复用的设计模式

## 关键文件索引 | Key Files Index

| 文件 | 大小 | 说明 | 优先级 |
|------|------|------|--------|
| `src/main.tsx` | 786KB | 应用入口点 | ⭐⭐⭐⭐⭐ |
| `src/QueryEngine.ts` | 46KB | LLM 查询引擎核心 | ⭐⭐⭐⭐⭐ |
| `src/Tool.ts` | 29KB | 工具类型定义 | ⭐⭐⭐⭐⭐ |
| `src/commands.ts` | 25KB | 命令注册表 | ⭐⭐⭐⭐ |
| `src/tools.ts` | 17KB | 工具注册表 | ⭐⭐⭐⭐ |
| `src/hooks/toolPermission/` | - | 权限系统实现 | ⭐⭐⭐⭐⭐ |

## 术语对照表 | Terminology

| 中文 | English |
|------|---------|
| 工具系统 | Tool System |
| 查询引擎 | Query Engine |
| 权限模型 | Permission Model |
| 源映射 | Source Map |
| 软件供应链 | Software Supply Chain |
| 模型上下文协议 | Model Context Protocol (MCP) |
| 语言服务器协议 | Language Server Protocol (LSP) |
| 开放遥测 | OpenTelemetry |

## 研究资源 | Research Resources

- [架构图](../diagrams/) - Architecture Diagrams
- [关键代码片段](../snippets/) - Code Snippets
- [研究主题建议](./research-topics.md) - Research Topics
- [扩展阅读](./further-reading.md) - Further Reading

## 贡献指南 | Contributing

欢迎改进研究指南！请参阅 [贡献指南](./contributing.md)

## 许可与声明 | License & Disclaimer

本指南仅用于教育和安全研究目的。泄露的源代码版权归 Anthropic 所有。

This guide is for educational and security research purposes only. The leaked source code is the property of Anthropic.

---

**最后更新 | Last Updated:** 2026-03-31
**版本 | Version:** 1.0.0
