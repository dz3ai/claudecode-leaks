# Claude Code 源代码泄露研究指南编写计划

## 项目概述

为 claudecode-leaks 项目创建双语（中文/英文）代码研究指南，帮助安全研究者、开发者和学术界理解这次泄露事件的技术细节和架构价值。

**目标受众：**
- 安全研究人员（软件供应链安全）
- AI 工具开发者
- 构建系统和包管理器维护者
- 学术研究者
- 开源社区贡献者

## 指南结构设计

### 中文版指南结构

**文件名：** `docs/guide/代码研究指南.md`

```markdown
# Claude Code 源代码泄露研究指南

## 第一部分：事件背景
- 1.1 泄露时间线
- 1.2 泄露机制（source map 暴露）
- 1.3 影响范围评估
- 1.4 研究价值与伦理考量

## 第二部分：快速导航
- 2.1 按研究主题导航
- 2.2 按技术栈导航
- 2.3 推荐阅读路径（初学者/进阶/专家）

## 第三部分：核心架构解析
- 3.1 工具系统（Tool System）
  - 3.1.1 工具接口定义
  - 3.1.2 权限模型
  - 3.1.3 核心工具实现案例
- 3.2 查询引擎（QueryEngine）
  - 3.2.1 LLM 交互循环
  - 3.2.2 流式响应处理
  - 3.2.3 工具调用协调
- 3.3 命令系统（Command System）
- 3.4 权限系统（Permission System）
- 3.5 桥接系统（Bridge System）

## 第四部分：关键技术专题
- 4.1 React + Ink 终端 UI 架构
- 4.2 Bun 特性使用（feature flags, bundle）
- 4.3 MCP 协议集成
- 4.4 LSP 集成架构
- 4.5 OAuth 2.0 + JWT 认证流程
- 4.6 OpenTelemetry 遥测
- 4.7 多代理协调（Multi-Agent）

## 第五部分：供应链安全 lessons
- 5.1 Source Map 风险详解
- 5.2 构建配置最佳实践
- 5.3 CI/CD 安全检查点
- 5.4 包发布验证流程

## 第六部分：代码阅读实战
- 6.1 案例1：从用户输入到工具执行
- 6.2 案例2：权限检查流程
- 6.3 案例3：流式响应处理
- 6.4 案例4：MCP 服务器连接

## 第七部分：研究资源
- 7.1 关键文件索引
- 7.2 代码交叉引用
- 7.3 架构图解
- 7.4 扩展阅读建议
```

### 英文版指南结构

**文件名：** `docs/guide/Code-Research-Guide.md`

```markdown
# Claude Code Source Leak Research Guide

## Part 1: Incident Background
- 1.1 Timeline of Events
- 1.2 Leak Mechanism (Source Map Exposure)
- 1.3 Impact Assessment
- 1.4 Research Value and Ethics

## Part 2: Quick Navigation
- 2.1 Navigate by Research Topic
- 2.2 Navigate by Tech Stack
- 2.3 Recommended Reading Paths (Beginner/Advanced/Expert)

## Part 3: Core Architecture Analysis
- 3.1 Tool System
  - 3.1.1 Tool Interface Definition
  - 3.1.2 Permission Model
  - 3.1.3 Core Tool Implementation Examples
- 3.2 Query Engine
  - 3.2.1 LLM Interaction Loop
  - 3.2.2 Streaming Response Handling
  - 3.2.3 Tool Call Coordination
- 3.3 Command System
- 3.4 Permission System
- 3.5 Bridge System

## Part 4: Key Technical Topics
- 4.1 React + Ink Terminal UI Architecture
- 4.2 Bun Features (Feature Flags, Bundle)
- 4.3 MCP Protocol Integration
- 4.4 LSP Integration Architecture
- 4.5 OAuth 2.0 + JWT Authentication Flow
- 4.6 OpenTelemetry Telemetry
- 4.7 Multi-Agent Coordination

## Part 5: Supply Chain Security Lessons
- 5.1 Source Map Risks Explained
- 5.2 Build Configuration Best Practices
- 5.3 CI/CD Security Checkpoints
- 5.4 Package Release Verification

## Part 6: Code Reading Practice
- 6.1 Case 1: From User Input to Tool Execution
- 6.2 Case 2: Permission Check Flow
- 6.3 Case 3: Streaming Response Handling
- 6.4 Case 4: MCP Server Connection

## Part 7: Research Resources
- 7.1 Key File Index
- 7.2 Code Cross-References
- 7.3 Architecture Diagrams
- 7.4 Further Reading
```

## 编写计划

### 阶段 1：基础框架搭建（1-2 小时）

**任务 1.1：创建文档目录结构**
```bash
mkdir -p docs/guide
mkdir -p docs/guide/zh
mkdir -p docs/guide/en
mkdir -p docs/diagrams
```

**任务 1.2：编写中英文目录框架**
- 创建两个版本的 Markdown 文件
- 建立统一的章节编号系统
- 确保双语章节对应

**任务 1.3：创建索引文件**
- `docs/guide/README.md` - 指南导航页
- 双语切换说明
- 快速查找表

### 阶段 2：核心内容编写（4-6 小时）

**任务 2.1：事件背景部分（1 小时）**
- 整理时间线数据
- 说明泄露机制
- 评估影响范围
- 中英文同步编写

**任务 2.2：快速导航部分（30 分钟）**
- 按研究主题分类（工具系统/权限系统/供应链安全等）
- 按技术栈分类（React/Bun/MCP/LSP 等）
- 设计 3 条阅读路径

**任务 2.3：核心架构解析（2-3 小时）**

这是最重要的部分，需要深入代码：

3.1 工具系统
- 阅读 `src/Tool.ts`，理解工具接口
- 阅读 `src/tools/` 下核心工具实现
- 分析权限模型实现
- 提供代码示例和流程图

3.2 查询引擎
- 深入分析 `src/QueryEngine.ts`（46KB 核心文件）
- 理解 LLM 交互循环
- 分析流式响应处理
- 绘制执行流程图

3.3-3.5 其他核心系统
- 命令系统：`src/commands.ts` + 示例命令
- 权限系统：`src/hooks/toolPermission/`
- 桥接系统：`src/bridge/`

**任务 2.4：关键技术专题（2 小时）**
- 每个主题需要阅读相关源码
- 提供架构说明
- 代码片段示例
- 设计要点总结

### 阶段 3：安全专题编写（2-3 小时）

**任务 3.1：Source Map 风险详解（1 小时）**
- 解释 source map 机制
- 分析为何会暴露源码
- 提供检测和修复方法
- 中英文对照

**任务 3.2：构建配置最佳实践（1 小时）**
- 安全的打包配置示例
- CI/CD 检查点代码示例
- 包发布验证流程

**任务 3.3：代码阅读实战（1 小时）**
- 选择 4 个典型案例
- 绘制调用流程图
- 提供追踪路径
- 解释关键代码片段

### 阶段 4：研究资源整理（1 小时）

**任务 4.1：关键文件索引**
- 按功能分类所有重要文件
- 提供文件路径和简要说明
- 创建交叉引用表

**任务 4.2：架构图解**
- 创建系统架构图（使用 Mermaid 或 Graphviz）
- 绘制数据流图
- 绘制组件关系图

**任务 4.3：扩展阅读**
- 相关技术文档链接
- 安全标准和框架（SLSA 等）
- 学术研究论文参考

### 阶段 5：审校与优化（1-2 小时）

**任务 5.1：中英文对照检查**
- 确保术语一致
- 检查翻译准确性
- 统一格式和风格

**任务 5.2：代码示例验证**
- 确保所有代码片段可读
- 验证文件路径正确
- 检查行号引用准确

**任务 5.3：交叉链接**
- 添加章节间链接
- 创建快速跳转索引
- 补充术语表

## 时间估算

| 阶段 | 任务 | 预计时间 | 优先级 |
|------|------|----------|--------|
| 1 | 基础框架搭建 | 1-2 小时 | 高 |
| 2 | 核心内容编写 | 4-6 小时 | 高 |
| 3 | 安全专题编写 | 2-3 小时 | 高 |
| 4 | 研究资源整理 | 1 小时 | 中 |
| 5 | 审校与优化 | 1-2 小时 | 中 |

**总计：9-14 小时**

## 交付成果

### 主要文档
1. `docs/guide/代码研究指南.md` - 中文完整指南（预计 8,000-12,000 字）
2. `docs/guide/Code-Research-Guide.md` - 英文完整指南（预计 5,000-8,000 词）

### 辅助资源
3. `docs/guide/README.md` - 导航索引
4. `docs/diagrams/` - 架构图和流程图
5. `docs/guide/key-files-index.md` - 关键文件索引
6. `docs/guide/terminology.md` - 中英术语对照表

### 可选扩展
7. `docs/guide/quick-start.md` - 快速开始指南
8. `docs/guide/research-topics.md` - 研究主题建议
9. `docs/guide/contribution-guide.md` - 贡献指南

## 质量标准

### 内容质量
- ✅ 所有技术细节准确，引用实际代码
- ✅ 架构描述清晰，易于理解
- ✅ 代码示例完整，可独立理解
- ✅ 安全建议实用，可直接应用

### 语言质量
- ✅ 中文流畅专业，术语准确
- ✅ 英文地道自然，符合技术文档规范
- ✅ 双语内容对等，无遗漏

### 文档质量
- ✅ 结构清晰，层次分明
- ✅ 交叉链接完善，便于导航
- ✅ 图表丰富，辅助理解
- ✅ 格式统一，排版美观

## 使用场景

### 研究者使用指南
1. **快速了解事件** → 阅读第一部分
2. **学习架构设计** → 按第三部分深入学习
3. **研究特定技术** → 查阅第四部分对应专题
4. **供应链安全研究** → 重点阅读第五部分
5. **代码阅读实践** → 跟随第六部分案例

### 讲师使用指南
1. **准备课程材料** → 使用各章节作为讲义基础
2. **案例教学** → 使用第六部分的代码阅读案例
3. **实验指导** → 参考第四部分技术专题设计实验
4. **安全培训** → 使用第五部分作为供应链安全教材

## 下一步行动

**立即开始：**
1. 创建目录结构
2. 编写第一部分（事件背景）
3. 建立中英文对照框架

**并行进行：**
- 深入阅读核心源码（Tool.ts, QueryEngine.ts）
- 收集关键代码片段
- 绘制初步架构图

**顺序完成：**
- 按章节顺序编写核心内容
- 最后整理研究资源和索引

## 维护计划

- 每月更新一次，补充新发现的研究价值
- 根据社区反馈优化内容
- 添加新的代码阅读案例
- 更新安全实践建议

---

**创建日期：** 2026-03-31
**预计完成：** 2026-04-02
**状态：** 待执行
