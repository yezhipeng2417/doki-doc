# GitOps + Helm + 回滚讲义（让发布可审计可回退）

## 本周目标
第6周结束后你应该能做到：
1. 解释 Helm、Chart、values、release、revision 的关系。
2. 用 Helm 管理 Kubernetes 应用，而不是手写散乱 YAML。
3. 理解 GitOps 的核心思想：Git 是唯一事实来源，集群状态由控制器自动对齐。
4. 能使用 Argo CD（或同类）做自动同步、差异查看、回滚。
5. 能在公司沟通中使用：`desired state`、`drift`、`sync`、`rollback`、`revision`。

---

## 先记住两句话
1. Helm 解决“如何标准化打包和参数化部署”。
2. GitOps 解决“谁说了算，以及如何持续对齐和审计”。

你把这两件事合起来，发布流程就会明显稳定。

---

## 讲义知识地图（模块）
1. 模块1：Helm 基础与 Chart 结构。
2. 模块2：把你现有 Deployment/Service 改造成 Helm Chart。
3. 模块3：多环境 values（dev/stage/prod）与模板实践。
4. 模块4：GitOps 概念 + Argo CD 快速入门。
5. 模块5：接入 Git 仓库自动同步部署。
6. 模块6：发布策略与回滚（Helm rollback + Git 回滚）。
7. 模块7：故障演练 + 可复用发布规范模板。

---

## 模块1：Helm 基础

## 1.1 Helm 是什么
Helm 是 Kubernetes 的“包管理器”。

你要理解这些词：
1. Chart：应用模板包（K8s 资源模板集合）。
2. values.yaml：参数输入。
3. release：某个 chart 的一次安装实例。
4. revision：release 的版本号（每次升级 +1）。

## 1.2 为什么用 Helm
1. 减少重复 YAML。
2. 同一套模板支持多环境。
3. 回滚更标准化。

## 1.3 安装与验证
```bash
helm version
kubectl version --client
```

## 1.4 快速生成 chart 骨架
```bash
mkdir -p ~/projects/platform_learning/labs/06-gitops-helm
cd ~/projects/platform_learning/labs/06-gitops-helm
helm create week6-api
```

生成目录关键文件：
1. `Chart.yaml`：chart 元数据。
2. `values.yaml`：默认参数。
3. `templates/`：K8s 模板。

## 1.5 模块1 验收
1. 你能说清 chart/release/revision 的区别。
2. 你能创建并看懂 chart 目录结构。

---

## 模块2：把现有应用改造成 Helm Chart

## 2.1 精简默认模板（建议）
进入 chart 目录，保留你需要的模板：
1. `templates/deployment.yaml`
2. `templates/service.yaml`
3. `templates/ingress.yaml`（可选）
4. `templates/serviceaccount.yaml`

## 2.2 values.yaml（示例）
```yaml
replicaCount: 2

image:
  repository: "<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/week3-api"
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 5000

ingress:
  enabled: true
  className: alb
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
  hosts:
    - host: ""
      paths:
        - path: /
          pathType: Prefix

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "300m"
    memory: "256Mi"

probes:
  readiness:
    path: /health
    initialDelaySeconds: 5
    periodSeconds: 10
  liveness:
    path: /health
    initialDelaySeconds: 10
    periodSeconds: 15

serviceAccount:
  create: true
  name: "week6-api-sa"
```

## 2.3 模板关键写法（deployment 片段）
```yaml
containers:
- name: {{ .Chart.Name }}
  image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
  imagePullPolicy: {{ .Values.image.pullPolicy }}
  ports:
  - containerPort: {{ .Values.service.targetPort }}
  readinessProbe:
    httpGet:
      path: {{ .Values.probes.readiness.path }}
      port: {{ .Values.service.targetPort }}
    initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
    periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
  livenessProbe:
    httpGet:
      path: {{ .Values.probes.liveness.path }}
      port: {{ .Values.service.targetPort }}
    initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
    periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
```

## 2.4 本地渲染检查
```bash
helm template week6-api ./week6-api > rendered.yaml
```

先看渲染结果再部署，能提前发现模板错误。

## 2.5 模块2 验收
1. 你能把 image/tag/replica/resources 放到 values 管理。
2. 你能用 `helm template` 检查渲染结果。

---

## 模块3：多环境 values

## 3.1 目录建议
在 chart 目录下创建：
1. `values-dev.yaml`
2. `values-stage.yaml`
3. `values-prod.yaml`

## 3.2 示例：values-dev.yaml
```yaml
replicaCount: 1
image:
  tag: dev-latest
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"
```

## 3.3 示例：values-prod.yaml
```yaml
replicaCount: 3
image:
  tag: prod-latest
resources:
  requests:
    cpu: "300m"
    memory: "512Mi"
  limits:
    cpu: "1000m"
    memory: "1Gi"
```

## 3.4 部署命令（dev）
```bash
helm upgrade --install week6-api ./week6-api \
  -n dev \
  --create-namespace \
  -f ./week6-api/values-dev.yaml
```

## 3.5 查看 release 状态
```bash
helm list -n dev
helm status week6-api -n dev
helm history week6-api -n dev
```

## 3.6 模块3 验收
1. 同一 chart 能在不同环境使用不同参数。
2. 你能查 release history。

---

## 模块4：GitOps 与 Argo CD 入门

## 4.1 GitOps 的核心
1. Git 存“期望状态”（desired state）。
2. 控制器持续比较“实际状态”与“期望状态”。
3. 有偏差（drift）就自动或手动同步（sync）。

## 4.2 你要得到的收益
1. 所有变更可审计（PR 记录）。
2. 集群漂移可检测并纠正。
3. 回滚可以通过 Git revert 或 revision 回退。

## 4.3 Argo CD 你先理解这几个对象
1. Application：一个应用声明（指向 Git 路径和目标集群/namespace）。
2. Sync Policy：自动同步或手动同步。
3. Health/Sync Status：健康状态与同步状态。

## 4.4 模块4 验收
1. 你能讲清 GitOps 和“手工 kubectl apply”的差异。
2. 你能解释 drift 是什么。

---

## 模块5：接入 Git 仓库自动同步

## 5.1 Git 仓库结构建议
```text
platform-config/
  apps/
    week6-api/
      Chart.yaml
      values-dev.yaml
      values-stage.yaml
      values-prod.yaml
      templates/
  envs/
    dev/
      week6-api-app.yaml
    stage/
      week6-api-app.yaml
    prod/
      week6-api-app.yaml
```

## 5.2 Argo CD Application 示例（dev）
`envs/dev/week6-api-app.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: week6-api-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-org>/platform-config.git
    targetRevision: main
    path: apps/week6-api
    helm:
      valueFiles:
      - values-dev.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

## 5.3 你要理解的两个开关
1. `prune: true`：Git 删除了资源，集群也删除。
2. `selfHeal: true`：集群被手工改动后，自动拉回 Git 状态。

## 5.4 模块5 验收
1. 你能把 chart 放到 Git 并被 Argo CD 同步。
2. 你能看懂 Argo 的 Sync/Health 状态。

---

## 模块6：回滚策略（必须会）

## 6.1 Helm 级回滚
查看历史：
```bash
helm history week6-api -n dev
```
回滚到指定 revision：
```bash
helm rollback week6-api <REVISION> -n dev
```

## 6.2 GitOps 级回滚
1. 把有问题的提交 `git revert`。
2. 合并后 Argo CD 自动同步到旧配置。

这是团队最常用的“可审计回滚”方式。

## 6.3 发布保护建议
1. dev 自动同步。
2. stage 半自动（需要人工确认）。
3. prod 手动同步 + 审批。

## 6.4 发布失败排错顺序
1. 看 Argo Application 事件。
2. 看 Helm 渲染错误（values 或模板）。
3. 看 Pod 事件、日志、probe。
4. 看 Ingress/Service/endpoints。

## 6.5 模块6 验收
1. 你能执行一次 Helm rollback。
2. 你能通过 Git revert 触发 GitOps 回滚。

---

## 模块7：故障演练与模板沉淀

## 7.1 必做故障演练
1. 故意把 image tag 改成不存在，观察发布失败并回滚。
2. 故意把 readiness path 改错，观察健康失败并修复。
3. 故意在集群手工改副本数，观察 Argo self-heal 拉回。

## 7.2 你要沉淀一个“团队可复用模板”
输出文档建议包含：
1. chart 目录规范。
2. values 命名规范。
3. 环境发布策略。
4. 回滚 SOP（标准操作步骤）。

## 7.3 周复盘模板
1. 我学会了哪些 Helm 命令与 Argo 概念。
2. 我踩过哪些模板/参数坑。
3. 我是如何回滚并恢复服务的。
4. 我如何把这套流程接入第7周可观测性。

---

## 公司沟通可直接用的话术
1. “部署状态以 Git 为准，避免人工改集群导致漂移。”
2. “先看 Argo sync status，再看 Helm 渲染和 Pod 事件。”
3. “回滚优先走 Git revert，保证变更链路可审计。”
4. “同一 chart 多环境通过 values 区分，不复制模板文件。”
5. “prod 只允许受控同步，避免自动同步造成误发。”

---

## 第6周自测题（含答案）

## 题目
1. Helm chart、release、revision 分别是什么？
2. 为什么要用 values 文件做环境差异？
3. GitOps 的“desired state”是什么意思？
4. drift 是什么，如何处理？
5. `prune` 和 `selfHeal` 分别做什么？
6. 什么时候用 Helm rollback，什么时候用 Git revert？
7. 为什么 GitOps 比手工 `kubectl apply` 更可审计？
8. 发布失败时为什么先看渲染再看运行时？
9. prod 环境为什么不建议无审批自动同步？
10. 你的一条标准回滚 SOP 应包含哪些步骤？

## 参考答案
1. chart 是模板包，release 是安装实例，revision 是 release 历史版本号。
2. 统一模板、减少重复、清晰管理环境差异。
3. Git 中声明的期望资源状态。
4. 实际状态偏离 Git 声明，使用 sync/self-heal 拉回。
5. prune 删除多余资源，selfHeal 自动纠正漂移。
6. Helm rollback 用于快速回退 release；Git revert 用于可审计配置回退。
7. 每次变更有 PR/commit 记录，回滚路径明确。
8. 渲染错误会导致资源根本创建失败，先排除模板问题更高效。
9. 降低误操作风险，确保关键变更经过人工确认。
10. 定位失败、选择回滚方式、执行回滚、验证健康、记录复盘。

---

## 第6周完成标准（80%）
满足以下 7 条即合格：
1. 你能独立维护一个 Helm chart。
2. 你能用 values 管理 dev/stage/prod 差异。
3. 你能用 Helm 完成升级和回滚。
4. 你能解释 GitOps 与 drift/self-heal。
5. 你能部署并查看 Argo CD Application 状态。
6. 你能做一次 Git revert 驱动的回滚。
7. 你能产出一份团队可复用发布规范。

---

## 第7周衔接
下周进入可观测性，你会把发布链路和监控链路打通：
1. 指标、日志、追踪统一观测。
2. 发布事件与故障告警关联。
3. 建立“发现问题 -> 定位 -> 回滚”的闭环。


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
