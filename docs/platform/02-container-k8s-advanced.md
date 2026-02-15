# 容器进阶 + Kubernetes 实操基础讲义（EKS 前置）

## 这一周你会达到什么水平
第2周结束后你应该能做到：
1. 不依赖模板，自己写出可用的 Dockerfile（含优化思路）。
2. 看懂并修改基础 K8s YAML（Deployment/Service/Ingress）。
3. 解释并使用 ConfigMap、Secret、Namespace、Probe、Resource Limit。
4. 能在公司讨论里听懂这些词：`replica`、`rolling update`、`liveness/readiness`、`requests/limits`。
5. 为第3周 EKS 上云部署打好“术语和动作”基础。

---

## 学习方式（和第1周一致）
每天三步：
1. 先读讲解（20-30 分钟）。
2. 再做实操（60 分钟左右）。
3. 最后复述（10 分钟）。

你的重点依然是“动手”。

---

## 本周总览（7天）
1. 模块1：Dockerfile 进阶与镜像优化。
2. 模块2：K8s 核心对象进阶（Deployment/Service）。
3. 模块3：Ingress、Namespace、Label、Selector。
4. 模块4：ConfigMap/Secret 与环境变量注入。
5. 模块5：健康检查与资源限制（生产必会）。
6. 模块6：滚动发布、回滚、排错流程。
7. 模块7：周复盘 + mini 演练（本地 K8s 一套完整 YAML）。

---

## 模块1：Dockerfile 进阶与镜像优化

## 1.1 你要先理解 4 个问题
1. 为什么镜像越小越好？
答：拉取更快、启动更快、漏洞面更小。
2. 为什么依赖文件先 COPY？
答：利用构建缓存，代码改了不用每次重装依赖。
3. 为什么不要把 AK/SK 写进镜像？
答：镜像会被推到仓库，密钥会泄露。
4. 为什么要固定依赖版本？
答：避免同样代码在不同时间构建出不同结果。

## 1.2 写一个更规范的 Dockerfile
在 `~/projects/platform_learning/labs/02-api-demo/` 创建：

`app.py`
```python
from flask import Flask
import os

app = Flask(__name__)

@app.get("/health")
def health():
    return {"status": "ok"}

@app.get("/")
def home():
    env = os.getenv("APP_ENV", "dev")
    return {"message": "week2 api", "env": env}
```

`requirements.txt`
```txt
flask==3.0.3
gunicorn==22.0.0
```

`Dockerfile`
```dockerfile
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r /app/requirements.txt

COPY app.py /app/app.py

EXPOSE 5000
CMD ["gunicorn", "-b", "0.0.0.0:5000", "app:app"]
```

`.dockerignore`
```txt
.git
__pycache__
*.pyc
.env
venv
```

## 1.3 构建和运行
```bash
mkdir -p ~/projects/platform_learning/labs/02-api-demo
cd ~/projects/platform_learning/labs/02-api-demo
# 把上面文件保存后执行
docker build -t week2-api:v1 .
docker run -d --name week2-api -p 5000:5000 -e APP_ENV=local week2-api:v1
curl http://localhost:5000/
curl http://localhost:5000/health
```

## 1.4 模块1 验收
1. 你能讲清 `.dockerignore` 作用。
2. 你能说出为什么用 `gunicorn` 而不是 flask 开发服务器。
3. API 能返回 `env: local`。

---

## 模块2：Deployment + Service（K8s 最核心二件套）

## 2.1 先讲 Deployment
Deployment 负责“声明式管理副本和更新策略”。

你要记住：
1. Pod 会挂，Deployment 会自动拉新 Pod。
2. `replicas` 控制副本数。
3. 更新镜像时，Deployment 默认滚动更新。

## 2.2 写最小 Deployment YAML
创建 `k8s/deployment.yaml`：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: week2-api
  namespace: week2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: week2-api
  template:
    metadata:
      labels:
        app: week2-api
    spec:
      containers:
      - name: api
        image: week2-api:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
```

## 2.3 写 Service YAML
创建 `k8s/service.yaml`：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: week2-api-svc
  namespace: week2
spec:
  selector:
    app: week2-api
  ports:
  - port: 80
    targetPort: 5000
  type: ClusterIP
```

## 2.4 你必须理解 selector
Service 通过 `selector` 找 Pod。只要 label 对不上，Service 就“没有后端”。

---

## 模块3：Ingress + Namespace + Label 体系

## 3.1 Namespace 是什么
Namespace 是集群里的“逻辑隔离空间”。

常见做法：
1. `dev`、`staging`、`prod` 分命名空间。
2. 团队 A/B 分命名空间。

## 3.2 Label 是什么
Label 是资源标签，用于：
1. 让 Service 找到 Pod。
2. 筛选查询对象。
3. 组织治理策略。

## 3.3 Ingress 是什么（再强化）
Ingress 是七层路由规则，常按域名/路径转发。

创建 `k8s/ingress.yaml`：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: week2-api-ing
  namespace: week2
spec:
  rules:
  - host: week2.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: week2-api-svc
            port:
              number: 80
```

## 3.4 模块3 验收
1. 你能解释 namespace 的作用。
2. 你能解释 label/selector 为什么重要。
3. 你能复述 Ingress 到 Service 到 Pod 的路径。

---

## 模块4：ConfigMap + Secret（配置和密钥分离）

## 4.1 为什么要分离配置
把配置写死在镜像里会导致：
1. 换环境要重打镜像。
2. 配置泄漏和变更风险高。

## 4.2 ConfigMap 用法
创建 `k8s/configmap.yaml`：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: week2-api-config
  namespace: week2
data:
  APP_ENV: "dev"
  LOG_LEVEL: "info"
```

## 4.3 Secret 用法（注意：默认只是 base64，不是强加密）
创建 `k8s/secret.yaml`：
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: week2-api-secret
  namespace: week2
type: Opaque
stringData:
  API_TOKEN: "replace-me"
```

## 4.4 在 Deployment 里注入环境变量
把 deployment 里 container 段改成：
```yaml
      containers:
      - name: api
        image: week2-api:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
        envFrom:
        - configMapRef:
            name: week2-api-config
        - secretRef:
            name: week2-api-secret
```

## 4.5 模块4 验收
1. 你能说出 ConfigMap 和 Secret 的区别。
2. 你知道 Secret 默认不是“绝对安全”，仍需 RBAC/加密/审计配合。

---

## 模块5：健康检查 + 资源限制（生产必会）

## 5.1 健康检查为什么重要
1. Liveness Probe：进程“活着吗”，挂了就重启。
2. Readiness Probe：能“接流量吗”，不健康就先摘流。

没有 probe 的服务，线上会出现“已启动但不可用”。

## 5.2 资源限制为什么重要
1. requests：调度时保证的最小资源。
2. limits：容器最多可用资源。

不配资源限制容易导致“抢占资源、影响邻居”。

## 5.3 Deployment 增强版片段
在容器配置中增加：
```yaml
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
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "300m"
            memory: "256Mi"
```

## 5.4 模块5 验收
1. 你能解释 readiness 和 liveness 的区别。
2. 你能解释 requests/limits 的作用。

---

## 模块6：滚动发布、回滚与排错流程

## 6.1 你要理解的发布术语
1. Rolling Update：逐步替换老 Pod，不中断服务。
2. Rollback：新版本异常时回退到旧版本。
3. ReplicaSet：Deployment 每个版本背后的副本集。

## 6.2 基础命令（先记住）
```bash
kubectl get pods -n week2
kubectl get svc -n week2
kubectl get ingress -n week2
kubectl describe pod <pod-name> -n week2
kubectl logs <pod-name> -n week2
kubectl rollout status deployment/week2-api -n week2
kubectl rollout history deployment/week2-api -n week2
kubectl rollout undo deployment/week2-api -n week2
```

## 6.3 标准排错顺序
1. 看 Pod 是否 Running/Ready。
2. 看 `kubectl describe` 事件。
3. 看应用日志。
4. 看 Service selector 是否匹配 Pod label。
5. 看 Ingress 后端是否正确。

## 6.4 模块6 验收
1. 你能说出“发布失败时第一步看什么”。
2. 你知道怎么回滚 Deployment。

---

## 模块7：本周总演练（把所有点串起来）

## 7.1 演练目标
完成一套完整资源定义：
1. Namespace
2. ConfigMap
3. Secret
4. Deployment（含 probes + resources）
5. Service
6. Ingress

## 7.2 建议文件结构
在 `~/projects/platform_learning/labs/02-api-demo/k8s/` 放这些文件：
1. `namespace.yaml`
2. `configmap.yaml`
3. `secret.yaml`
4. `deployment.yaml`
5. `service.yaml`
6. `ingress.yaml`

`namespace.yaml` 内容：
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: week2
```

## 7.3 周复盘模板
1. 我现在能讲清的 10 个术语。
2. 我本周做过的 10 个命令。
3. 我踩过的 3 个坑和修复方法。
4. 我对 EKS 还不懂的 5 个问题（下周解决）。

---

## 公司开会你可以直接说的句子
1. “这个服务先补 readiness/liveness，不然发布后可能接流量异常。”
2. “我先核对 deployment label 和 service selector 是否一致。”
3. “配置建议用 ConfigMap/Secret 注入，不要打进镜像。”
4. “如果新版本异常，先看 rollout 状态和 pod 事件，再决定回滚。”
5. “资源 requests/limits 需要先定一个安全基线，避免抢占。”

---

## 第2周自测题（含答案）

## 题目
1. Deployment 和 Pod 的关系是什么？
2. 为什么 Service 需要 selector？
3. Namespace 在团队协作里有什么作用？
4. ConfigMap 和 Secret 的用途差异？
5. 为什么说 Secret 默认不等于“绝对安全”？
6. readinessProbe 与 livenessProbe 的区别？
7. requests 与 limits 的区别？
8. 发布后服务不可用，先查哪 3 个地方？
9. Ingress 自己能转发流量吗？
10. 为什么要做滚动发布和回滚？

## 参考答案
1. Deployment 管理 Pod 副本和更新策略，Pod 是运行实例。
2. selector 用于把 Service 绑定到目标 Pod。
3. Namespace 用于逻辑隔离环境/团队，避免资源混乱。
4. ConfigMap 放普通配置，Secret 放敏感信息。
5. 因为默认仅 base64 编码，还需配合 RBAC 和加密等措施。
6. readiness 决定是否接流量，liveness 决定是否重启容器。
7. requests 是调度保证，limits 是资源上限。
8. Pod 状态/事件、应用日志、Service selector 与 endpoints。
9. 不能，需 Ingress Controller 实际执行路由。
10. 为了降低发布风险并可快速恢复稳定版本。

---

## 第2周完成标准（达到 80%）
满足以下 6 条即合格：
1. 能独立写出可运行 Dockerfile（含依赖与启动命令）。
2. 能写 Deployment + Service 并解释 selector。
3. 能使用 ConfigMap/Secret 注入配置。
4. 能解释并配置 readiness/liveness。
5. 能解释并配置 requests/limits。
6. 能说出 rollout/rollback 基本命令和排错顺序。

---

## 给第3周的过渡提醒（EKS）
你下周要把“本地/概念层”迁移到 AWS EKS：
1. 镜像将从本地转到 ECR。
2. 集群从本地环境变成托管 EKS。
3. Ingress 会接入 AWS ALB Controller。
4. 权限会涉及 IAM Role for Service Account（IRSA）。


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
