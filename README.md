# doki-doc 文档仓库规范

这个仓库只用于存放 Doki 网站加载的文档内容。

## 目录结构
```txt
docs/
  index.md
  <category>/
    <doc>.md
```

示例：
```txt
docs/
  index.md
  platform/
    01-aws-docker-k8s-foundation.md
  ai/
    prompt-basics.md
  product/
    prd-template.md
```

## 规则
1. 所有文档必须放在 `docs/` 目录下。
2. 一级目录代表分类（如 `platform/`、`ai/`、`product/`）。
3. 文档文件必须是 `.md`。
4. 文件名建议使用 `kebab-case`，并可用数字前缀控制顺序（如 `01-xxx.md`）。
5. `docs/index.md` 作为总览入口，建议维护目录链接。

## 更新流程（必须走 PR）
1. 从 `main` 新建分支。
2. 在 `docs/` 下新增或修改 markdown。
3. 提交 PR 并通过审核后合并。
4. 合并后会触发 webhook，Doki 站点自动刷新文档缓存，无需重新部署。

## 不要提交的内容
- 代码、脚本、构建产物
- 二进制大文件（除非明确需要）
- `docs/` 目录外的业务文件
