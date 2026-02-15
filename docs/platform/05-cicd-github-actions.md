# CI/CD + GitHub Actions 讲义（从手动发布到自动发布）

## 本周目标
第5周结束后你应该能做到：
1. 解释 CI、CD、Pipeline、Artifact、Runner、Environment 的含义。
2. 在 GitHub Actions 里搭建一条可用流水线：`lint/test -> build -> push ECR -> deploy EKS(dev)`。
3. 为 dev/stage/prod 设计基础分层规则（分支、变量、审批）。
4. 理解“发布失败如何止损”：失败即停、健康检查、手动审批闸门。
5. 在公司里能听懂并使用：`workflow_dispatch`、`secrets`、`OIDC`、`matrix`、`concurrency`。

---

## 先讲核心概念（先背）
1. CI（持续集成）：代码合并前后，自动跑检查和测试，尽快发现问题。
2. CD（持续交付/部署）：通过自动化把构建产物部署到环境。
3. Artifact：构建产物（镜像、包等）。
4. Pipeline：一条按步骤执行的自动化流程。
5. Runner：执行 workflow 的机器。

一句话：`CI 保证“能合”、CD 保证“能发”`。

---

## 讲义知识地图（模块）
1. 模块1：CI/CD 架构与 GitHub Actions 基础。
2. 模块2：代码质量门禁（lint/test）与触发策略。
3. 模块3：构建镜像并推送 ECR。
4. 模块4：自动部署到 EKS dev。
5. 模块5：环境分层（dev/stage/prod）+ secrets 管理。
6. 模块6：发布保护（并发控制、审批、失败阻断）。
7. 模块7：完整演练 + 故障演练 + 复盘。

---

## 模块1：GitHub Actions 基础

## 1.1 workflow 文件位置
GitHub Actions 配置放在：
`.github/workflows/*.yml`

最小 workflow 结构：
```yaml
name: ci-cd

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: echo "hello"
```

## 1.2 触发方式你要会区分
1. `push`：代码推送触发。
2. `pull_request`：PR 触发（常用于 CI）。
3. `workflow_dispatch`：手动触发（常用于上线）。
4. `schedule`：定时任务。

## 1.3 Job 与 Step
1. Job 是一组步骤，默认并行。
2. Step 是单个动作。
3. 用 `needs` 指定依赖顺序。

## 1.4 模块1 验收
1. 你能创建一个最小 workflow 并跑成功。
2. 你能解释 on/jobs/steps 的关系。

---

## 模块2：先把 CI 立起来（lint + test）

## 2.1 为什么先做 CI
因为“发布失败”的大头来自低级错误：语法错、测试不过、依赖问题。CI 要在部署前拦住它们。

## 2.2 示例：Python 项目的 CI 工作流
创建 `.github/workflows/ci.yml`：
```yaml
name: ci

on:
  pull_request:
    branches: ["main"]
  push:
    branches: ["main"]

jobs:
  lint-test:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Setup Python
      uses: actions/setup-python@v5
      with:
        python-version: "3.11"

    - name: Install deps
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install pytest ruff

    - name: Lint
      run: ruff check .

    - name: Test
      run: pytest -q
```

## 2.3 触发策略建议
1. PR：必须通过 CI 才允许合并。
2. main push：再跑一次 CI，防止分支差异。

## 2.4 模块2 验收
1. lint 和 test 都在 PR 中自动执行。
2. 你能解释“为什么 CI 失败要阻断合并”。

---

## 模块3：构建镜像并推送 ECR

## 3.1 登录 AWS 的两种方法
1. 传统 AK/SK secrets（简单但长期凭证风险高）。
2. GitHub OIDC 假设角色（推荐，生产常用）。

你先用 secrets 跑通，再升级 OIDC。

## 3.2 需要的 GitHub Secrets
在 repo settings -> Secrets and variables -> Actions 添加：
1. `AWS_ACCESS_KEY_ID`
2. `AWS_SECRET_ACCESS_KEY`
3. `AWS_REGION`（例如 `us-east-1`）
4. `AWS_ACCOUNT_ID`
5. `ECR_REPOSITORY`（例如 `week3-api`）

## 3.3 构建并推 ECR 的 workflow
创建 `.github/workflows/build-push.yml`：
```yaml
name: build-push

on:
  push:
    branches: ["main"]

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}

    - name: Login to ECR
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build and push image
      env:
        ACCOUNT: ${{ secrets.AWS_ACCOUNT_ID }}
        REGION: ${{ secrets.AWS_REGION }}
        REPO: ${{ secrets.ECR_REPOSITORY }}
        TAG: ${{ github.sha }}
      run: |
        IMAGE_URI=$ACCOUNT.dkr.ecr.$REGION.amazonaws.com/$REPO:$TAG
        docker build -t $IMAGE_URI .
        docker push $IMAGE_URI
        echo "IMAGE_URI=$IMAGE_URI" >> $GITHUB_ENV

    - name: Print image uri
      run: echo $IMAGE_URI
```

## 3.4 你要记住的实践
1. 镜像 tag 用 `github.sha`，可追溯到代码提交。
2. 不要只用 `latest`。

## 3.5 模块3 验收
1. main push 后，ECR 出现新镜像。
2. tag 对应 commit SHA。

---

## 模块4：自动部署到 EKS dev

## 4.1 准备 kubeconfig
如果你使用自托管 runner，可提前有 kubeconfig。
如果是 GitHub-hosted runner，通常在 workflow 里调用 AWS API 生成 kubeconfig。

## 4.2 示例：部署 job（接 模块3）
创建 `.github/workflows/deploy-dev.yml`：
```yaml
name: deploy-dev

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: "Image tag to deploy"
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: dev

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}

    - name: Setup kubectl
      uses: azure/setup-kubectl@v4

    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig --name week3-eks --region ${{ secrets.AWS_REGION }}

    - name: Deploy image to dev namespace
      env:
        ACCOUNT: ${{ secrets.AWS_ACCOUNT_ID }}
        REGION: ${{ secrets.AWS_REGION }}
        REPO: ${{ secrets.ECR_REPOSITORY }}
        TAG: ${{ github.event.inputs.image_tag }}
      run: |
        IMAGE_URI=$ACCOUNT.dkr.ecr.$REGION.amazonaws.com/$REPO:$TAG
        kubectl -n dev set image deployment/week3-api api=$IMAGE_URI
        kubectl -n dev rollout status deployment/week3-api --timeout=180s
```

## 4.3 如果你还没 `dev` namespace
```bash
kubectl create namespace dev
```

## 4.4 模块4 验收
1. 你能手动触发 `deploy-dev`。
2. 部署后 `rollout status` 成功。
3. 服务可访问。

---

## 模块5：环境分层（dev/stage/prod）

## 5.1 为什么必须分层
1. dev：快速试错。
2. stage：接近生产的验证。
3. prod：对外服务，变更要谨慎。

## 5.2 推荐分支策略（简版）
1. feature/* -> PR -> main。
2. main 自动部署 dev。
3. 打 tag（如 `v1.2.0`）触发 stage/prod 发布流程。

## 5.3 GitHub Environments
在 repo 设置里创建：
1. `dev`
2. `stage`
3. `prod`

可配置：
1. 环境 secrets（不同环境不同值）。
2. Required reviewers（prod 发布前人工审批）。

## 5.4 环境变量与密钥分离
1. 普通配置放 Variables。
2. 敏感信息放 Secrets。
3. 不同环境各自维护，避免串环境。

## 5.5 模块5 验收
1. 你能说明 dev/stage/prod 的职责。
2. 你知道如何在 GitHub Environment 做 prod 审批。

---

## 模块6：发布保护（稳定性）

## 6.1 并发控制（防止重复发布）
```yaml
concurrency:
  group: deploy-dev
  cancel-in-progress: true
```

作用：同一环境同时只跑一个部署任务。

## 6.2 失败即停
1. 默认 step 失败会中断 job。
2. 不要随便用 `continue-on-error: true`。

## 6.3 关键步骤设置超时
```yaml
timeout-minutes: 20
```

避免卡死占 runner。

## 6.4 部署后健康检查
```yaml
- name: Post-deploy smoke test
  run: |
    curl -f http://<YOUR_DEV_URL>/health
```

`-f` 让 4xx/5xx 直接失败。

## 6.5 发布流程建议
1. CI 不过，禁止部署。
2. 部署后 smoke test 不过，标记失败。
3. prod 必须审批。

## 6.6 模块6 验收
1. 你能说明并发控制的意义。
2. 你能加一个部署后 health check。

---

## 模块7：完整演练 + 故障演练

## 7.1 你要完成的完整链路
1. 提交代码 -> CI 通过。
2. main push -> 镜像 build/push 到 ECR。
3. 手动触发 deploy-dev。
4. rollout 成功 + health check 通过。

## 7.2 故障演练（必须做）
1. 故意把镜像 tag 写错，观察部署失败日志。
2. 故意让 `/health` 返回 500，观察 smoke test 拦截。
3. 故意改错 namespace，观察 `kubectl set image` 报错并修复。

## 7.3 复盘模板
1. 我这周学会了哪些 pipeline 关键节点。
2. 我最常见的失败点是什么。
3. 我如何在 workflow 里增加护栏避免同类失败。
4. 下周我会把部署方式升级到 Helm/GitOps。

---

## 公司里可直接说的话术
1. “镜像 tag 我用 commit SHA，发布可回溯。”
2. “CI 必须作为 deploy 前置门禁，不通过不发版。”
3. “prod 发布走 environment 审批，避免误发。”
4. “deploy 后我会跑 smoke test，不通过立即阻断。”
5. “同一环境开启 concurrency，避免并发发布互相覆盖。”

---

## 第5周自测题（含答案）

## 题目
1. CI 和 CD 的核心差异是什么？
2. 为什么镜像 tag 不建议只用 latest？
3. GitHub Actions 的 job 和 step 有什么关系？
4. main 分支自动部署到哪个环境更合理，为什么？
5. 为什么 prod 要加人工审批？
6. `concurrency` 解决什么问题？
7. 部署后为什么还要做 smoke test？
8. workflow 里 secrets 和 variables 怎么分工？
9. rollout 失败时你第一步看什么？
10. 一条最小可用 CI/CD 链路包含哪些阶段？

## 参考答案
1. CI 保证代码可合并，CD 保证产物可部署。
2. latest 不可追溯，问题定位和回滚困难。
3. job 是步骤集合，step 是具体动作。
4. dev，风险低，便于快速验证。
5. 降低误操作风险，增加变更可控性。
6. 防止同环境多个部署并发冲突。
7. 防止“部署成功但服务不可用”。
8. secrets 放敏感信息，variables 放普通配置。
9. 看 Actions 日志和 `kubectl rollout status/describe/logs` 输出。
10. lint/test -> build -> push artifact -> deploy -> verify。

---

## 第5周完成标准（80%）
满足以下 7 条即合格：
1. 你能独立写一个基础 CI workflow。
2. 你能把镜像自动 push 到 ECR。
3. 你能手动触发并完成 dev 部署。
4. 你能做部署后健康检查。
5. 你能设计 dev/stage/prod 分层方案。
6. 你能加上并发控制与审批保护。
7. 你能定位并修复一次流水线失败。

---

## 第6周衔接
下周你会把“命令式部署”升级为“声明式发布”：
1. 用 Helm 管理版本和配置。
2. 用 GitOps 管理环境状态。
3. 建立标准回滚策略（发布失败自动回退）。


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
