# EKS Networking + Security 讲义（把集群跑稳并跑安全）

## 本周目标
第4周结束后你应该能做到：
1. 解释 EKS 网络路径：Pod IP、Service、CoreDNS、Ingress(ALB)、VPC 路由。
2. 说清楚公有/私有子网在 EKS 架构里的职责分工。
3. 理解并应用安全基线：RBAC、IRSA、最小权限、Secret 管理、镜像来源控制。
4. 能定位常见网络故障：DNS 失败、503、超时、跨命名空间访问异常。
5. 能在公司交流里听懂并使用：`CNI`、`Security Group for Pods`、`NetworkPolicy`、`RBAC`、`KMS encryption`。

---

## 先记住这张逻辑图（文字版）
1. 外部流量：`Internet -> ALB -> Ingress -> Service -> Pod`
2. 集群内流量：`Pod A -> CoreDNS -> Service 名称解析 -> Pod B`
3. 出网流量：`Pod -> Node ENI -> Route Table -> NAT Gateway/IGW`

EKS 网络问题 80% 都在这三条链路里。

---

## 讲义知识地图（模块）
1. 模块1：VPC 与 EKS 网络模型（必须打牢）。
2. 模块2：Service/DNS/跨命名空间通信与排查。
3. 模块3：ALB Ingress 网络细节（子网、SG、健康检查）。
4. 模块4：K8s 安全基线（RBAC、ServiceAccount、Admission 思路）。
5. 模块5：AWS 侧安全（IRSA 深化、IAM 最小权限、Secret/KMS）。
6. 模块6：NetworkPolicy 与东西向流量隔离。
7. 模块7：一套“可讲可演示”的网络与安全检查清单。

---

## 模块1：VPC 与 EKS 网络模型

## 1.1 你要先理解 EKS 的“IP 是真的 VPC IP”
在 AWS VPC CNI 下（EKS 默认）：
1. Pod 通常拿到的是 VPC 网段真实 IP。
2. Node 上 ENI 提供可分配 IP。
3. 这让 Pod 能直接和 VPC 资源通信（受 SG/NACL/路由影响）。

## 1.2 公有子网和私有子网在 EKS 的推荐分工
1. Node（工作节点）优先放私有子网。
2. ALB 放公有子网（互联网入口）。
3. Node 出网走 NAT（如果需要拉镜像/访问外部 API）。

## 1.3 必会网络组件职责
1. `Route Table`：决定流量下一跳。
2. `Internet Gateway`：公网进出网关。
3. `NAT Gateway`：私网实例主动出网。
4. `Security Group`：有状态实例防火墙。
5. `NACL`：无状态子网防火墙。

## 1.4 你要会画的最小 EKS 架构
1. 2 个公有子网（ALB）。
2. 2 个私有子网（NodeGroup）。
3. 每个 AZ 至少一个子网。
4. 私有子网路由到 NAT。
5. 公有子网路由到 IGW。

## 1.5 模块1 验收
1. 你能解释“为什么 Node 不建议直接暴露公网”。
2. 你能口述 Pod 出网路径。
3. 你能区分 SG 与 NACL 的差异。

---

## 模块2：Service + DNS + 集群内通信排查

## 2.1 Service 的本质
Service 提供稳定虚拟入口，背后通过 Endpoints 指向一组 Pod。

你要检查 3 件事：
1. Service selector 是否匹配 Pod label。
2. Endpoints 是否为空。
3. Pod readiness 是否通过。

## 2.2 CoreDNS 的作用
CoreDNS 负责把服务名解析成集群内地址。

典型访问形式：
1. 同 namespace：`http://week3-api-svc`
2. 跨 namespace：`http://week3-api-svc.week3.svc.cluster.local`

## 2.3 常用排错命令
```bash
kubectl get svc -A
kubectl get endpoints -A
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system deploy/coredns
kubectl run -it --rm netshoot --image=nicolaka/netshoot -- sh
```

进入调试 Pod 后：
```bash
nslookup week3-api-svc.week3.svc.cluster.local
curl -v http://week3-api-svc.week3.svc.cluster.local/health
```

## 2.4 常见故障判断
1. DNS 解析失败：CoreDNS 或网络策略限制。
2. DNS 成功但请求超时：Service 后端不可达、NetworkPolicy 拦截、Pod 没监听端口。
3. 返回 503：Service 无健康后端。

## 2.5 模块2 验收
1. 你能独立查出“为什么 Service 没后端”。
2. 你能解释 FQDN 访问格式。

---

## 模块3：ALB Ingress 网络细节

## 3.1 ALB 为什么创建不出来
高频原因：
1. 子网没打对标签。
2. Controller 权限不足。
3. Ingress annotation 不正确。
4. SG/NACL 阻断健康检查。

## 3.2 子网标签（你要会查）
EKS + ALB 常见标签：
1. 公有子网：`kubernetes.io/role/elb=1`
2. 私有子网：`kubernetes.io/role/internal-elb=1`
3. 集群标签（按版本/方案可能不同）需与当前实践对齐。

查询示例：
```bash
aws ec2 describe-subnets --filters Name=vpc-id,Values=<VPC_ID>
```

## 3.3 Ingress 常用 annotation（示例）
```yaml
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
```

## 3.4 你要理解 target-type
1. `ip`：ALB 直接把 Pod IP 作为 target（EKS 常用）。
2. `instance`：ALB 指向 NodePort。

## 3.5 ALB 排错顺序
1. `kubectl describe ingress -n <ns> <name>` 看事件。
2. 看 controller 日志：
```bash
kubectl logs -n kube-system deploy/aws-load-balancer-controller
```
3. AWS 控制台看 target group 健康状态。
4. 检查 SG/NACL/子网路由。

## 3.6 模块3 验收
1. 你能说清 ingress 到 ALB 的映射逻辑。
2. 你会查 target group 健康失败原因。

---

## 模块4：K8s 安全基线（RBAC + 身份边界）

## 4.1 RBAC 的三个对象
1. Role/ClusterRole：定义能做什么。
2. RoleBinding/ClusterRoleBinding：把权限绑定给主体。
3. Subject：User/Group/ServiceAccount。

## 4.2 最小权限思想在 K8s 的落地
1. 默认不授予过大权限（例如 cluster-admin）。
2. 按 namespace 切权限。
3. 只给服务所需 API 访问能力。

## 4.3 ServiceAccount 安全原则
1. 每个应用用独立 ServiceAccount。
2. 不复用默认 ServiceAccount。
3. 与 IRSA 一起控制 AWS API 权限。

## 4.4 RBAC 示例（只读 Pod）
`rbac-read-pods.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: week3
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-pod-reader
  namespace: week3
subjects:
- kind: ServiceAccount
  name: week3-api-sa
  namespace: week3
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

## 4.5 权限验证命令
```bash
kubectl auth can-i list pods -n week3 --as=system:serviceaccount:week3:week3-api-sa
kubectl auth can-i delete pods -n week3 --as=system:serviceaccount:week3:week3-api-sa
```

## 4.6 模块4 验收
1. 你能写出最小 Role + RoleBinding。
2. 你能用 `kubectl auth can-i` 验证权限。

---

## 模块5：AWS 侧安全（IRSA 深化 + Secret + KMS）

## 5.1 IRSA 强化理解
IRSA = `K8s ServiceAccount` 对应 `IAM Role`，Pod 获取临时凭证调用 AWS API。

安全收益：
1. 无长期密钥。
2. 精准授权到服务。
3. 审计可追踪到角色。

## 5.2 IAM 最小权限实战建议
1. 先从“只读策略”起步。
2. 明确资源 ARN，避免 `*`。
3. 按功能拆策略（S3 读、SQS 写分开）。

## 5.3 Secret 管理原则
1. K8s Secret 不等于高强度加密系统。
2. 敏感信息不要进镜像、不要写 Git。
3. 生产建议用外部 Secret 管理（如 AWS Secrets Manager）并控制访问策略。

## 5.4 EKS Secret 加密
EKS 可配合 KMS 对 Secret at-rest 加密（集群层配置）。

你要理解：
1. “传输加密（TLS）”和“存储加密（KMS）”是两件事。
2. Secret 安全是“加密 + 权限 + 审计”的组合，不是单点功能。

## 5.5 模块5 验收
1. 你能解释 IRSA 与 AK/SK 的安全差异。
2. 你能说出 Secret 管理的 3 条底线。
3. 你知道 KMS 加密解决的是哪一层风险。

---

## 模块6：NetworkPolicy（东西向流量隔离）

## 6.1 为什么要 NetworkPolicy
默认很多集群内部通信是“较开放”的。NetworkPolicy 可以限制谁能访问谁。

## 6.2 先理解模型
1. 先选中一组 Pod（podSelector）。
2. 再定义允许的 ingress/egress 规则。
3. 未被允许的流量被拒绝（取决于网络插件支持）。

## 6.3 示例：只允许同 namespace 的 app=frontend 访问 app=backend
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-only-from-frontend
  namespace: week3
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 5000
```

## 6.4 你要注意的现实点
1. NetworkPolicy 是否生效依赖 CNI/策略引擎支持。
2. 放开规则要包含 DNS 等基础通信需求。
3. 一上来不要全拒绝，先按业务流量逐步收敛。

## 6.5 模块6 验收
1. 你能读懂并修改一条 NetworkPolicy。
2. 你能解释“为什么加了策略后服务忽然不通”。

---

## 模块7：总演练（网络 + 安全检查清单）

## 7.1 你要完成一份可执行检查单

### A. 网络检查
1. Node 在私有子网。
2. ALB 在公有子网。
3. 路由和 NAT 正常。
4. Service endpoints 非空。
5. CoreDNS 正常解析。

### B. 安全检查
1. 每个工作负载独立 ServiceAccount。
2. IRSA 已配置，容器无 AK/SK。
3. RBAC 仅授予必要权限。
4. Secret 不入 Git，不入镜像。
5. 至少一条 NetworkPolicy 生效。

### C. 可观测性基础
1. Controller 日志可查。
2. 关键错误有告警入口（下周会系统化）。

## 7.2 必做演练
1. 故意把 Service selector 改错，复现 503，再修复。
2. 故意把 readiness path 改错，观察 target health 变化，再修复。
3. 给某 Pod 移除必要权限，验证 IRSA 拒绝，再恢复。

## 7.3 周复盘模板
1. 我能解释的 20 个 networking/security 术语。
2. 我能独立执行的 15 个命令。
3. 我本周遇到的 3 个真实问题和定位过程。
4. 我准备在第5周 CI/CD 落地时如何避免发布风险。

---

## 公司里可直接说的话术
1. “先确认 service endpoints，再看 ingress 和 ALB target health，避免盲目改代码。”
2. “这个服务建议独立 ServiceAccount + IRSA，不要继承默认账号权限。”
3. “安全组是有状态，NACL 是无状态，排查时要两边都看。”
4. “Secret 不能只靠 base64，当成加密方案，需要配合 KMS 与访问控制。”
5. “NetworkPolicy 先按最小开放逐步收敛，不要一次性全封导致业务中断。”

---

## 第4周自测题（含答案）

## 题目
1. 为什么 EKS 节点通常放私有子网？
2. Pod 到公网常见路径是什么？
3. Service 有 selector 但 endpoints 为空，最可能的问题是什么？
4. DNS 解析失败时你先查哪个组件？
5. ALB 健康检查失败通常看哪三块？
6. RBAC 的 Role 和 RoleBinding 分别负责什么？
7. 为什么默认 ServiceAccount 不建议直接用于生产服务？
8. IRSA 相比硬编码 AK/SK 的核心优势是什么？
9. NetworkPolicy 生效依赖什么前提？
10. Secret 安全为什么是“组合拳”而不是单一功能？

## 参考答案
1. 减少暴露面，降低攻击面，配合 NAT 出网更安全。
2. Pod -> Node ENI -> Route Table -> NAT/IGW。
3. label/selector 不匹配或 Pod 未 Ready。
4. CoreDNS（及其网络可达性）。
5. Ingress/controller 事件、target group 健康、SG/NACL/路由。
6. Role 定义权限，RoleBinding 绑定权限给主体。
7. 权限边界不清，容易越权和审计困难。
8. 无长期密钥、最小权限、可审计。
9. 网络插件/策略引擎支持该能力。
10. 因为需要同时解决存储加密、访问控制、密钥轮换和审计。

---

## 第4周完成标准（80%）
满足以下 7 条即合格：
1. 能画出 EKS 网络主路径（内外流量）。
2. 能定位至少一次 Service/Ingress 不通问题。
3. 能用命令验证 DNS 与 endpoints。
4. 能写出基础 RBAC 并验证权限。
5. 能说明并检查 IRSA 是否正确使用。
6. 能写出一条可用 NetworkPolicy。
7. 能输出一份网络与安全检查清单给团队。

---

## 给第5周的衔接
你下周进入 CI/CD，会把“网络和安全底座”接到“自动发布流程”：
1. GitHub Actions 构建镜像并 push 到 ECR。
2. 自动更新集群部署。
3. 失败自动回滚。
4. 在发布流程中固化安全检查（镜像来源、权限、环境隔离）。


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
