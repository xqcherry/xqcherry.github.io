---
title: 什么是Spec Coding？
date: 2026-08-22 03:33:00
categories:
  - 大模型
tags:
  - Agent
  - Spec
---


# Spec Coding、Superpowers 与 OpenSpec

## 一、什么是 Spec Coding？

**Spec Coding（Specification-driven Coding）**，通常指 **Spec-Driven Development（SDD，规格驱动开发）**：

> **先明确“系统应该做什么”，再让 AI 按照规格实现，而不是直接从模糊需求跳到写代码。**

传统 AI Coding：

```text
需求
 ↓
Prompt
 ↓
AI 自己理解需求
 ↓
写代码
 ↓
发现问题
 ↓
继续修改
```

Spec Coding：

```text
需求
 ↓
Spec（规格）
 ↓
人和 AI 对齐
 ↓
Plan / Design
 ↓
实现
 ↓
测试 / 验证
```

### 为什么需要 Spec？

因为 AI Coding 最大的问题之一不是“不会写代码”，而是：

> **AI 不知道你真正想要什么。**

比如：

```text
“给登录系统增加 2FA”
```

这个需求其实有很多未确定的问题：

* 什么情况下要求 2FA？
* 使用什么认证方式？
* OTP 还是其他方式？
* 登录接口怎么变化？
* 老用户怎么办？
* 失败怎么处理？
* 是否需要修改数据库？
* 如何测试？

如果直接让 AI Coding，AI 会自行补全这些信息。

Spec Coding 则要求：

```text
Requirement:
用户开启 2FA 后，登录必须完成第二因素验证。

Scenario:
WHEN 用户输入正确用户名和密码
AND 用户已开启 2FA
THEN 系统要求输入 OTP
```

这样 AI 的实现空间就被明确约束了。

---

# 二、Spec Coding 的核心思想

可以记住三个关键词：

### 1. Intent

明确：

> **我要什么？**

### 2. Specification

明确：

> **系统具体应该表现成什么样？**

### 3. Verification

明确：

> **怎么证明 AI 实现对了？**

因此：

```text
Intent
  ↓
Spec
  ↓
Implementation
  ↓
Verification
```

Spec 本质上是 **人和 AI 之间的契约**。

---

# 三、Spec Coding 不等于“写一个 Markdown 文档”

这是一个很重要的区别。

普通文档：

```text
我们要做一个登录系统。
登录之后用户可以访问个人中心。
```

Spec 更强调**可验证的行为**：

```text
Requirement:
用户成功登录后，系统 SHALL 返回 JWT。

Scenario:
WHEN 用户提交正确的用户名和密码
THEN 返回 HTTP 200
AND response 中包含 JWT。
```

所以好的 Spec 通常具有：

* 明确的行为
* 明确的边界
* 明确的场景
* 可以转换成测试
* 可以被 AI 执行

---

# 四、Superpowers 是什么？

[Superpowers GitHub](https://github.com/obra/Superpowers?utm_source=chatgpt.com)

**Superpowers** 是一个面向 Coding Agent 的 **Agentic Skills Framework + 软件开发方法论**。

它不是单纯的 Spec 管理工具。

它更像是在告诉 Agent：

> **“你应该按照什么工程流程来开发软件。”**

Superpowers 把很多工程实践封装成 Agent Skills，例如：

```text
brainstorming
       ↓
writing-plans
       ↓
executing-plans
       ↓
TDD
       ↓
code review
       ↓
verification
       ↓
finish branch
```

官方强调的核心理念包括：

* TDD
* Systematic over ad-hoc
* Complexity reduction
* Evidence over claims

也就是：

> **不要猜，先分析；不要直接写，先设计；不要声称完成，要验证。**

---

# 五、Superpowers 的工作方式

假设你告诉 Agent：

```text
我要给项目增加用户搜索功能。
```

Superpowers 不希望 Agent 直接：

```text
开始写代码
```

而是：

```text
用户需求
 ↓
Brainstorming
 ↓
明确需求和设计
 ↓
Writing Plan
 ↓
拆成具体任务
 ↓
TDD
 ↓
实现
 ↓
Review
 ↓
Verification
```

它甚至支持 **subagent-driven development**：

```text
主 Agent
   ↓
拆任务
   ↓
Subagent 处理 Task 1
   ↓
Review
   ↓
Subagent 处理 Task 2
   ↓
Review
   ↓
...
```

官方描述中，这种流程会进行两阶段 review：

```text
1. Spec Compliance
   ↓
是否实现了需求？

2. Code Quality
   ↓
代码质量是否合格？
```

---

# 六、OpenSpec 是什么？

[OpenSpec GitHub](https://github.com/Fission-AI/OpenSpec?utm_source=chatgpt.com)

**OpenSpec** 是一个专门面向 AI Coding Assistant 的 **Spec-Driven Development 框架**。

它解决的问题非常明确：

> **让人和 AI 在写代码之前，先对“要实现什么”达成一致。**

它会把规格和变更真正放进项目：

```text
project/
│
├── src/
├── tests/
│
└── openspec/
    ├── specs/
    │
    └── changes/
```

其中最重要的是：

```text
openspec/specs/
```

表示：

> **当前系统的规格 / Truth**

而：

```text
openspec/changes/
```

表示：

> **准备对系统做什么修改**

---

# 七、OpenSpec 的典型流程

现在 OpenSpec 的工作流可以理解为：

```text
探索
 ↓
Proposal
 ↓
Spec
 ↓
Design
 ↓
Tasks
 ↓
Implementation
 ↓
Verification
 ↓
Archive
```

例如：

```text
/opsx:explore
```

先让 Agent 阅读项目、分析问题。

确定需求之后：

```text
/opsx:propose add-dark-mode
```

生成：

```text
openspec/changes/add-dark-mode/
├── proposal.md
├── specs/
├── design.md
└── tasks.md
```

然后：

```text
/opsx:apply
```

开始实现。

最后：

```text
/opsx:archive
```

把完成后的变更归档，并更新长期维护的 Specs。

---

# 八、OpenSpec 最重要的设计

OpenSpec 有一个非常值得理解的思想：

```text
Current Truth
     │
     │
     ▼
openspec/specs/
     │
     │ proposed change
     ▼
openspec/changes/
     │
     │ implementation
     ▼
代码
     │
     │ archive
     ▼
更新后的 specs/
```

也就是说：

> **Spec 不是一次性 Prompt，而是项目长期存在的“系统行为描述”。**

例如项目现在：

```text
用户登录需要密码
```

后来增加 2FA。

Change：

```text
add-2fa/
```

描述：

```text
MODIFIED:
登录流程

ADDED:
OTP 验证
```

实现完成以后，把这个变化合并进系统的正式 Spec。

所以随着项目不断发展：

```text
Spec v1
 ↓
Change A
 ↓
Spec v2
 ↓
Change B
 ↓
Spec v3
 ↓
...
```

这样 AI 下一次接手项目时，可以先理解：

> **这个系统现在到底是什么状态。**

---

# 九、Superpowers 和 OpenSpec 的区别

这是最容易混淆的地方。

可以简单理解成：

```text
Superpowers
=
“Agent 应该怎么工作？”

OpenSpec
=
“系统应该是什么，以及这次要改什么？”
```

|                 | Superpowers          | OpenSpec                   |
| --------------- | -------------------- | -------------------------- |
| 定位              | Agent 开发方法论 / Skills | Spec-Driven Development 框架 |
| 核心              | 怎么开发                 | 规定开发什么                     |
| 重点              | Workflow             | Specification              |
| TDD             | 强调                   | 可以结合                       |
| Code Review     | 有                    | 可以结合                       |
| Subagent        | 有                    | 不是核心                       |
| Spec            | 有需求分析过程              | 核心能力                       |
| 长期项目知识          | 不是主要目标               | `openspec/specs/`          |
| Change Tracking | 不是核心                 | 核心                         |
| 适合              | 规范 Agent 行为          | 管理需求和系统规格                  |

---

# 十、两者其实可以一起用

它们不是竞争关系。

可以组合：

```text
              用户
               │
               ▼
        OpenSpec / Spec
               │
       “到底要实现什么？”
               │
               ▼
          Superpowers
               │
       “应该怎么实现？”
               │
               ▼
        Plan / TDD / Agent
               │
               ▼
             Code
               │
               ▼
          Verification
```

例如：

```text
OpenSpec：
我要增加 Windows Shift+Enter 换行。

       ↓

Spec：
Shift+Enter → newline
Enter → submit
Linux/macOS → 不受影响

       ↓

Superpowers：
先分析代码
→ 写测试
→ 制定 Plan
→ 实现
→ Review
→ 验证

       ↓

代码
```

所以可以把二者理解成：

> **OpenSpec 管“目标和规格”，Superpowers 管“Agent 的工程执行过程”。**

---

# 十一、和普通 Plan 模式有什么区别？

普通 Agent：

```text
Prompt
 ↓
Plan
 ↓
Code
```

Plan 通常是：

> **为了完成当前任务临时生成的执行计划。**

而 Spec：

> **描述系统应该具有什么行为，并且可以长期保存。**

所以：

```text
Plan
= How do I do this?

Spec
= What should the system do?

Design
= How should we build it?

Code
= Actual implementation
```

这是理解 Spec Coding 最重要的一组概念。

---

# 十二、最终可以这样记

```text
Spec Coding
│
├── Spec
│   └── 定义“系统应该是什么”
│
├── OpenSpec
│   └── 管理 Spec、Change、Design、Tasks
│
└── Superpowers
    └── 规范 Agent 如何分析、计划、编码、测试、Review
```

