# EKS 入门实战讲义（从本地 K8s 到 AWS 托管集群）

## 本周目标（你会达到什么水平）
第3周结束后你应该能做到：
1. 说清楚 EKS、NodeGroup、Fargate、ECR、ALB Ingress Controller、IRSA 的作用。
2. 在 AWS 上创建一个可用 EKS 集群并连接 `kubectl`。
3. 把本地镜像推到 ECR 并部署到 EKS。
4. 用 ALB Ingress 对外暴露服务。
5. 能解释为什么在 EKS 里要用 IRSA，而不是把 AK/SK 放在 Pod 里。

---

## 先说大图（你要背下来）
从“代码”到“公网访问”的路径是：
1. 代码打包成 Docker 镜像。
2. 镜像推到 ECR。
3. EKS 集群拉取镜像运行 Pod。
4. Service 负责集群内稳定访问。
5. ALB Ingress Controller 创建 ALB，把公网流量引到 Service。

一句话：`ECR -> EKS(Pod) -> Service -> Ingress(ALB) -> Internet`。

---

## 讲义知识地图（模块）
1. 模块1：EKS 核心概念与环境准备。
2. 模块2：创建 EKS 集群与 NodeGroup。
3. 模块3：ECR 实战（构建、标记、推送镜像）。
4. 模块4：把应用部署到 EKS（Deployment/Service）。
5. 模块5：安装并使用 ALB Ingress Controller。
6. 模块6：IRSA（IAM Role for Service Account）实战。
7. 模块7：总演练 + 排错复盘。

---

## 模块1：EKS 概念和准备

## 1.1 EKS 是什么
EKS（Elastic Kubernetes Service）是 AWS 托管的 Kubernetes 控制面。

你要理解：
1. 控制面（API Server 等）由 AWS 托管。
2. 你主要管理工作节点（Node）和工作负载（Pod/Deployment）。
3. 这样能少做很多“自建集群运维”工作。

## 1.2 NodeGroup 和 Fargate 的区别
1. Managed NodeGroup：
   - 本质是 EC2 节点池。
   - 你控制实例类型、容量、成本。
   - 适合大多数通用场景。
2. Fargate：
   - 无服务器模式，Pod 不需要你管 Node。
   - 更省节点运维，但成本模型和限制不同。

初学建议：先用 NodeGroup 跑通，再理解 Fargate。

## 1.3 本周工具检查
```bash
aws --version
kubectl version --client
eksctl version
docker --version
```

如果 `eksctl` 没装，你可以用 Homebrew：
```bash
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

## 1.4 AWS CLI 身份确认
```bash
aws sts get-caller-identity
aws configure list
```

你必须确认当前账号和区域，避免资源创建错区。

## 1.5 模块1 验收
1. 你能讲清 EKS 与自建 K8s 的差异。
2. 你能区分 NodeGroup 与 Fargate。
3. 你的 `aws/kubectl/eksctl/docker` 全可用。

---

## 模块2：创建 EKS 集群（NodeGroup 方式）

## 2.1 创建集群（最小学习配置）
> 注意：以下命令会创建云资源并产生费用，学习后及时删除。

```bash
export AWS_REGION=us-east-1
export CLUSTER_NAME=week3-eks

eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --version 1.30 \
  --nodegroup-name ng-default \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 3 \
  --managed
```

## 2.2 验证集群连接
```bash
aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME
kubectl get nodes
kubectl get pods -A
```

你应该看到 2 个 `Ready` 节点。

## 2.3 你必须看懂的输出
1. `kubectl get nodes`：检查节点是否 Ready。
2. `kubectl get pods -A`：系统组件是否正常（如 `kube-system` 下组件）。

## 2.4 模块2 常见报错
1. `AccessDeniedException`：IAM 权限不够。
2. `NoCredentialProviders`：本地 AWS 凭证没配置好。
3. 节点不 Ready：通常是网络/权限/配额问题，先看 `eksctl utils describe-stacks` 和 CloudFormation 事件。

## 2.5 模块2 验收
1. EKS 集群创建成功。
2. `kubectl get nodes` 显示 Ready。
3. 你能描述“控制面托管、数据面自管”的含义。

---

## 模块3：ECR 镜像仓库实战

## 3.1 创建 ECR 仓库
```bash
export ECR_REPO=week3-api
aws ecr create-repository --repository-name $ECR_REPO --region $AWS_REGION
```

## 3.2 登录 ECR
```bash
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.$AWS_REGION.amazonaws.com
```

把 `<ACCOUNT_ID>` 换成你的 AWS 账号 ID（可用 `aws sts get-caller-identity` 查看）。

## 3.3 构建并打标签
在你第2周的 `api-demo` 目录：
```bash
docker build -t week3-api:v1 .
docker tag week3-api:v1 <ACCOUNT_ID>.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:v1
```

## 3.4 推送镜像
```bash
docker push <ACCOUNT_ID>.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:v1
```

## 3.5 验证镜像已上传
```bash
aws ecr describe-images --repository-name $ECR_REPO --region $AWS_REGION
```

## 3.6 模块3 验收
1. ECR 仓库创建成功。
2. 镜像成功 push。
3. 你能在控制台看到镜像 tag `v1`。

---

## 模块4：部署应用到 EKS

## 4.1 创建命名空间
`k8s/namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: week3
```

## 4.2 创建 Deployment（使用 ECR 镜像）
`k8s/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: week3-api
  namespace: week3
spec:
  replicas: 2
  selector:
    matchLabels:
      app: week3-api
  template:
    metadata:
      labels:
        app: week3-api
    spec:
      containers:
      - name: api
        image: <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/week3-api:v1
        ports:
        - containerPort: 5000
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 15
```

把 `<ACCOUNT_ID>` 和 `<REGION>` 换成真实值。

## 4.3 创建 Service
`k8s/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: week3-api-svc
  namespace: week3
spec:
  selector:
    app: week3-api
  ports:
  - port: 80
    targetPort: 5000
  type: ClusterIP
```

## 4.4 应用部署
```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

kubectl get pods -n week3
kubectl get svc -n week3
kubectl get endpoints -n week3
```

关键检查：`endpoints` 不为空，否则说明 Service 没匹配到 Pod。

## 4.5 模块4 验收
1. Pod 全部 Running/Ready。
2. Service 和 Endpoints 正常。
3. 你能解释“为什么 Service selector 很关键”。

---

## 模块5：ALB Ingress Controller（对外访问）

## 5.1 先理解它做什么
AWS Load Balancer Controller 会把 K8s Ingress 资源转换成 AWS ALB。

你写的是 Ingress YAML，AWS 帮你创建 ALB 并配置监听/路由规则。

## 5.2 安装前提
1. 集群有 OIDC Provider。
2. Controller 需要 IAM 权限（后面 模块6 用 IRSA 强化）。

关联 OIDC：
```bash
eksctl utils associate-iam-oidc-provider --region $AWS_REGION --cluster $CLUSTER_NAME --approve
```

## 5.3 用 Helm 安装 controller（学习路径）
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

kubectl create namespace kube-system || true

helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=true \
  --set region=$AWS_REGION \
  --set vpcId=<YOUR_VPC_ID>
```

> `vpcId` 可用 `aws ec2 describe-vpcs` 查询，选你的 EKS 所在 VPC。

## 5.4 创建 Ingress
`k8s/ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: week3-api-ing
  namespace: week3
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: week3-api-svc
            port:
              number: 80
```

应用并检查：
```bash
kubectl apply -f k8s/ingress.yaml
kubectl get ingress -n week3
```

等待几分钟后，`ADDRESS` 应出现 ALB DNS。

## 5.5 验证公网访问
```bash
curl http://<ALB_DNS>/health
```

## 5.6 模块5 常见问题
1. Ingress 一直没地址：Controller 未就绪或权限不足。
2. 返回 503：Service 无 endpoints 或探针未 Ready。
3. ALB 有地址但访问超时：SG/NACL/子网标签问题。

## 5.7 模块5 验收
1. `kubectl get ingress` 有 ALB 地址。
2. 外网访问 `/health` 成功。
3. 你能讲清 Ingress 到 ALB 的映射关系。

---

## 模块6：IRSA（IAM Role for Service Account）

## 6.1 为什么需要 IRSA
不推荐把 AK/SK 放进容器里。IRSA 通过 OIDC 把 K8s ServiceAccount 映射到 IAM Role，让 Pod 临时获取权限。

优点：
1. 不存长期密钥。
2. 权限按服务粒度隔离。
3. 审计更清晰。

## 6.2 基本流程
1. 集群关联 OIDC（模块5 已做）。
2. 创建 IAM Policy（服务需要什么权限就给什么）。
3. 创建 IAM Role 并绑定 ServiceAccount。
4. Pod 使用该 ServiceAccount 运行。

## 6.3 示例：给 Pod S3 只读权限
1) 创建策略文件 `s3-readonly-policy.json`：
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::<YOUR_BUCKET>",
        "arn:aws:s3:::<YOUR_BUCKET>/*"
      ]
    }
  ]
}
```

2) 创建 IAM Policy：
```bash
aws iam create-policy \
  --policy-name Week3S3ReadOnlyPolicy \
  --policy-document file://s3-readonly-policy.json
```

3) 创建并绑定 ServiceAccount：
```bash
eksctl create iamserviceaccount \
  --name week3-api-sa \
  --namespace week3 \
  --cluster $CLUSTER_NAME \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/Week3S3ReadOnlyPolicy \
  --approve
```

4) 在 deployment 里指定：
```yaml
spec:
  serviceAccountName: week3-api-sa
```

## 6.4 验证 IRSA 生效
1. 进入 Pod 执行 AWS API（例如列 S3）。
2. 预期：有权限的动作成功，没授权的动作失败。

## 6.5 模块6 验收
1. 你能解释 IRSA 原理。
2. 你能描述为什么它比硬编码密钥更安全。
3. 你知道 ServiceAccount 与 IAM Role 的绑定关系。

---

## 模块7：总演练 + 排错复盘

## 7.1 一次完整演练
按顺序复做：
1. 建集群（可跳过，若已存在）。
2. 建 ECR 并 push 镜像。
3. 部署 Deployment + Service。
4. 部署 Ingress 并拿到 ALB 地址。
5. 验证外网访问。
6. 绑定 IRSA（至少理解并走一遍命令）。

## 7.2 标准排错顺序（必须记住）
1. `kubectl get pods -n week3`（Pod 状态）
2. `kubectl describe pod <pod> -n week3`（事件）
3. `kubectl logs <pod> -n week3`（应用日志）
4. `kubectl get svc,endpoints -n week3`（后端是否存在）
5. `kubectl get ingress -n week3`（ALB 地址是否下发）
6. Controller 日志（`kube-system`）和 AWS 控制台事件

## 7.3 周复盘模板
1. 我能解释的 15 个 EKS 相关术语。
2. 我独立执行过的关键命令。
3. 我踩到的 3 个坑和修复过程。
4. 我对第4周（EKS networking/security）的问题清单。

---

## 公司沟通可直接用的话术
1. “镜像先推 ECR，再由 EKS 拉取，确保部署版本可追溯。”
2. “Service selector 和 pod label 先对齐，不然 Ingress 后面没有 endpoints。”
3. “ALB Ingress Controller 本质是把 Ingress 资源翻译成 ALB 规则。”
4. “Pod 权限建议用 IRSA，不要塞长期 AK/SK 到容器里。”
5. “发布异常先看 pod 事件和日志，再看 service/endpoints，最后看 ingress 和 controller。”

---

## 第3周自测题（含答案）

## 题目
1. EKS 托管了 K8s 的哪一部分？你主要管哪一部分？
2. NodeGroup 和 Fargate 的核心差异是什么？
3. ECR 在整条链路中解决什么问题？
4. 为什么 Service endpoints 为空会导致 503？
5. ALB Ingress Controller 和普通 Ingress 的关系是什么？
6. IRSA 解决了什么安全问题？
7. 为什么不建议在 Pod 里放长期 AK/SK？
8. 发布后访问失败，最先查哪三条命令？
9. `kubectl get ingress` 有地址但访问超时，通常看哪几块？
10. 写出完整外部流量路径。

## 参考答案
1. EKS 托管控制面；你主要管理节点和工作负载。
2. NodeGroup 基于 EC2 管理节点；Fargate 无需管节点。
3. ECR 提供镜像仓库，支持版本化、拉取和部署。
4. 因为 Service 没有后端 Pod 可转发。
5. Controller 负责把 Ingress 规则真正落地到 ALB。
6. 用临时凭证和服务级权限替代硬编码密钥。
7. 泄漏风险高、轮换困难、审计不清晰。
8. `kubectl get pods`、`kubectl describe pod`、`kubectl logs`。
9. 看 SG/NACL、子网标签、ALB target 健康状态。
10. `Internet -> ALB(Ingress) -> Service -> Pod`。

---

## 本周完成标准（80%）
满足以下 6 条即合格：
1. EKS 集群创建并可用。
2. 镜像能推送到 ECR。
3. 应用成功部署到 EKS。
4. Ingress 创建 ALB 并可外网访问。
5. 能解释 IRSA 机制和优势。
6. 能按固定顺序排查至少一次部署故障。

---

## 费用与收尾（必须执行）
学习结束后，避免忘记删资源：
```bash
eksctl delete cluster --name $CLUSTER_NAME --region $AWS_REGION
```
同时检查并清理：
1. ECR 仓库（不需要可删除）。
2. ALB/NLB 是否残留。
3. CloudWatch 日志组是否需要保留。


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
