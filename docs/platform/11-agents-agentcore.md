# Bedrock Agents / AgentCore 讲义（把会回答升级为会执行）

## 本周目标
第11周结束后你应该能做到：
1. 解释 Agent 与普通聊天模型的本质区别。
2. 理解 Bedrock Agents 的核心组件：Instruction、Action Group、Knowledge Base、Orchestration。
3. 用 AgentCore 快速搭一个可运行的内部运维助手原型。
4. 为 Agent 工具调用建立权限边界与安全审查链路（结合第10周 Guardrails）。
5. 在公司里听懂并使用：`tool calling`、`function schema`、`agent plan`、`grounding`、`handoff`。

---

## 先记一句话
Agent 不是“更会聊天”，而是“会按目标调用工具完成任务”。

---

## 讲义知识地图（模块）
1. 模块1：Agent 架构与工作流。
2. 模块2：Bedrock Agent 基础配置（Instruction + Session）。
3. 模块3：Action Group（工具函数定义与调用）。
4. 模块4：Knowledge Base 与检索增强（RAG）接入。
5. 模块5：AgentCore 快速脚手架与端到端 Demo。
6. 模块6：权限、安全、观测与故障排查。
7. 模块7：沉淀“内部运维助手”最小可交付版本。

---

## 模块1：Agent 基础架构

## 1.1 Agent 与普通 LLM 的区别
1. 普通 LLM：输入 -> 输出文本。
2. Agent：输入 -> 规划 -> 调工具 -> 组合结果 -> 输出。

Agent 核心能力：
1. 任务分解。
2. 工具选择。
3. 多步执行。

## 1.2 你要理解的链路
`User Query -> Agent Plan -> Tool Calls -> Result Aggregation -> Final Response`

## 1.3 适合 Agent 的场景
1. 需要查多个系统数据。
2. 需要执行操作（读状态、触发任务）。
3. 需要按步骤完成任务（而不是一次性问答）。

## 1.4 不适合 Agent 的场景
1. 纯静态知识问答。
2. 时延极致敏感且无需工具调用。

## 1.5 模块1 验收
1. 你能解释 Agent 价值边界。
2. 你能说出你们 infra 组 2 个适合 Agent 的用例。

---

## 模块2：Bedrock Agent 基础配置

## 2.1 Agent 最小配置项
1. Agent 指令（Instruction）。
2. 可用模型。
3. 会话策略（session memory 范围）。
4. Guardrails 关联（建议接入）。

## 2.2 Instruction 编写模板
```text
你是内部运维助手。
职责：帮助排查 EKS 与 CI/CD 问题。
边界：不执行破坏性操作，不输出未验证结论。
流程：先确认上下文，再调用工具，再给出结论和证据。
输出格式：
1) 结论
2) 证据
3) 下一步建议
```

## 2.3 会话策略建议
1. 限制会话上下文窗口，避免成本失控。
2. 关键操作前要求显式确认。
3. 系统提示与用户提示分离管理。

## 2.4 模块2 验收
1. 你能写出一份可执行 instruction。
2. 你能解释为什么 Agent 也需要 Guardrails。

---

## 模块3：Action Group（工具调用）

## 3.1 Action Group 是什么
Action Group 是 Agent 可调用工具的集合，通常映射到 API 或函数。

## 3.2 工具 schema 设计原则
1. 输入参数明确（类型、必填、范围）。
2. 输出结构稳定（便于模型解析）。
3. 错误码可判断（可重试/不可重试）。

## 3.3 示例：运维查询工具
1. `get_service_status(service_name, env)`
2. `get_recent_errors(service_name, minutes)`
3. `get_deploy_history(service_name, env)`

## 3.4 示例函数 schema（概念）
```json
{
  "name": "get_service_status",
  "description": "Query service health status",
  "inputSchema": {
    "type": "object",
    "properties": {
      "service_name": {"type": "string"},
      "env": {"type": "string", "enum": ["dev", "stage", "prod"]}
    },
    "required": ["service_name", "env"]
  }
}
```

## 3.5 工具调用安全边界
1. 默认只读工具优先。
2. 写操作必须显式确认。
3. 高风险操作必须人工审批。

## 3.6 模块3 验收
1. 你能设计 3 个可落地工具 schema。
2. 你能解释“为什么 schema 设计不好会导致 Agent 乱调用”。

---

## 模块4：Knowledge Base（RAG）

## 4.1 为什么要 Knowledge Base
Agent 如果只靠模型记忆，容易幻觉。接入知识库可让回答“有来源可依”。

## 4.2 可接入资料类型
1. Runbook 文档。
2. 系统架构文档。
3. 常见故障 FAQ。
4. 发布规范与治理策略。

## 4.3 你要注意的文档治理
1. 文档必须版本化。
2. 过期文档及时下线。
3. 敏感文档按权限分层检索。

## 4.4 RAG 回答建议格式
1. 先结论。
2. 再引用来源（文档名称/章节）。
3. 再给执行步骤。

## 4.5 模块4 验收
1. 你能解释 RAG 如何降低幻觉。
2. 你能给出一个“来源可追溯”的回答格式。

---

## 模块5：AgentCore 快速脚手架 + Demo

## 5.1 Demo 目标
做一个“内部运维助手”最小闭环：
1. 用户问“某服务健康如何”。
2. Agent 调状态工具。
3. 返回结论 + 证据 + 建议。

## 5.2 建议目录结构
```text
ops-agent/
  app/
    main.py
    tools.py
    prompt.py
    guardrails.py
  tests/
  README.md
```

## 5.3 `tools.py` 示例（伪代码）
```python
def get_service_status(service_name: str, env: str):
    # 这里可接你们真实接口，例如 /healthz 或内部状态API
    return {
        "service_name": service_name,
        "env": env,
        "status": "healthy",
        "evidence": "last 5m error rate < 1%"
    }
```

## 5.4 Agent 响应模板（建议）
```text
结论: <healthy/degraded/down>
证据:
- <metric/log/deploy record>
下一步建议:
1) ...
2) ...
```

## 5.5 模块5 验收
1. 你能跑起一个可调用工具的 Agent demo。
2. Agent 输出包含“结论+证据+建议”。

---

## 模块6：安全、权限、观测、排错

## 6.1 安全链路（结合第10周）
1. 用户输入先过 Guardrails。
2. 工具参数校验（白名单+类型）。
3. 工具输出再过 Guardrails。
4. 最终回答返回前审查。

## 6.2 权限边界建议
1. Agent Role 最小权限。
2. 不允许 Agent 直接拿到高权限凭证。
3. 工具 API 做服务端二次鉴权。

## 6.3 观测字段建议
每次请求至少记录：
1. session_id
2. user_intent
3. selected_tools
4. tool_latency_ms
5. tool_error_type
6. final_answer_length
7. safety_intervention（是否触发）

## 6.4 常见故障排查
1. Agent 不调用工具：instruction 不清晰或 schema 不匹配。
2. 频繁调用错误工具：参数定义模糊、描述不准确。
3. 结果幻觉：知识库缺失或工具证据不足。
4. 延迟过高：多步规划过长、工具接口慢。

## 6.5 模块6 验收
1. 你能定位一次“工具调用失败”原因。
2. 你能提出 2 条降低延迟的优化动作。

---

## 模块7：最小可交付“内部运维助手”

## 7.1 交付标准
1. 支持至少 3 个只读工具。
2. 支持知识库检索回答。
3. 支持 Guardrails 输入输出拦截。
4. 有基础日志和错误追踪。
5. 有最小回归测试（关键问句集）。

## 7.2 必做演练
1. 正常问题：返回正确结论。
2. 工具异常：返回降级提示，不胡编。
3. 高风险输入：被安全策略拦截。
4. 知识缺失：明确“不确定”并请求更多信息。

## 7.3 复盘模板
1. 哪类问题 Agent 最擅长。
2. 哪类问题误判最多。
3. 工具层和知识层哪个是主要瓶颈。
4. 第12周最终项目要如何封装展示。

---

## 公司沟通可直接说的话术
1. “Agent 的核心是工具执行，不是纯文本生成。”
2. “工具 schema 的稳定性决定了 Agent 的执行可靠性。”
3. “高风险操作不走全自动，保留人工确认环节。”
4. “回答必须附证据来源，避免无依据结论。”
5. “我们会用工具成功率和平均延迟来衡量 Agent 质量。”

---

## 第11周自测题（含答案）

## 题目
1. Agent 与普通 LLM 的核心差异是什么？
2. Action Group 的作用是什么？
3. 为什么工具 schema 需要严格定义参数？
4. Knowledge Base 如何减少幻觉？
5. Guardrails 在 Agent 链路中放在哪些位置？
6. 为什么高风险工具调用要人工确认？
7. Agent 输出为什么必须附“证据”？
8. 工具调用失败时，系统应该怎么做？
9. 如何衡量 Agent 是否可上线？
10. 你会先优化 Agent 的哪一层：Prompt、Tool、Knowledge，为什么？

## 参考答案
1. Agent 会规划并调用工具完成任务，LLM 主要做文本生成。
2. 为 Agent 提供可执行的外部能力接口。
3. 防止误调用和参数歧义导致错误执行。
4. 提供可检索事实来源，减少模型臆测。
5. 输入前、工具结果后、输出前都可放审查点。
6. 降低误操作风险并满足合规审计要求。
7. 让结论可验证、可追溯、可复盘。
8. 返回可解释降级结果，不应编造成功。
9. 看任务成功率、错误率、延迟、风险拦截效果。
10. 优先 Tool 层，因执行正确性是 Agent 价值核心。

---

## 第11周完成标准（80%）
满足以下 7 条即合格：
1. 你能解释 Agent 全链路。
2. 能定义并接入至少 3 个工具。
3. 能接入知识库并输出带来源回答。
4. 能把 Guardrails 嵌入输入/输出链路。
5. 有调用日志与错误分类。
6. 能完成一次工具失败演练并恢复。
7. 产出可演示 Demo 与简要文档。

---

## 第12周衔接
下周做最终项目与答辩，你会把 1-11 周能力串成端到端平台 Demo：
1. 事件输入。
2. Agent 分析与工具执行。
3. Guardrails 安全审查。
4. 结果输出与可观测追踪。


---

## 讲义补充：知识图谱
1. 核心概念：先建立定义，再建立关系，再落实到可执行动作。
2. 工程路径：先跑通最小链路，再补安全和可观测，最后再做优化。
3. 质量标准：可复现、可观测、可回滚、可审计。

## 讲义补充：高频误区
1. 只看功能演示，不做失败路径与回滚设计。
2. 只有命令清单，没有“为什么这样做”的解释。
3. 只有部署动作，没有权限边界和成本约束。
4. 只做一次跑通，没有沉淀可复用模板。

## 讲义补充：进阶实践清单
1. 把本讲义的关键配置全部放进 Git 并走 PR 审核。
2. 为每个关键动作补一条自动化检查（lint/test/policy/check）。
3. 给每次变更写最小复盘：变更点、风险、回滚点、验证结果。
4. 在团队内做一次 10 分钟分享，验证你是否真的掌握。

## 讲义补充：学习验收方式
1. 能口头讲清关键概念和关系图。
2. 能独立完成一次从配置到验证到排错的闭环。
3. 能回答“为什么这么设计、风险在哪里、如何回滚”。
4. 能把内容迁移到你们真实业务场景。
