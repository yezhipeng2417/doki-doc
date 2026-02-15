# 可观测性讲义（Metrics + Logs + Traces + Alerting）

## 本周目标
第7周结束后你应该能做到：
1. 解释可观测性的三个支柱：指标（Metrics）、日志（Logs）、链路（Traces）。
2. 在 EKS 场景下使用 CloudWatch/Container Insights 看集群与应用状态。
3. 设计最小可用告警体系（错误率、延迟、可用性、资源异常）。
4. 建立“发布事件”和“故障指标”关联，能快速判断是否是新版本引发问题。
5. 能在公司里听懂并使用：`SLI`、`SLO`、`error budget`、`MTTR`、`golden signals`。

---

## 先讲一句话
可观测性不是“看图”，而是“发现异常 -> 快速定位 -> 快速恢复”的能力系统。

---

## 讲义知识地图（模块）
1. 模块1：可观测性基础与黄金指标（Golden Signals）。
2. 模块2：CloudWatch Metrics 与 Dashboard 实战。
3. 模块3：CloudWatch Logs 与结构化日志规范。
4. 模块4：Container Insights（EKS 维度观测）。
5. 模块5：告警策略设计（阈值、去噪、升级）。
6. 模块6：发布关联与故障定位流程（Runbook）。
7. 模块7：一套可复用观测模板 + 演练复盘。

---

## 模块1：可观测性基础

## 1.1 三个支柱
1. Metrics：时间序列数值，适合看趋势和阈值。
2. Logs：事件文本，适合定位具体错误。
3. Traces：请求链路，适合看跨服务调用瓶颈。

## 1.2 Golden Signals（高频框架）
1. Latency（延迟）
2. Traffic（流量）
3. Errors（错误）
4. Saturation（饱和度）

这四个信号足够支撑 80% 一线排障。

## 1.3 你要补的运维指标概念
1. SLI（Service Level Indicator）：服务质量指标（如 99.9% 请求 < 500ms）。
2. SLO（Service Level Objective）：目标值（如可用性 99.9%）。
3. Error Budget：在周期内可接受的失败额度。
4. MTTR：平均恢复时间。

## 1.4 模块1 验收
1. 你能解释 metrics/logs/traces 的分工。
2. 你能说出 Golden Signals。
3. 你能给出一个简单 SLO 例子。

---

## 模块2：CloudWatch Metrics + Dashboard

## 2.1 CloudWatch Metrics 来源
EKS 常见指标来源：
1. EC2 节点（CPU、网络、磁盘）
2. ALB（请求量、5xx、TargetResponseTime）
3. 应用自定义指标（通过 SDK 或 sidecar）
4. Container Insights（Pod/Node 维度）

## 2.2 最小 Dashboard 该放什么
建议先放 8 个图：
1. ALB 请求量
2. ALB 4xx/5xx
3. ALB P95 延迟
4. Pod 重启次数
5. Pod CPU 使用率
6. Pod Memory 使用率
7. 节点 CPU/Memory
8. 应用错误日志速率（可用 metric filter）

## 2.3 命名规范建议
Dashboard 名称：
`team-service-env` 例如 `infra-week3api-dev`

Metric 维度建议统一：
1. `service`
2. `env`
3. `version`

## 2.4 模块2 实操任务
1. 新建一个 CloudWatch Dashboard。
2. 加入 ALB 关键图表。
3. 加入 EKS 节点关键图表。

## 2.5 模块2 验收
1. 你能打开 dashboard 快速判断“服务是否异常”。
2. 你能回答“现在是流量问题、错误问题还是资源问题”。

---

## 模块3：CloudWatch Logs + 结构化日志

## 3.1 为什么日志要结构化
纯文本日志不利于查询和聚合。

推荐 JSON 结构日志字段：
1. `timestamp`
2. `level`
3. `service`
4. `env`
5. `trace_id`
6. `message`
7. `error_code`（可选）
8. `request_id`（可选）

## 3.2 Python 示例（结构化日志）
```python
import json
import logging
import time

class JsonFormatter(logging.Formatter):
    def format(self, record):
        payload = {
            "timestamp": int(time.time() * 1000),
            "level": record.levelname,
            "service": "week3-api",
            "env": "dev",
            "message": record.getMessage(),
        }
        return json.dumps(payload)

logger = logging.getLogger()
handler = logging.StreamHandler()
handler.setFormatter(JsonFormatter())
logger.addHandler(handler)
logger.setLevel(logging.INFO)
```

## 3.3 Logs Insights 常用查询
1. 看最近错误：
```sql
fields @timestamp, @message
| filter level = "ERROR"
| sort @timestamp desc
| limit 50
```
2. 按错误码聚合：
```sql
fields error_code
| stats count() by error_code
| sort count() desc
```
3. 看某 request_id 全链路日志：
```sql
fields @timestamp, @message, request_id
| filter request_id = "abc-123"
| sort @timestamp asc
```

## 3.4 模块3 验收
1. 你能写一条日志查询语句定位错误。
2. 你能解释为什么要加 `trace_id/request_id`。

---

## 模块4：Container Insights（EKS 维度）

## 4.1 Container Insights 看什么
1. Node 层：CPU/Memory/Network。
2. Pod 层：重启、资源占用、状态变化。
3. Cluster 层：整体容量与趋势。

## 4.2 你要重点盯的信号
1. Pod 重启突然上升。
2. 某 Deployment Ready 比例下降。
3. 节点 Memory 接近上限。
4. 某 namespace 的资源异常增长。

## 4.3 常见现象与解释
1. CPU 高但延迟不高：可能流量上升但系统仍有余量。
2. CPU 不高但延迟高：可能 I/O、下游依赖或锁竞争。
3. 重启频繁：可能 OOM、探针失败、配置异常。

## 4.4 模块4 验收
1. 你能从 Container Insights 找到“哪一层在异常”。
2. 你能把异常和部署变更时间对齐。

---

## 模块5：告警策略设计

## 5.1 告警设计目标
不是“越多越好”，而是“关键问题必须及时发现，噪声尽量少”。

## 5.2 最小告警清单（推荐）
1. 可用性：5xx 比例超过阈值。
2. 延迟：P95 延迟超过阈值。
3. 资源：Pod OOM 或节点内存高水位。
4. 部署：rollout 失败。
5. 心跳：关键服务健康检查失败。

## 5.3 阈值设置思路
1. 先基于近 2-4 周基线。
2. 先宽后紧，不要一开始就过敏。
3. 使用持续时间窗口（例如 5 分钟）降低抖动。

## 5.4 告警分级建议
1. P1：用户明显受影响（立即响应）。
2. P2：部分影响或风险升高（尽快处理）。
3. P3：优化项或非实时问题（排期处理）。

## 5.5 模块5 实操建议
1. 创建至少 3 条 CloudWatch Alarm（错误率、延迟、资源）。
2. 绑定通知通道（SNS -> 邮件/Slack）。

## 5.6 模块5 验收
1. 你有“可执行”的告警规则而非口头约定。
2. 你能解释每条告警的业务意义。

---

## 模块6：发布关联与 Runbook

## 6.1 为什么要关联“发布事件”
当错误率突然上升时，第一判断是：
1. 是新版本引发？
2. 还是外部依赖/流量波动？

所以要把“部署时间点”打到日志/指标里。

## 6.2 实践方式
1. 发布时记录版本号（commit sha）到应用日志。
2. 指标维度包含 version。
3. 仪表盘标注 deploy 时间（annotation）。

## 6.3 标准排障 Runbook（必须背）
1. 告警触发，确认影响范围（单服务/全链路）。
2. 看 dashboard：错误、延迟、流量、饱和度。
3. 对齐最近 deploy 时间。
4. 钻日志定位错误类型。
5. 判断是否回滚。
6. 回滚后验证指标恢复。
7. 记录事故复盘。

## 6.4 一条可直接用的 Runbook 模板
1. 现象：
2. 影响范围：
3. 检测时间：
4. 最近变更：
5. 排查路径：
6. 根因：
7. 处置动作：
8. 恢复时间：
9. 后续改进：

## 6.5 模块6 验收
1. 你能按 Runbook 在 15 分钟内定位大致问题方向。
2. 你能说明“何时应该立即回滚”。

---

## 模块7：演练与模板沉淀

## 7.1 必做演练
1. 故意制造 `/health` 失败，观察告警触发并恢复。
2. 故意让错误率升高（模拟异常请求），观察 dashboard 变化。
3. 做一次发布，验证版本维度与部署标注是否可追踪。

## 7.2 你要沉淀的 3 份模板
1. Dashboard 模板（服务统一版）。
2. Alarm 模板（分级阈值版）。
3. Incident Runbook 模板。

## 7.3 周复盘模板
1. 哪些指标最能反映真实故障。
2. 哪些告警噪声最大，如何优化。
3. 我从告警到定位用了多久（MTTR）。
4. 下周安全与成本治理如何接上观测数据。

---

## 公司沟通可直接说的话术
1. “先看 Golden Signals，再决定是扩容、回滚还是查下游依赖。”
2. “这次错误峰值和 xx:xx 的发布时间重合，优先排查新版本。”
3. “日志需要结构化并带 request_id/trace_id，不然定位效率太低。”
4. “告警阈值会基于历史基线逐步收敛，先保证有效，再降噪。”
5. “我们会把 Runbook 固化，确保一线同学按同样步骤处理故障。”

---

## 第7周自测题（含答案）

## 题目
1. Metrics、Logs、Traces 的分工是什么？
2. Golden Signals 包含哪四项？
3. SLI、SLO、Error Budget 的关系是什么？
4. 为什么日志必须结构化？
5. 为什么要记录 request_id 或 trace_id？
6. 告警阈值为什么不能“一刀切”？
7. 发布关联为什么能提升排障速度？
8. MTTR 是什么，如何降低？
9. 一条好的 Runbook 应包含哪些关键项？
10. 你会如何判断“先回滚还是继续排查”？

## 参考答案
1. Metrics 看趋势，Logs 看细节，Traces 看跨服务链路。
2. 延迟、流量、错误、饱和度。
3. SLI 是测量，SLO 是目标，Error Budget 是可容忍失败额度。
4. 便于查询、聚合、统计和自动化分析。
5. 用于跨日志关联同一请求，快速还原问题路径。
6. 不同服务基线不同，阈值需结合历史数据与业务影响。
7. 能快速判断是否由新版本引发，减少盲目排查。
8. 平均恢复时间；通过标准化告警、Runbook、自动化回滚降低。
9. 现象、影响、时间线、变更、排查、根因、处置、改进。
10. 看影响范围和恢复收益；用户受损明显且回滚成本低时优先回滚。

---

## 第7周完成标准（80%）
满足以下 7 条即合格：
1. 有一套可用 dashboard。
2. 有至少 3 条关键告警并能通知到人。
3. 日志结构化并可查询聚合。
4. 能把发布事件与故障曲线对齐。
5. 有可执行的 Runbook。
6. 能完成一次从告警到恢复的演练。
7. 能输出 MTTR 和后续优化方向。

---

## 第8周衔接
下周进入安全与成本治理，你会把观测数据用于治理决策：
1. 安全事件监控与权限审计。
2. 资源利用率与成本优化。
3. 从“看见问题”升级到“持续优化”。


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
