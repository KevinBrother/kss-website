---
title: "读 DeepSeek dsh-superagent 源码：Agent 真正难的不是写代码，而是知道何时该动手"
date: 2026-11-02
description: "从 GitHub Issue 到独立执行，再到 PR 与后续评论，拆解 dsh-superagent 的 Source、Eval、Mandate、Ticket、Executor 和 Outbox。"
tags: ["AI", "Agent", "DeepSeek", "TypeScript", "架构设计"]
categories: ["技术"]
---

![DeepSeek dsh-superagent 源码阅读封面](/images/deepseek-superagent/cover.svg "DeepSeek dsh-superagent 源码阅读")

最近看了 DeepSeek 开源的 [dsh-superagent-prototype](https://github.com/deepseek-ai/dsh-superagent-prototype)。起初我以为它只是一个「监听 GitHub Issue，自动改代码、提 PR」的工具。

读完 README、领域模型和部分核心实现后，我觉得最值得学的不是怎么调用大模型，而是另一个问题：**当 Agent 可以替整个团队干活时，谁来决定它是否应该干、能干到哪一步、什么时候闭嘴？**

我的判断是：Agent 写代码只是执行环节。长期在团队运行，更难的是**授权边界、持续上下文与人的注意力管理**。这个项目恰好把这几个问题拆得比较清楚。

先说边界：仓库 README 明确写了，这是**研究原型，不是正式 DeepSeek 产品，也不保证可以直接用于生产**。部分研究文章里的设想也未必在本仓库实现。下面是基于公开代码与文档的源码阅读，不是我自己的生产部署记录或性能实测。

## 一、它到底在做什么？

不妨设想一个场景：

1. 你在 GitHub Issue 里向 Agent 提出：「给登录接口补充测试」。
2. 系统先判断：发起人是否被允许要求它做事？请求是不是明确指向 Agent？
3. 请求通过后，系统建立一项独立工作，Agent 在自己的 Session 和目录里完成代码、测试，必要时创建 PR。
4. Reviewer 在 PR 里补一句「这里要考虑 token 过期」，这条评论能回到**原来的任务**，而不是重新生成一个无关的 Agent。
5. Agent 只在结果可用、需要人决策或遇到阻塞时回复。

这是解释设计的示例，并不意味着默认对所有 Issue 自动开始工作。是否接收、是否自动接受，取决于来源、用户与 Policy 配置。

系统的输入不只是 GitHub，也可以选配飞书。重点不是多接一个聊天入口，而是**把事件来源和领域决策分离**。

![dsh-superagent 从请求到交付的简化架构](/images/deepseek-superagent/architecture.svg "从 Source 到 Executor 的流程")

## 二、先搞懂几个名词，不然源码容易看乱

仓库的 [领域模型文档](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/001-domain.md) 定义了这些术语：

| 名词 | 负责什么 | 我的理解 |
| --- | --- | --- |
| Source | 接收 GitHub、飞书等外部事件 | 入口 |
| Trigger | 一次不可变的事件，例如评论或 Issue 更新 | 事件 |
| Thread | 同一条讨论及其上下文 | 会话线 |
| Eval | 评估是否发起新工作 | 判断 |
| Policy | 谁能发起、在什么条件下执行 | 规则 |
| Mandate | 某次接受所授予的工作范围 | 授权凭据 |
| Ticket | 一项被授权的工作及其生命周期 | 任务 |
| Assignment | Ticket 和一次执行实例的关系 | 本次分派 |
| Subscription | 让后续事件回到已有关联任务 | 订阅 |
| Channel / Outbox | 把消息和结果可靠地发回去 | 回传 |

最容易混的是 **Ticket 和 Assignment**。

Ticket 代表「这件事」；Assignment 代表「这一次谁、在哪个执行实例上做它」。同一项工作可能因为失败重试、进程重启而经历多个 Assignment。若把两者混成一条记录，恢复、重试和审计就会很麻烦。

另一个容易混的是 **Policy 和 Mandate**：Policy 是规则，Mandate 是某次接受时授予的工作范围。后续评论带来的新指令不能自动扩大授权。

## 三、为什么新请求和后续评论不是一条路？

这是整个实现我最想单独讲的设计。

![新请求与后续评论的两条处理路径](/images/deepseek-superagent/followups.svg "Eval 与 Subscription 两条不同路线")

### 新请求：先 Eval，再决定是否创建 Ticket

从 [src/core/engine.ts](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/src/core/engine.ts) 看，外部事件进入 Engine 的 receive 方法，实际接收逻辑交给 Intake。符合条件、需要发起新工作的 Trigger 进入 Eval。

系统综合发起人、Source 的 Policy、已有 Ticket 和 Thread 上下文，最后记录 Decision。

**不是收到一条评论就开一个 Worker。** 否则热闹的 Issue 可能被拆成很多重复任务。

### 后续评论：优先回到已有 Ticket

假设 Agent 已为某个 Issue 开了 PR，Reviewer 又提出修改意见。如果这条讨论已经被 Ticket 订阅，后续事件会回到原任务的 Executor，不必重新开始。

但它仍然受原 Mandate 限制。如果评论要求的是超出任务范围的新工作，Executor 可以交还请求，让系统再评估。

**Subscription 解决的是上下文，不是授权。** CI 和机器人消息可以提供上下文，却不会仅凭送达就获得新的执行权限。

阅读入口：[intake.ts](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/src/core/intake.ts)、[evals.ts](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/src/core/evals.ts)、[tickets.ts](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/src/core/tickets.ts)。

## 四、Engine 为什么不直接去调 GitHub API？

在 [src/app/runtime.ts](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/src/app/runtime.ts) 中，Runtime 负责组装 Store、Sources、Assessor、Executor、Effects、Outbox 等组件。

而 Engine 偏向于领域状态的唯一变更入口，接收事件、排队评估、记录决定、绑定讨论、推进任务。

这里的关键不是某个复杂的 AI 算法，而是**把「修改领域状态」与「真正执行外部操作」分开**。

[架构文档](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/007-architecture.md) 解释，Engine 在事务中修改领域数据；外部工作先排队，提交后再由适配器执行。

试想一个失败场景：系统已经在 GitHub 发出确认消息，却没来得及记录成功；重启后再次发送。借助持久化的 Outbox、投递记录和重试规则，就能显式管理这类问题。

当然，这不代表所有 GitHub API 自动获得严格的 exactly-once 语义。仍要结合外部 API、幂等与重试策略理解。

如果熟悉后端架构，这其实很像「事务 + Outbox + Worker」。只不过这里的 Worker 是一个会阅读代码、调用工具的 Agent Session。

## 五、Executor：一个 Ticket 并不等于一个线程

我一开始也有疑问：它是不是每收到一个 Issue 就开一个 Node.js 线程？

不能这样理解。

根据 [执行文档](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/003-execution.md)，每个 Ticket 都会关联 Executor Session 和独立工作目录；Assignment 表示某次具体执行。并发受 worker slots 约束，执行期间可能有后台 shell 命令或子 Agent。

有几个细节比较值得借鉴：

- 一个执行回合结束，不代表后台工作完成；后台作业仍可能占据 slot。
- 需要人决策时，可以交接并释放 slot，不必一直空转。
- 执行失败或进程重启后，系统依靠 Ticket、Assignment 和 Input 的记录决定如何恢复。
- 新工作的并发受到约束，同时考虑人类回复已有任务时的唤醒需求。

所以它更像**持久化任务调度系统 + 多个 Agent Session**，不是遍历 Issue 后同时跑几个 Promise。

## 六、Prompt 是策略的一部分，但不能代替权限

项目把团队沟通方式、任务完成标准、Skill 与内置 Prompt 分开维护。比如什么时候向人发消息、在什么条件下交付 PR，以及需要怎样的证据。

但**Prompt 告诉 Agent 应该怎么做，代码里的授权校验决定它被允许做什么**。

如果把「不要响应未授权请求」仅仅写进 System Prompt，遇到恶意评论或 Prompt Injection 时，边界很脆弱。

本项目通过来源接入规则、Policy、Mandate 和工具权限约束执行范围。相关内容可读 [授权文档](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/002-authorization.md) 与 [Prompt 文档](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/004-prompts.md)。

## 七、我会怎样阅读它的代码？

如果要亲自学习这个项目，我不会一上来读遍每个 TypeScript 文件。我会按调用路线走：

1. **理念和术语**：先读 [000 Philosophy](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/000-philosophy.md) 与 [001 Domain](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/001-domain.md)。
2. **组装入口**：读 [src/index.ts](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/src/index.ts) 和 [src/app/runtime.ts](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/src/app/runtime.ts)，理解组件怎么连接。
3. **主流程**：沿着 Engine.receive → Intake → Evals → Tickets → Assignments 追踪一次请求的状态变化。
4. **实际执行**：读 [src/executor/dsh.ts](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/src/executor/dsh.ts)。
5. **可靠回传**：读 [src/app/outbox.ts](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/src/app/outbox.ts) 与 [src/app/effects.ts](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/src/app/effects.ts)。
6. **最后再部署**：照 [SETUP.md](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/SETUP.md) 和 [006 Deployment](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/006-deployment.md) 理解准备工作。

如果只是学架构，没必要先赋予真实生产仓库的写权限。先搞清 Ticket、Mandate 和 Assignment 在何时创建、何时结束，收获已经很大了。

## 八、哪些设计值得借鉴，哪些没必要照搬？

我最想借鉴的是三件事：

- **新请求与后续消息分流**：避免同一个讨论被当作无数次新任务。
- **授权与执行实例分离**：重试和重启不会随便扩大已有任务的范围。
- **对外回复可追踪**：把消息发送视为需要持久化、恢复和审计的工作。

但如果场景只是每天扫描仓库并汇总 Issue，简单的 GitHub Actions 或定时任务就够了。

只有当系统确实需要多来源事件、长期跟进、并发执行、人类中途反馈和失败恢复时，Ticket / Subscription / Assignment 这一套才更有价值。

此外，原项目还是研究原型，实际可用性、安全边界、成本与稳定性需要自己验证，不能只因为它能运行就拿去生产。

## 九、几个容易混淆的问题

**Q：它和 GitHub Actions 有什么区别？**

GitHub Actions 主要执行事先确定的流程；这里更关心「一个请求是否应该变成工作、允许做多大、如何持续接收后续讨论」。两者可以结合使用，CI 结果也可成为 Agent 的反馈来源。

**Q：每条新评论都需要再跑一次 Eval 吗？**

不一定。已订阅 Thread 的后续讨论可以直接到已有 Ticket；范围外的新工作则可能被交还重新判断。

**Q：Agent 能否自动合并 PR？**

项目有相关可配置机制，但不是说默认就自动合并。SETUP 文档建议初始阶段先由人合并，确认结果可靠后再考虑放宽。

**Q：「主动 Agent」是不是可以随便找事做？**

不是。项目的核心原则是仅响应明确请求或团队授权的书面规则。对推断出的主动工作，SETUP 推荐从 observe 模式开始，先记录本来会执行什么，而不直接动手。

**Q：如果只能读一个源码文件，先读哪里？**

我会选 [src/core/engine.ts](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/src/core/engine.ts)。它是主要领域入口，能帮我们顺着方法找到状态变化与具体实现。

## 最后

看完这套设计，我对「让 Agent 更自主」的理解反而更克制了：不是给它更多工具、更多并发和更多自动触发，而是让它**在有授权时持续完成一项工作，在需要人时明确交接，在不需要人时尽量安静**。

这比做一个会自动提交代码的机器人难得多，也更值得从源码里学。

> 主要参考：[项目 README](https://github.com/deepseek-ai/dsh-superagent-prototype) · [领域模型](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/001-domain.md) · [授权](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/002-authorization.md) · [执行](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/003-execution.md) · [架构](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/007-architecture.md)
