# Bedrock 基础与 API 实战讲义（把 LLM 接进你的平台）

## 本周目标
第9周结束后你应该能做到：
1. 解释 Bedrock 在 AWS AI 架构中的位置与价值。
2. 使用 Bedrock Runtime（Converse API）完成基础对话调用。
3. 理解模型参数（temperature、top_p、max_tokens）与输出质量关系。
4. 为模型调用建立最小可用工程规范：超时、重试、日志、成本记录。
5. 在公司里听懂并使用：`prompt template`、`inference config`、`token cost`、`latency budget`。

---

## 先记一句话
接入 LLM 不难，难的是让它“稳定、可控、可观测、可治理”。

---

## 讲义知识地图（模块）
1. 模块1：Bedrock 基础与模型调用全景。
2. 模块2：Converse API 最小可用调用。
3. 模块3：Prompt 模板化与参数调优。
4. 模块4：错误处理、重试、超时与降级。
5. 模块5：日志、指标、成本记录与可观测。
6. 模块6：基础评测（准确性、稳定性、延迟、成本）。
7. 模块7：产出“可复用的 Bedrock 调用层”模板。

---

## 模块1：Bedrock 基础

## 1.1 Bedrock 是什么
Amazon Bedrock 是 AWS 的托管式生成式 AI 平台。

你要理解它解决的事情：
1. 统一调用多个基础模型（FM）。
2. AWS 身份与权限体系统一接入（IAM/CloudTrail）。
3. 便于企业级治理（安全、审计、成本）。

## 1.2 常见组件
1. Bedrock Runtime：真正发起推理请求。
2. Converse API：统一对话接口。
3. Guardrails：输入/输出安全策略。
4. Agents：工具编排与任务执行（第11周深入）。

## 1.3 你本周先做什么
先把 “稳定调用 + 可观测 + 基础评测” 做起来，Guardrails 和 Agents 下一周继续。

## 1.4 模块1 验收
1. 你能讲清 Bedrock 与“直接调模型厂商 API”的区别。
2. 你能说出 Bedrock Runtime、Guardrails、Agents 各自职责。

---

## 模块2：Converse API 最小调用

## 2.1 先准备权限
调用 Bedrock 至少要有：
1. `bedrock:InvokeModel`
2. `bedrock:InvokeModelWithResponseStream`（流式时）

最小化建议：
1. 仅允许指定 `modelId`。
2. 不要直接 `Resource:*`。

## 2.2 Python 最小示例（boto3）
创建 `bedrock_demo.py`：
```python
import boto3
import json

REGION = "us-east-1"
MODEL_ID = "us.anthropic.claude-3-5-sonnet-20241022-v2:0"  # 示例，请按账号可用模型调整

client = boto3.client("bedrock-runtime", region_name=REGION)

resp = client.converse(
    modelId=MODEL_ID,
    messages=[
        {
            "role": "user",
            "content": [{"text": "用两句话解释什么是 Kubernetes Service"}]
        }
    ],
    inferenceConfig={
        "temperature": 0.2,
        "maxTokens": 300,
        "topP": 0.9
    }
)

# 兼容读取：不同模型返回结构略有差异，先打印后再抽取
print(json.dumps(resp, ensure_ascii=False, indent=2))
```

## 2.3 运行
```bash
python bedrock_demo.py
```

## 2.4 你要关注输出里的关键字段
1. 输出文本内容。
2. token 使用信息（如果返回）。
3. 停止原因（stop reason）。
4. 请求耗时（你自己在代码层统计）。

## 2.5 模块2 常见报错
1. `AccessDeniedException`：权限不足或模型未开通。
2. `ValidationException`：modelId/参数格式错误。
3. `ThrottlingException`：请求频率过高。

## 2.6 模块2 验收
1. 你能成功调通一次 Converse API。
2. 你能识别并解释一次调用失败原因。

---

## 模块3：Prompt 模板与参数调优

## 3.1 为什么要模板化 Prompt
1. 统一行为，减少“同问题不同输出”漂移。
2. 可版本化管理。
3. 便于线上回滚与AB测试。

## 3.2 推荐 Prompt 结构
1. System：角色和边界。
2. Context：业务上下文。
3. User Task：用户请求。
4. Output Format：输出格式约束。

## 3.3 模板示例
```text
[System]
你是内部平台助手，只回答基础设施相关问题。不要编造不存在的命令。

[Context]
环境: dev
平台: AWS + EKS

[User Task]
{user_query}

[Output Format]
1) 结论
2) 操作步骤
3) 风险提示
```

## 3.4 参数调优规律（入门够用）
1. `temperature` 越低，输出越稳定。
2. `top_p` 控制采样范围，通常 0.8~0.95。
3. `max_tokens` 控制输出上限，影响成本和延迟。

建议默认：
1. 问答型：`temperature=0.2`
2. 创意型：`temperature=0.7`

## 3.5 模块3 验收
1. 你能写一个可复用 Prompt 模板。
2. 你能解释三个参数对输出的影响。

---

## 模块4：稳定性工程（超时、重试、降级）

## 4.1 为什么必须做
LLM 调用本质是远程依赖，不做稳定性控制会放大故障。

## 4.2 你要实现的最小机制
1. 请求超时。
2. 指数退避重试（仅对可重试错误）。
3. 降级策略（例如简短固定回复或切换备用模型）。

## 4.3 Python 伪代码示例
```python
import time

MAX_RETRY = 3
for i in range(MAX_RETRY):
    try:
        # call bedrock converse
        # with timeout config at client/session layer
        break
    except Exception as e:
        is_retryable = "Throttling" in str(e) or "Timeout" in str(e)
        if not is_retryable or i == MAX_RETRY - 1:
            raise
        sleep_s = 2 ** i
        time.sleep(sleep_s)
```

## 4.4 错误分类建议
1. 参数错误：不重试，直接修请求。
2. 鉴权错误：不重试，修权限。
3. 限流/超时：有限重试。
4. 长时间不可用：触发降级。

## 4.5 模块4 验收
1. 你能在代码里实现超时 + 重试。
2. 你能说出哪些错误不该重试。

---

## 模块5：日志、指标、成本

## 5.1 每次模型调用至少记录什么
1. request_id
2. model_id
3. prompt_version
4. latency_ms
5. input_tokens/output_tokens（若可得）
6. status（success/error）
7. error_type（失败时）

## 5.2 你要做的两个指标
1. 调用成功率（success rate）。
2. P95 延迟（latency）。

进阶再加：
1. 单请求平均 token 消耗。
2. 按业务场景分组成本。

## 5.3 成本控制的实用动作
1. 控制 `max_tokens`。
2. 压缩无效上下文。
3. 按场景选更合适模型。

## 5.4 模块5 验收
1. 你有一份可查询的调用日志。
2. 你能回答“昨天哪类请求最贵/最慢”。

---

## 模块6：基础评测框架（小白可落地）

## 6.1 评测维度（先做 4 个）
1. 准确性：是否答到点子上。
2. 稳定性：同问题多次输出一致性。
3. 延迟：P50/P95。
4. 成本：平均 token 与估算费用。

## 6.2 建一个最小评测集
准备 `eval_cases.json`（20-30 条）包含：
1. 用户问题
2. 期望要点（关键词）
3. 不可接受项（例如胡编命令）

## 6.3 最小自动评测思路
1. 批量调用模型。
2. 记录输出和耗时。
3. 用规则打分（关键词覆盖、禁词检测）。
4. 输出汇总报告。

## 6.4 模块6 验收
1. 你能跑完一轮评测并产出结果表。
2. 你能基于结果给出一个优化动作（改 prompt 或改参数）。

---

## 模块7：沉淀可复用调用层

## 7.1 目标
把本周内容整理成一个“团队可复用 Bedrock SDK 封装层”。

## 7.2 目录建议
```text
llm_client/
  client.py          # bedrock 调用封装
  prompt_templates.py
  retry.py
  metrics.py
  eval.py
  README.md
```

## 7.3 你要固化的接口
1. `invoke(prompt, model, config)`
2. `invoke_with_fallback(primary_model, backup_model)`
3. `evaluate(cases)`

## 7.4 输出文档至少包含
1. 支持哪些模型与参数。
2. 默认超时/重试策略。
3. 日志字段标准。
4. 常见错误与处理建议。

## 7.5 周复盘模板
1. 哪个模型在你场景下最稳。
2. 哪类问题最贵或最慢。
3. 哪些 prompt 模板效果最好。
4. 第10周 Guardrails 需要重点拦截哪些风险。

---

## 公司沟通可直接说的话术
1. “Bedrock 调用层先做标准化封装，避免业务方重复踩稳定性坑。”
2. “我们会记录 prompt 版本和模型参数，便于回溯问题。”
3. “错误分类后只对限流/超时做重试，参数和权限错误直接失败。”
4. “评测先用小样本规则集跑起来，再逐步引入人工评审。”
5. “成本控制优先从 max_tokens 和上下文压缩开始。”

---

## 第9周自测题（含答案）

## 题目
1. Bedrock Runtime 和 Guardrails 的职责差异是什么？
2. Converse API 最小调用需要哪些要素？
3. 为什么 prompt 要模板化和版本化？
4. temperature/top_p/max_tokens 各影响什么？
5. 哪些错误应该重试，哪些不应该？
6. 为什么要记录 prompt_version 和 model_id？
7. LLM 调用最小可观测字段有哪些？
8. 如何用最小代价开始做评测？
9. 你如何判断模型“够稳定可以上线”？
10. 成本优化的前三个动作是什么？

## 参考答案
1. Runtime 负责推理调用，Guardrails 负责输入输出安全策略。
2. modelId、messages、inferenceConfig、有效权限与区域配置。
3. 便于稳定输出、回溯问题和快速回滚。
4. temperature 控随机性，top_p 控采样范围，max_tokens 控输出长度与成本。
5. 限流/超时可重试，参数/权限错误不重试。
6. 方便定位“哪个版本配置导致问题”。
7. request_id/model_id/prompt_version/latency/token/status/error。
8. 先做小规模样本+规则打分，再迭代。
9. 看成功率、稳定性、延迟和成本是否达阈值。
10. 降 max_tokens、减少无效上下文、按场景选模型。

---

## 第9周完成标准（80%）
满足以下 7 条即合格：
1. 你能稳定调用 Bedrock Converse API。
2. 有 prompt 模板和参数基线。
3. 有超时/重试/降级机制。
4. 有最小可观测日志和关键指标。
5. 能回答“哪些请求最慢最贵”。
6. 有一套可运行基础评测脚本。
7. 输出可复用调用层文档。

---

## 第10周衔接
下周你将把安全治理深入到 AI 层：
1. Bedrock Guardrails 配置与策略。
2. 输入输出内容安全、PII 处理、拒绝策略。
3. Guardrails 指标与告警联动。


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
