# AWS + Docker + Kubernetes 基础讲义（小白版）

## 你只需要先记住这一句话
这一周你要掌握的是：
1. AWS 基础资源怎么组成一套可运行环境。
2. Docker 怎么把一个 API 打包并跑起来。
3. Kubernetes 里 Pod、Service、Ingress 各干什么。

学完后你在公司里至少能做到：
1. 听懂 80% 基础术语。
2. 能独立跑起一个容器化 API。
3. 能在讨论里准确说出 `Ingress -> Service -> Pod` 的流量路径。

---

## 学习方式（非常重要）
每天按下面三步走：
1. 先看“知识讲解”（20-30 分钟）。
2. 再做“动手实操”（40-60 分钟）。
3. 最后做“复述训练”（10 分钟，把今天内容讲出来）。

不要只看不做。你觉得“懂了”不算懂，做出来才算。

---

## 模块0：环境准备（30-60 分钟）

## 0.1 安装工具
在终端执行：
```bash
aws --version
docker --version
kubectl version --client
git --version
```

如果命令报 `command not found`，说明没装好。优先保证 `aws` 和 `docker` 可用。

## 0.2 AWS 账户安全设置
1. 开启 MFA。
2. 创建预算告警（建议 20 USD/月）。
3. 日常不用 root 用户，后续都用 IAM 用户/角色。

## 0.3 本周目标产物
到周末你必须交付这 3 个东西：
1. `docker run` 跑起来的 API（`/health` 返回 `{"status":"ok"}`）。
2. 一张手画图：`Internet -> Ingress -> Service -> Pod`。
3. 一页总结：IAM、VPC、EC2、S3、CloudWatch、Docker、K8s 核心对象各是什么。

---

## 模块1：Linux 和 Docker 入门

## 1.1 先理解 Linux 常用命令（必须会）
1. `pwd`：我现在在哪个目录。
2. `ls -la`：看目录文件详情。
3. `cd <dir>`：切换目录。
4. `cat <file>`：看文件内容。
5. `grep <keyword> <file>`：查关键字。
6. `ps aux | grep <name>`：查进程。
7. `curl <url>`：发 HTTP 请求。

你在 infra 组天天会用它们。

## 1.2 Docker 核心概念（先不深究原理）
1. Image（镜像）：程序模板，像“安装包”。
2. Container（容器）：镜像启动后的运行实例，像“正在运行的进程”。
3. Registry：镜像仓库（比如 ECR/Docker Hub）。

一句话：`镜像是静态包，容器是运行中的实例`。

## 1.3 第一次跑容器
```bash
docker run -d --name hello-nginx -p 8080:80 nginx
curl http://localhost:8080
docker ps
docker logs hello-nginx
docker stop hello-nginx
docker rm hello-nginx
```

你要理解 `-p 8080:80`：
1. 左边 `8080` 是你本机端口。
2. 右边 `80` 是容器内部端口。
3. 浏览器访问本机 `8080`，会转发到容器 `80`。

## 1.4 模块1 常见报错
1. `port is already allocated`：8080 被占用，换成 `-p 8081:80`。
2. `Cannot connect to the Docker daemon`：Docker Desktop 没启动。

## 1.5 模块1 验收
1. 你能说清 image 和 container 区别。
2. 你能解释端口映射。
3. 你能自己停止并删除容器。

---

## 模块2：IAM（身份和权限）

## 2.1 IAM 是什么
IAM 是 AWS 的“门禁系统”。它决定：
1. 谁可以访问（身份）。
2. 可以做什么（权限）。

## 2.2 三个核心对象
1. User：给人长期用。
2. Role：给服务或临时身份用（生产推荐）。
3. Policy：权限规则（JSON）。

## 2.3 你必须理解的两条原则
1. 最小权限（Least Privilege）：只给刚好够用的权限。
2. 显式拒绝优先：如果有 Deny，哪怕 Allow 也没用。

## 2.4 用 AWS CLI 验证身份
先配置：
```bash
aws configure
```
依次输入：
1. Access Key ID
2. Secret Access Key
3. Region（例如 `us-east-1`）
4. Output format（`json`）

再验证：
```bash
aws sts get-caller-identity
```
预期能看到 `Account`、`Arn`、`UserId`。

## 2.5 模块2 高概率被问到的问题
1. 为什么生产不用长期 AK/SK？
答：泄漏风险高，轮换难，审计复杂。优先用 Role + 临时凭证。
2. Policy 放在哪？
答：可以挂在 User、Group、Role 上，业务服务一般挂 Role。

## 2.6 模块2 验收
1. 你能讲清 User、Role、Policy 差异。
2. 你能在本地跑通 `aws sts get-caller-identity`。

---

## 模块3：VPC + EC2（网络和计算）

## 3.1 VPC 你要这样理解
VPC 就是你在 AWS 里的私有网络。

VPC 里最核心的 4 个东西：
1. Subnet：子网（公有或私有）。
2. Route Table：路由表（流量往哪走）。
3. Internet Gateway（IGW）：访问公网的出入口。
4. Security Group（SG）：实例级防火墙。

## 3.2 公有子网和私有子网
1. 公有子网：路由到 IGW，可直接出公网。
2. 私有子网：不直接到 IGW，通常经 NAT 出网。

## 3.3 SG 为什么说“有状态”
1. 入站放行 80 端口。
2. 返回流量自动允许，不需要再单独放回包规则。

## 3.4 启动一台最小 EC2
你在控制台创建时重点看：
1. AMI：Linux（如 Amazon Linux）。
2. 实例规格：`t3.micro`（学习够用）。
3. 网络：选择默认 VPC 的公有子网。
4. SG：只开必要端口（SSH 22，临时测试可开 80）。
5. 密钥对：下载好 `.pem`。

## 3.5 SSH 连接（Mac/Linux）
```bash
chmod 400 your-key.pem
ssh -i your-key.pem ec2-user@<PUBLIC_IP>
```

## 3.6 模块3 常见故障排查顺序
1. 先看 EC2 是否 Running。
2. 再看 SG 是否放行端口。
3. 再看子网路由是否连到 IGW。
4. 最后看应用本身是否启动。

## 3.7 模块3 验收
1. 你能讲清公网访问 EC2 的路径。
2. 你知道 SG 是状态防火墙。

---

## 模块4：S3 + CloudWatch（存储和观测）

## 4.1 S3 先掌握这几个词
1. Bucket：桶，全球唯一名字。
2. Object：对象，一个文件就是一个对象。
3. Prefix：对象前缀（类似目录效果）。

S3 不是传统硬盘，不支持“随机写文件块”那套语义，它是对象存储。

## 4.2 S3 常见使用场景
1. 日志归档。
2. 静态资源存储。
3. 训练/推理数据文件存储。

## 4.3 CloudWatch 三件套
1. Metrics：看趋势（CPU、请求数、错误数）。
2. Logs：看具体文本日志。
3. Alarms：指标超阈值触发通知。

## 4.4 实操任务
1. 创建一个 Bucket，上传一个文本文件。
2. 在 EC2 上配置 CPU 告警：`CPUUtilization > 70%`，持续 `5 minutes`。

## 4.5 你必须能说出的结论
1. 指标负责“发现异常趋势”。
2. 日志负责“定位具体报错”。
3. 告警负责“第一时间通知”。

## 4.6 模块4 验收
1. 你能进控制台找到 S3 对象。
2. 你能在 CloudWatch 找到 EC2 CPU 曲线。

---

## 模块5：从 0 写一个 API 并 Docker 化（核心实战）

今天是本周最重要的一天。

## 5.1 创建项目目录
```bash
mkdir -p ~/projects/platform_learning/labs/01-api-demo
cd ~/projects/platform_learning/labs/01-api-demo
```

## 5.2 写 API 代码
创建 `app.py`：
```python
from flask import Flask

app = Flask(__name__)

@app.get("/health")
def health():
    return {"status": "ok"}

@app.get("/")
def home():
    return {"message": "week1 api running"}
```

## 5.3 写依赖文件
创建 `requirements.txt`：
```txt
flask==3.0.3
```

## 5.4 写 Dockerfile（逐行理解）
创建 `Dockerfile`：
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py /app/app.py
EXPOSE 5000
CMD ["flask", "--app", "app", "run", "--host=0.0.0.0", "--port=5000"]
```

逐行解释：
1. `FROM`：基础镜像。
2. `WORKDIR`：后续命令工作目录。
3. `COPY requirements.txt`：先拷依赖文件，有利于缓存。
4. `RUN pip install`：安装依赖。
5. `COPY app.py`：再拷代码。
6. `EXPOSE 5000`：声明容器监听端口。
7. `CMD`：容器启动命令。

## 5.5 构建并运行
```bash
docker build -t week1-api:v1 .
docker run -d --name week1-api -p 5000:5000 week1-api:v1
curl http://localhost:5000/health
curl http://localhost:5000/
```

预期输出：
1. `/health` 返回 `{"status":"ok"}`。
2. `/` 返回 `{"message":"week1 api running"}`。

## 5.6 看日志和进入容器
```bash
docker logs week1-api
docker exec -it week1-api sh
```

## 5.7 模块5 常见报错
1. `No module named flask`：依赖没装，检查 `requirements.txt` 和 build 日志。
2. `Address already in use`：5000 被占用，改成 `-p 5001:5000`。
3. `curl` 连接失败：先 `docker ps` 看容器是否在运行。

## 5.8 模块5 验收
1. 容器内 API 可访问。
2. 你能解释 Dockerfile 每一行作用。

---

## 模块6：Kubernetes 核心对象讲透（Pod/Service/Ingress）

今天目标不是搭集群，而是把概念彻底搞明白。

## 6.1 Pod 是什么
1. K8s 最小调度单位。
2. 一个 Pod 内可有多个容器，但最常见是 1 个主容器。
3. Pod IP 会变化，不适合作为稳定访问地址。

你可以把 Pod 理解为“临时工位”。

## 6.2 Service 是什么
1. 给一组 Pod 提供稳定访问入口。
2. 通过 Label Selector 找到目标 Pod。
3. 常见类型：
   - `ClusterIP`：只在集群内访问。
   - `NodePort`：在每个 Node 开端口。
   - `LoadBalancer`：云厂商分配外部 LB。

Service 解决的是“Pod 地址会变”的问题。

## 6.3 Ingress 是什么
1. 七层（HTTP/HTTPS）入口规则。
2. 按域名和路径转发到不同 Service。
3. Ingress 自己不转发流量，实际由 Ingress Controller 执行。

## 6.4 三者关系（必须背）
1. Pod 跑应用。
2. Service 稳定地找到 Pod。
3. Ingress 把外部请求路由到 Service。

固定流量路径：`Internet -> Ingress -> Service -> Pod`。

## 6.5 最小 YAML 示例（读懂就够）
Pod：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  labels:
    app: demo
spec:
  containers:
  - name: demo
    image: nginx
    ports:
    - containerPort: 80
```

Service：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-svc
spec:
  selector:
    app: demo
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

Ingress：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ing
spec:
  rules:
  - host: demo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo-svc
            port:
              number: 80
```

## 6.6 模块6 验收
1. 你能口头解释为什么“不能直接拿 Pod 对外提供长期服务”。
2. 你能说出 Ingress 与 Service 不是替代关系，而是上下游关系。

---

## 模块7：复盘输出（把知识变成你自己的）

## 7.1 写 1 页复盘（模板）
1. 今天我能解释的术语有哪些。
2. 我亲手完成了哪些命令和操作。
3. 遇到过哪些错误，我怎么排查。
4. 下周我要提前补哪些概念。

## 7.2 3 分钟口述训练稿（照着练）
“这一周我主要掌握了 AWS 基础和容器基础。IAM 负责身份和权限，生产推荐 Role + 最小权限。VPC 是私有网络，子网加路由决定流量路径，SG 是有状态防火墙。EC2 是云主机。S3 是对象存储。CloudWatch 里 metrics 看趋势，logs 看细节，alarm 触发通知。容器方面，image 是模板，container 是运行实例。我用 Flask 写了一个健康检查 API，并用 Dockerfile 构建镜像运行成功。K8s 里 Pod 跑应用，Service 提供稳定访问，Ingress 做外部七层路由，路径是 Ingress 到 Service 再到 Pod。”

## 7.3 模块7 验收
1. 你能不看稿讲完上面这段话。
2. 你能回答别人追问 2-3 个细节问题。

---

## 本周高频术语词典（开会会反复出现）
1. Least Privilege：最小权限。
2. Security Group：实例级防火墙，有状态。
3. Public Subnet：可路由到公网的子网。
4. Private Subnet：通常不直接暴露公网。
5. Image：容器镜像模板。
6. Container：镜像启动后的运行实例。
7. Pod：K8s 最小调度单元。
8. Service：Pod 稳定访问入口。
9. Ingress：HTTP/HTTPS 外部入口规则。
10. Metrics/Logs/Alarm：趋势/细节/通知。

---

## 实战排错手册（先看这个顺序）

## A. Docker 访问失败
1. `docker ps` 看容器在不在。
2. `docker logs <name>` 看应用是否报错。
3. `docker port <name>` 看端口映射对不对。
4. `curl localhost:<port>` 本机连通性验证。

## B. AWS CLI 调不通
1. `aws sts get-caller-identity` 验证凭证。
2. 检查 region 是否填错。
3. 检查 IAM policy 是否有对应权限。

## C. EC2 连接不上
1. 实例是否 Running。
2. SG 是否放行端口。
3. 子网路由是否接到 IGW。
4. 密钥和用户名是否正确。

---

## 第1周自测题（含答案）

## 题目
1. User、Role、Policy 的区别是什么？
2. 什么是最小权限？为什么重要？
3. VPC、Subnet、Route Table 各负责什么？
4. SG 为什么叫有状态防火墙？
5. S3 和本地磁盘最大的区别是什么？
6. Metrics、Logs、Alarm 分别解决什么问题？
7. Docker image 和 container 的关系是什么？
8. `docker run -p 5000:80` 的含义是什么？
9. 为什么 Service 不能省略直接用 Pod？
10. Ingress、Service、Pod 的标准流量路径是什么？

## 参考答案
1. User 给人长期使用，Role 给服务/临时身份使用，Policy 是权限规则。
2. 最小权限是只授予任务所需最少权限，能降低误操作和泄漏风险。
3. VPC 是私有网络，Subnet 是网络分区，Route Table 决定流量路径。
4. 因为回包自动允许，不必单独配置返回规则。
5. S3 是对象存储，不是块存储文件系统语义。
6. Metrics 看趋势，Logs 看细节，Alarm 负责通知。
7. Image 是模板，Container 是运行中的实例。
8. 本机 5000 端口转发到容器 80 端口。
9. Pod IP 会变，Service 提供稳定地址和负载分发。
10. `Internet -> Ingress -> Service -> Pod`。

---

## 本周完成标准（达到 80%）
你满足下面 5 条，就算第1周合格：
1. AWS CLI 身份校验通过。
2. 本地 Docker API 跑通。
3. 会看 Docker 日志并排查常见错误。
4. 能讲清 IAM/VPC/EC2/S3/CloudWatch 的基本作用。
5. 能完整解释 Pod/Service/Ingress 与流量路径。


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
