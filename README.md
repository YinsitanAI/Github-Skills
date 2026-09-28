# knowledgeT 两人协同开发与容器部署教程

> **适用对象**：YinsitanAI 与 xianyupro 两个 GitHub 账号共同开发 knowledgeT 项目
> 
> **覆盖范围**：私有 / 公开项目 × 代码 / 镜像 × 双方 token，以及 Fork 自动同步、自动发布镜像、服务器部署和系统化问题排查

---

## 目录

1. 约定与基础概念
2. 核心原则：先授权，后用 token
3. 权限与 token 全景
4. 选择协作模式
5. 模式 A：共享仓库协作（推荐）
6. 模式 B：Fork + 自动同步 + 自动发布镜像
7. 镜像权限配置
8. 服务器部署
9. 问题排查体系
10. Arcane 面板部署专题
11. 附录：验收清单与命令速查

---

## 1. 约定与基础概念

### 1.1 示例约定

| 项目  | 取值  | 说明  |
| --- | --- | --- |
| 主仓库 | `YinsitanAI/knowledgeT` | 负责发布镜像，部署以它为准 |
| 上游仓库（仅模式 B） | `xianyupro/knowledgeT` | Fork 的来源 |
| 镜像地址 | `ghcr.io/yinsitanai/knowledget-backend`<br>`ghcr.io/yinsitanai/knowledget-frontend` | ghcr.io 地址**必须全小写** |
| 服务器部署目录 | `/opt/knowledget` | 与代码目录分开 |
| 占位符 | `<你的用户名>`、`<TOKEN>` | 替换为实际值 |

> 如果实际方向相反（xianyupro 是主仓库、YinsitanAI 是 fork），把文中两个账号名互换即可，步骤完全相同。

### 1.2 三类资源，三套权限

| 资源  | 管什么 | 权限在哪里设置 |
| --- | --- | --- |
| **代码仓库** | 谁能读代码、谁能推送代码 | 仓库 Settings → Collaborators |
| **容器镜像**（ghcr.io） | 谁能拉取、谁能推送镜像 | 镜像的 Package settings |
| **自动化**（GitHub Actions） | 工作流以什么身份运行 | 工作流文件 + 仓库 Secrets |

这三套权限**相互独立**（除非镜像开启了"继承仓库权限"）。实践中绝大多数"拉不下来""推不上去"的问题，都是把这三套权限混为一谈造成的。

### 1.3 组织账号的特殊性

如果 YinsitanAI 是**组织**（Organization）而不是个人账号，需要注意：

- **组织本身不能生成 token。** 文中"YinsitanAI 的 token"指组织成员（通常是管理员）用自己账号生成的 token。
- **SSO 授权**：如果组织启用了 SAML SSO，token 生成后还要在 token 列表中点 **Configure SSO → Authorize** 授权给该组织。
- **组织策略**：组织可以禁止 classic token 访问组织资源（组织 Settings → Personal access tokens），也可以禁止 fork 私有仓库、限制创建私有镜像。遇到"明明权限都对却被拒绝"时，要检查组织设置。

---

## 2. 核心原则：先授权，后用 token

### 2.1 token 使用优先级

| 优先级 | 做法  | 适用场景 |
| --- | --- | --- |
| **1. 首选** | 用**自己的账号和自己的 token**；需要访问对方资源时，请对方把权限**授予你的账号** | 所有场景 |
| **2. 次选** | 在 GitHub Actions 中使用自动生成的 `GITHUB_TOKEN`，不需要任何人的个人 token | 在本仓库内构建、发布镜像 |
| **3. 兜底** | 使用对方的 token | 对方确实无法给你的账号授权时 |

一句话：**解决访问问题的正确方式是"授权"，不是"共享 token"。**

### 2.2 不得不使用对方 token 时的底线

- **最小权限**：只读，只包含一个仓库（fine-grained token 选 `Contents: Read-only`；拉镜像的 classic token 只勾 `read:packages`）。
- **设置有效期**：不要选 No expiration。
- **安全存放**：只存放在 Actions Secret 或服务器上权限为 600 的文件里；不写进代码、不发在聊天里、不留在 shell 历史中。
- **及时撤销**：授权问题一旦解决，立即让对方撤销该 token。

### 2.3 安全使用 token 的习惯

```bash
# 把 token 放进临时变量，屏幕不显示、不进入历史记录
read -s T

# ……使用 $T 执行命令……

# 用完立即清除
unset T
```

`docker login` 时不要把 token 写在命令行参数里，让它交互式提示输入 Password 即可。

---

## 3. 权限与 token 全景

### 3.1 四种凭证对比

| 凭证  | 前缀 / 来源 | 能否跨两个所有者 | 能否用于 ghcr.io | 适合用途 |
| --- | --- | --- | --- | --- |
| **classic token** | `ghp_` | 能（账号有权限的仓库都能访问） | **能** | 拉取 / 推送镜像、跨仓库同步 |
| **fine-grained token** | `github_pat_` | 不能（只能选一个资源所有者） | **不能** | 只访问单个所有者的代码 |
| **GITHUB_TOKEN** | Actions 自动生成 | 不能（只能访问当前仓库） | 能，但只能推送到**当前仓库所有者**名下 | 在 Actions 中构建发布镜像 |
| **gh CLI 登录 / SSH 密钥** | `gh auth login` / `ssh-keygen` | 取决于账号权限 | 不用于 ghcr.io | 本地日常 git 操作 |

三条必须记住的规则：

1. **ghcr.io 只接受 classic token。** fine-grained token 执行 `docker login` 会显示成功，但拉取任何镜像都会返回 `denied`。
2. **`docker login` 成功只说明 token 有效，不代表有拉取权限。**
3. **用 `GITHUB_TOKEN` 推送代码不会触发其他工作流。** 需要"推送后自动构建"的链路，推送时必须用个人 token。

### 3.2 按用途选择 token

| 用途  | 使用谁的凭证 | 类型与权限 |
| --- | --- | --- |
| 本地 clone / push 代码 | 开发者本人 | `gh auth login` 或 SSH 密钥（推荐，免管理 token） |
| 服务器拉取私有镜像 | 部署者本人 | classic，只勾 `read:packages` |
| 本地手动推送镜像 | 推送者本人 | classic，勾 `write:packages`（已包含读取） |
| Actions 构建发布镜像 | 无需个人 token | `GITHUB_TOKEN`，工作流中声明 `packages: write` |
| Actions 同步上游并推送到 fork（`SYNC_TOKEN`） | fork 方成员本人 | classic，勾 `repo` 和 `workflow` |
| Actions 读取私有上游（`UPSTREAM_TOKEN`，兜底） | 上游所有者 | fine-grained，只选 knowledgeT，`Contents: Read-only` |

> 生成 classic token 的路径：GitHub → **Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic)**。

### 3.3 公开 / 私有 × 代码 / 镜像

|     | 代码  | 镜像  |
| --- | --- | --- |
| **公开项目** | 任何人可读、可 fork；推送需要协作者权限 | 可以把镜像设为 Public，任何人**无需登录**即可拉取。注意：**设为 Public 后不能再改回 Private** |
| **私有项目** | 读和写都需要协作者或组织成员权限；fork 出来的仓库也是私有的，且依赖对上游的访问权限 | 必须登录；token 必须是 classic 且含 `read:packages`；账号必须对该镜像有权限 |

> 镜像的可见性不一定和仓库一致，以镜像 Package 页面显示的为准。

### 3.4 双方 token 都有时，该用哪一个

判断规则：**操作谁的资源，就用对该资源有权限的本人身份；登录的账号必须对要拉取的镜像命名空间有权限。**

| 操作  | 首选  | 兜底  | 不可行 |
| --- | --- | --- | --- |
| 读取 `xianyupro/knowledgeT` 代码 | YinsitanAI 成员本人的 token（前提：已被加为协作者） | xianyupro 的只读 fine-grained token | —   |
| 推送到 `YinsitanAI/knowledgeT` | 对该仓库有写权限的本人账号（成员或协作者） | —   | xianyupro 的 fine-grained token（无法跨所有者） |
| 发布 `ghcr.io/yinsitanai/*` | YinsitanAI 仓库 Actions 的 `GITHUB_TOKEN` | 成员本人 classic `write:packages` 手动推送 | 在其他仓库的 Actions 中推送（报 `installation does not exist`） |
| 发布 `ghcr.io/xianyupro/*` | xianyupro 仓库 Actions 的 `GITHUB_TOKEN` | xianyupro 本人手动推送 | 同上  |
| 拉取 `ghcr.io/yinsitanai/*` | 部署者本人 classic `read:packages`（账号有该镜像权限） | 有权限的对方 classic token | 任何 fine-grained token |
| 拉取 `ghcr.io/xianyupro/*` | 部署者本人 classic（xianyupro 已给该账号授权） | xianyupro 的 classic `read:packages` | 任何 fine-grained token |

---

## 4. 选择协作模式

### 4.1 两种模式对比

|     | 模式 A：共享仓库 | 模式 B：Fork + 自动同步 |
| --- | --- | --- |
| 仓库数量 | 1 个 | 2 个（上游 + fork） |
| 推送权限 | 双方都是同一仓库的协作者 | 各自推送自己的仓库，通过 PR 贡献代码 |
| 镜像  | 1 套 | 每个仓库各发布一套 |
| token 需求 | 每人一个自己的 token | 额外需要同步用的 token |
| 复杂度 | 低   | 较高（同步、冲突、跨仓库权限） |
| 适用场景 | 双方长期共同维护同一个产品 | 一方只贡献代码或需要定制版本；无法获得写权限；需要独立部署 |

**建议：能用模式 A 就用模式 A。** 模式 B 适合双方仓库必须分开的情况。

### 4.2 模式 A 流程

```mermaid
flowchart LR
  X["xianyupro 本地"] -- "功能分支 + PR" --> R["YinsitanAI/knowledgeT"]
  Y["YinsitanAI 成员本地"] -- "功能分支 + PR" --> R
  R -- "合并到 main 触发构建" --> G["ghcr.io/yinsitanai/knowledget-*"]
  G -- "部署者用自己的 token 拉取" --> S["服务器"]
```

### 4.3 模式 B 流程

```mermaid
flowchart LR
  U["上游 xianyupro/knowledgeT"] -- "每 2 小时自动同步" --> F["Fork YinsitanAI/knowledgeT"]
  F -- "同步推送触发构建" --> G["ghcr.io/yinsitanai/knowledget-*"]
  F -- "功能分支 PR 贡献代码" --> U
  G -- "部署者用自己的 token 拉取" --> S["服务器"]
```

---

## 5. 模式 A：共享仓库协作（推荐）

### 5.1 仓库所有者初始化（YinsitanAI 操作）

**第 1 步：创建仓库**

创建 `YinsitanAI/knowledgeT`，按需要选择 Private 或 Public。

**第 2 步：添加协作者**

- **组织仓库**：Settings → **Collaborators and teams** → **Add people** → 输入 `xianyupro` → 角色选 **Write**（需要管理设置时选 Maintain）。
- **个人仓库**：Settings → **Collaborators** → **Add people** → 输入 `xianyupro`。个人仓库的协作者固定拥有写权限，不能选择角色。

**第 3 步：保护 main 分支**

Settings → **Rules → Rulesets**（或 **Branches → Add branch protection rule**），目标分支选 `main`，建议勾选：

- **Require a pull request before merging**（可设 Required approvals = 1）
- **Require status checks to pass**（选择镜像构建检查）
- **Block force pushes**

> 注意：私有仓库在 GitHub 免费计划下**不支持**分支保护，需要 Pro / Team 计划。不支持时，靠团队约定"不直接推送 main"。

**第 4 步：添加镜像发布工作流**

见 5.4 节。

**第 5 步：配置镜像权限**

首次发布成功后，进入镜像的 Package settings：私有项目确认勾选"继承仓库权限"，公开项目按需改为 Public。见第 7 节。

### 5.2 协作者加入（xianyupro 操作）

**第 1 步：接受邀请**

点击邮件中的链接，或打开：

```
https://github.com/YinsitanAI/knowledgeT/invitations
```

**第 2 步：本地认证（二选一）**

使用 GitHub CLI（推荐，不用手动管理 token）：

```bash
gh auth login
```

按提示选择 `GitHub.com` → `HTTPS` → 浏览器登录，然后执行：

```bash
gh auth setup-git
```

或者配置 SSH 密钥，按 GitHub 官方文档添加到自己的账号。

**第 3 步：克隆仓库**

```bash
git clone https://github.com/YinsitanAI/knowledgeT.git
cd knowledgeT
```

### 5.3 日常开发流程（双方相同）

**1. 从最新的 main 创建功能分支**

```bash
git switch main
git pull
git switch -c feat/简短描述
```

分支命名建议：`feat/` 新功能、`fix/` 修复、`docs/` 文档。

**2. 开发并提交**

```bash
git add -A
git commit -m "feat: 简要说明改动"
git push -u origin feat/简短描述
```

**3. 创建 PR**

```bash
gh pr create --fill
```

也可以在网页上操作。PR 描述里写清改了什么、如何测试。

**4. 评审与合并**

另一方评审 → CI 构建通过 → 选择 **Squash and merge** → 删除远程分支。

**5. 合并后双方同步本地**

```bash
git switch main
git pull
git branch -d feat/简短描述
```

**开发中途 main 有了新提交时**，先把功能分支更新到最新：

```bash
git fetch origin
git rebase origin/main
git push --force-with-lease
```

**协作约定：**

- 不直接推送 main，所有改动都走 PR。
- 小步提交，一个 PR 只做一件事。
- 修改公共文件（Dockerfile、docker-compose、工作流）前先和对方沟通。
- `.env` 等含密钥的文件加入 `.gitignore`，绝不提交。

### 5.4 镜像发布工作流

在仓库中创建 `.github/workflows/docker-publish.yml`：

```yaml
# 构建并发布前后端镜像
# - PR:只构建不推送,用于检查
# - 推送到 main:发布 latest 和 sha-xxxxxxx 标签
# - 推送 v1.2.3 这样的标签:发布 1.2.3 版本标签
# 镜像命名空间自动取当前仓库所有者(转小写),同一文件在上游和 fork 中都能直接使用
name: 构建并发布镜像

on:
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - service: backend
            context: ./backend
            build_args: ""
          - service: frontend
            context: ./frontend
            build_args: BACKEND_URL=http://backend:5001
    steps:
      - uses: actions/checkout@v4

      - name: 计算镜像名
        run: echo "IMAGE=ghcr.io/${GITHUB_REPOSITORY_OWNER,,}/knowledget-${{ matrix.service }}" >> "$GITHUB_ENV"

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        if: github.event_name != 'pull_request'
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE }}
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=sha
            type=semver,pattern={{version}}
            type=ref,event=pr

      - uses: docker/build-push-action@v6
        with:
          context: ${{ matrix.context }}
          build-args: ${{ matrix.build_args }}
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha,scope=${{ matrix.service }}
          cache-to: type=gha,mode=max,scope=${{ matrix.service }}
```

**这个工作流的三个设计要点：**

1. **镜像命名空间不写死。** `${GITHUB_REPOSITORY_OWNER,,}` 会自动取当前仓库所有者并转成小写。在 YinsitanAI 仓库中运行就发布到 `ghcr.io/yinsitanai`，在 xianyupro 仓库中运行就发布到 `ghcr.io/xianyupro`。写死命名空间的工作流在 fork 中运行时会报 `installation does not exist`。
2. **自动关联仓库。** `metadata-action` 会给镜像加上 `org.opencontainers.image.source` 标签，镜像因此自动关联到源仓库，可以继承仓库权限。
3. **构建参数写入镜像。** 前端的 `BACKEND_URL` 在构建时写入镜像，修改后必须重新构建才会生效。

**发布后的标签：**

| 触发方式 | 生成的标签 | 用途  |
| --- | --- | --- |
| 推送到 main | `latest`、`sha-abc1234` | 日常部署 / 回滚到某次提交 |
| 推送 `v1.2.3` 标签 | `1.2.3` | 正式版本 |
| PR  | 只构建，不推送 | 检查能否构建成功 |

### 5.5 发布正式版本

```bash
git switch main
git pull
git tag v1.0.0
git push origin v1.0.0
```

---

## 6. 模式 B：Fork + 自动同步 + 自动发布镜像

**目标**：上游 `xianyupro/knowledgeT` 有更新 → fork `YinsitanAI/knowledgeT` 自动同步 → 自动构建并发布 `ghcr.io/yinsitanai/*` → 服务器拉取更新。

### 6.1 创建 fork

在上游仓库页面点击 **Fork**，Owner 选择 `YinsitanAI`。

**私有仓库需要注意：**

- 上游仓库要允许 fork（仓库 Settings → General → **Allow forking**）；如果 fork 到组织，组织设置也要允许 fork 私有仓库。
- 私有仓库的 fork 同样是私有的。
- fork 依赖对上游的访问权限，权限被移除后 fork 可能随之被删除，重要代码要另有备份。

### 6.2 启用 Actions 并处理上游工作流

**第 1 步：启用 Actions**

fork 出来的仓库默认禁用 Actions。打开 fork 的 **Actions** 标签页，点击绿色的启用按钮。

**第 2 步：处理上游自带的工作流**

如果上游的工作流把镜像地址写死为 `ghcr.io/xianyupro/...`，它在 fork 中运行时一定会失败。处理方法：

- **最佳做法**：请上游改用 5.4 节的可移植工作流。同一个文件在上游和 fork 中都能正常运行，fork 不需要额外的发布工作流，同步时也不会冲突。
- **上游暂时不改时**：在 fork 的 Actions 页面禁用它（点击该工作流 → 右上角 **··· → Disable workflow**），然后在 fork 中另外添加 5.4 节的工作流，文件名用 `docker-publish-fork.yml`，避免与上游文件同名冲突。

> **不要直接修改或删除上游的工作流文件**，否则每次同步都可能产生合并冲突。禁用状态会一直保留，同步不会重新启用它。

### 6.3 配置同步用的 token（按优先级选择）

**方案 1（首选）：只用 fork 方自己的 token**

1. xianyupro 在 `xianyupro/knowledgeT` 的 **Settings → Collaborators** 中，把 YinsitanAI 的管理员账号（下文称"成员账号"）加为协作者。
2. 成员账号接受邀请后，生成自己的 classic token，勾选 **repo** 和 **workflow**。
3. 在 fork 仓库 **Settings → Secrets and variables → Actions → New repository secret** 中添加：名称 `SYNC_TOKEN`，值为这个 token。

这样一个 token 就能同时读取上游、推送到 fork。

**方案 2：上游是公开仓库**

读取上游不需要任何权限，只需按方案 1 的第 2、3 步配置 `SYNC_TOKEN`。

**方案 3（兜底）：上游私有，且无法把成员账号加为协作者**

在方案 1 的基础上，再添加一个上游 token：

1. xianyupro 生成 **fine-grained token**：Repository access 只选 `knowledgeT`，Permissions 中 **Contents 设为 Read-only**，设置有效期。
2. 在 fork 仓库中添加 Secret：名称 `UPSTREAM_TOKEN`，值为这个 token。
3. `SYNC_TOKEN` 仍然必须是 fork 方成员自己的 classic token（xianyupro 的 token 没有 fork 仓库的写权限）。

**配置前先验证 token 能否读取上游**（在任意 Linux 机器上执行）：

```bash
read -s T
```

```bash
git ls-remote "https://x-access-token:$T@github.com/xianyupro/knowledgeT.git" | head -3
```

```bash
unset T
```

输出几行提交哈希说明 token 可用；返回 `403` 或 `not found` 说明这个 token 没有上游仓库的权限。

### 6.4 同步工作流

在 fork 仓库中创建 `.github/workflows/fork-sync.yml`：

```yaml
# 定时把上游的更新合并到本仓库 main 分支
# 需要 Secret:
#   SYNC_TOKEN      必填,fork 方成员自己的 classic token(repo + workflow)
#   UPSTREAM_TOKEN  选填,上游只读 token;不填则用 SYNC_TOKEN 读取上游
# 必须用个人 token 推送,才能触发镜像构建工作流
name: 同步上游仓库

on:
  schedule:
    - cron: "0 */2 * * *"   # 每 2 小时检查一次
  workflow_dispatch:         # 允许在 Actions 页面手动运行

jobs:
  sync:
    # 只在 fork 中运行,即使此文件被合并到上游也不会执行
    if: github.repository != 'xianyupro/knowledgeT'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: main
          fetch-depth: 0
          persist-credentials: false   # 不保留默认凭证,下面为两个仓库分别指定 token

      - name: 合并上游 main 分支
        env:
          UPSTREAM_REPO: xianyupro/knowledgeT
          SYNC_TOKEN: ${{ secrets.SYNC_TOKEN }}
          UPSTREAM_TOKEN: ${{ secrets.UPSTREAM_TOKEN }}
        run: |
          UP_TOKEN="${UPSTREAM_TOKEN:-$SYNC_TOKEN}"
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git remote add upstream "https://x-access-token:${UP_TOKEN}@github.com/${UPSTREAM_REPO}.git"
          git remote set-url origin "https://x-access-token:${SYNC_TOKEN}@github.com/${GITHUB_REPOSITORY}.git"
          git fetch upstream main
          git merge --no-edit upstream/main
          git push origin main
```

**关键设计说明：**

| 设计  | 原因  |
| --- | --- |
| `persist-credentials: false` | checkout 默认会保留一份凭证，后续所有 github.com 请求都会带上它。如果这份凭证没有上游权限，读取上游会报 `403 Write access to repository not granted` |
| 两个仓库分别指定 token | 读取上游和推送 fork 可能需要不同的身份 |
| 推送使用 `SYNC_TOKEN` | 用 `GITHUB_TOKEN` 推送不会触发镜像构建工作流 |
| 需要 `workflow` 权限 | 上游修改了 `.github/workflows` 下的文件时，没有这个权限会推送失败 |

**测试**：打开 Actions → **同步上游仓库** → **Run workflow**。

- 上游有新提交：日志显示合并成功，随后自动触发"构建并发布镜像"。
- 上游没有新提交：日志显示 `Already up to date`，不会触发构建，这是正常的。

### 6.5 完整的自动更新链路

```
上游有新提交
  → 最多 2 小时后，同步工作流合并并推送到 fork（使用 SYNC_TOKEN）
  → 推送触发镜像构建工作流（使用 GITHUB_TOKEN）
  → 发布 ghcr.io/yinsitanai/knowledget-*:latest
  → 服务器执行 docker compose pull && docker compose up -d
```

**Actions 注意事项：**

- 定时任务只在默认分支（main）上运行。
- 公开仓库连续 60 天没有任何活动，定时任务会被自动停用。
- 私有仓库的 Actions 有免费时长限制。每 2 小时检查一次大约每月消耗 360 分钟，构建另算。想节省时长可改为 `"0 */6 * * *"`（每 6 小时一次）。

### 6.6 从 fork 向上游贡献代码

**第 1 步：本地添加上游地址**

```bash
git remote add upstream https://github.com/xianyupro/knowledgeT.git
git fetch upstream
```

**第 2 步：基于上游的 main 创建功能分支**

```bash
git switch -c feat/简短描述 upstream/main
```

必须基于 `upstream/main` 而不是 fork 的 main 创建分支。fork 的 main 里有 `fork-sync.yml` 等 fork 专用文件，基于它创建分支会把这些文件带进 PR。

**第 3 步：开发、推送到 fork**

```bash
git add -A
git commit -m "feat: 简要说明改动"
git push -u origin feat/简短描述
```

**第 4 步：创建 PR**

在网页上创建 PR：base 选 `xianyupro/knowledgeT` 的 `main`，compare 选 `YinsitanAI/knowledgeT` 的功能分支。

### 6.7 同步冲突的处理

同步工作流在 `git merge` 这一步失败，说明 fork 和上游修改了同一处代码，需要手动合并：

```bash
git clone https://github.com/YinsitanAI/knowledgeT.git
cd knowledgeT
git remote add upstream https://github.com/xianyupro/knowledgeT.git
git fetch upstream
git merge upstream/main
```

打开提示冲突的文件，删除 `<<<<<<<`、`=======`、`>>>>>>>` 标记并保留正确内容，然后：

```bash
git add -A
git commit
git push origin main
```

**避免冲突的方法**：不要在 fork 中修改上游的文件。需要定制的配置放在 `.env` 或单独的文件中。

---

## 7. 镜像权限配置

### 7.1 找到镜像设置页

| 镜像所有者 | 镜像列表地址 |
| --- | --- |
| 组织  | `https://github.com/orgs/YinsitanAI/packages` |
| 个人  | `https://github.com/xianyupro?tab=packages` |

点进镜像 → 右侧栏 **Package settings**。backend 和 frontend 是两个独立的镜像，**要分别设置**。

### 7.2 Package settings 各部分的作用

| 区域  | 作用  | 操作建议 |
| --- | --- | --- |
| **Repository source** | 镜像关联的源仓库，由镜像的 `org.opencontainers.image.source` 标签决定 | 确认是正确的仓库。这里也能看出镜像真正属于哪个仓库 |
| **Manage Actions access** | 哪些仓库的工作流可以读写这个镜像 | 发布仓库默认有 Admin 权限。其他仓库的工作流需要使用这个镜像时，在这里添加 |
| **Inherited access** | 勾选 **Inherit access from source repository** 后，仓库的协作者和成员自动获得对应的镜像权限 | **推荐保持勾选** |
| **Manage access** | 取消继承后，在这里单独给人授权 | 只在需要和仓库权限分开管理时使用 |
| **Danger Zone** | 修改可见性、删除镜像 | 公开项目在这里改为 Public |

### 7.3 私有镜像：给对方拉取权限

**已勾选继承仓库权限时（推荐）：**

不要在镜像页面操作，直接到仓库里加人：仓库 **Settings → Collaborators and teams → Add people**，角色选 **Read** 就能拉取。

**未勾选继承时：**

在 **Manage access** 中添加对方的 GitHub 用户名，角色选 **Read**。

**对方拉取时**：用自己的账号和自己的 classic token（勾选 `read:packages`）登录 ghcr.io。

### 7.4 公开镜像

Package settings → **Danger Zone → Change visibility → Public**。公开后任何人都可以免登录拉取。

> 此操作**不可逆**，公开后不能再改回私有。私有项目的镜像不要公开，镜像里包含了完整的程序代码。

---

## 8. 服务器部署

### 8.1 部署原则

1. **部署目录与代码目录分开。** 服务器上只放 `docker-compose.yml` 和 `.env`，不放源码，也不在 git 克隆的目录里直接部署。
2. **部署者用自己的账号和 token 登录 ghcr.io。**
3. **镜像来源用变量控制。** 切换镜像来源或回滚版本时只改 `.env`，不改 compose 文件。

### 8.2 部署文件

**`docker-compose.yml`**

```yaml
# knowledgeT 生产部署:只拉取镜像,不需要源码
# 镜像来源由 .env 中的 IMAGE_OWNER 和 IMAGE_TAG 控制

x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

services:
  backend:
    image: ghcr.io/${IMAGE_OWNER:-yinsitanai}/knowledget-backend:${IMAGE_TAG:-latest}
    container_name: knowledget-backend
    ports:
      # 只监听本机;前端通过 Docker 内部网络 http://backend:5001 访问后端
      - "127.0.0.1:5001:5001"
    environment:
      - FLASK_ENV=production
      - DEEPSEEK_API_KEY=${DEEPSEEK_API_KEY:?请在 .env 中设置 DEEPSEEK_API_KEY}
      - DEEPSEEK_BASE_URL=${DEEPSEEK_BASE_URL:-https://api.deepseek.com}
      - DEEPSEEK_MODEL=${DEEPSEEK_MODEL:-deepseek-chat}
    volumes:
      - backend_data:/app/instance
      - backend_uploads:/app/uploads
    restart: unless-stopped
    logging: *default-logging

  frontend:
    image: ghcr.io/${IMAGE_OWNER:-yinsitanai}/knowledget-frontend:${IMAGE_TAG:-latest}
    container_name: knowledget-frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend
    restart: unless-stopped
    logging: *default-logging

volumes:
  backend_data:
  backend_uploads:
```

**`.env`**

```bash
# 必填
DEEPSEEK_API_KEY=sk-xxxx

# 镜像来源(全小写),默认 yinsitanai;要使用 xianyupro 名下的镜像时改为 xianyupro
# IMAGE_OWNER=yinsitanai

# 镜像标签,默认 latest;回滚时改为 sha-xxxxxxx 或版本号如 1.0.0
# IMAGE_TAG=latest

# 可选
# DEEPSEEK_BASE_URL=https://api.deepseek.com
# DEEPSEEK_MODEL=deepseek-chat
```

**配置说明：**

- 后端端口只绑定 `127.0.0.1`。Docker 发布的端口会绕过 ufw / firewalld 等防火墙，写成 `5001:5001` 等于把后端直接暴露到公网。如果浏览器需要直接访问后端，再改回 `5001:5001`。
- 日志轮转限制每个容器最多保留 30MB 日志，防止长期运行写满磁盘。
- 数据卷名称带有部署目录名作为前缀（例如 `knowledget_backend_data`），**更换部署目录会导致使用新的空数据卷**。

**仓库中的本地构建文件 `docker-compose.build.yml`（开发用，服务器不需要）**

```yaml
# 在有源码的机器上本地构建:
#   docker compose -f docker-compose.yml -f docker-compose.build.yml up -d --build
# 需要 .env 中已设置 DEEPSEEK_API_KEY,否则解析配置时会报错
services:
  backend:
    build: ./backend
  frontend:
    build:
      context: ./frontend
      args:
        BACKEND_URL: http://backend:5001
```

### 8.3 首次部署

**1. 创建部署目录**

```bash
mkdir -p /opt/knowledget
cd /opt/knowledget
```

**2. 放入部署文件**

在本机上传：

```bash
scp docker-compose.yml root@服务器IP:/opt/knowledget/
```

或者在服务器上新建文件，粘贴 8.2 节的内容：

```bash
nano docker-compose.yml
```

**3. 创建 .env 并限制权限**

```bash
nano .env
```

```bash
chmod 600 .env
```

**4. 用自己的账号登录 ghcr.io**

```bash
docker login ghcr.io -u <你的用户名>
```

Password 处粘贴**你自己的** classic token（勾选 `read:packages`），看到 `Login Succeeded` 即可。

> `docker login` 和后面的 `docker compose` 必须由**同一个用户**执行。root 登录的凭证保存在 `/root/.docker/config.json`，普通用户的保存在自己的 `~/.docker/config.json`，两者互不相通。

**5. 确认镜像地址**

```bash
docker compose config --images
```

应该显示两个 `ghcr.io/yinsitanai/knowledget-...` 地址。

**6. 拉取并启动**

```bash
docker compose pull
```

```bash
docker compose up -d
```

**7. 检查运行状态**

```bash
docker compose ps
```

两个容器都显示 `Up` 即部署成功。浏览器访问 `http://服务器IP:3000`，打不开时检查云服务器安全组是否放行 3000 端口。

### 8.4 日常运维

| 操作  | 命令  |
| --- | --- |
| 更新到最新版本 | `docker compose pull` 然后 `docker compose up -d` |
| 回滚到某个版本 | `.env` 中设置 `IMAGE_TAG=sha-abc1234`，然后 `docker compose up -d` |
| 恢复最新版本 | `.env` 中删除 `IMAGE_TAG` 这一行，然后 `docker compose pull` 和 `docker compose up -d` |
| 查看日志 | `docker compose logs -f backend` |
| 停止服务 | `docker compose down` |

> **不要执行 `docker compose down -v`**，`-v` 会删除数据卷，数据库和上传的文件会全部丢失。

可用的版本标签在镜像的 Package 页面查看。

### 8.5 使用面板部署（Arcane、Portainer、1Panel 等）

- **面板不读取命令行的 `docker login`。** 要在面板的镜像仓库设置（如 Arcane 的 Container Registries）中添加 `ghcr.io`，用户名填自己的用户名，密码填自己的 classic token。
- **先拉取，再部署。** 有用户反馈 Arcane 部署时可能没有用上已配置的仓库凭证，遇到这种情况时，先在面板的 Images 页面手动拉取镜像，再部署项目。
- **面板项目的配置文件同样要检查镜像地址。** 面板项目目录下执行 `docker compose config --images` 即可查看。
- **排查时先脱离面板。** 先在命令行 `docker pull` 成功，再回到面板排查凭证配置。

> 使用 Arcane 时遇到"终端能拉取、面板部署失败"，按第 10 章处理。

### 8.6 国内服务器的网络问题

从中国大陆访问 ghcr.io 经常很慢或超时。注意：`/etc/docker/daemon.json` 中的 `registry-mirrors` **只对 Docker Hub 生效，对 ghcr.io 无效**。

**方法 1：给 Docker 守护进程配置代理**

创建 `/etc/systemd/system/docker.service.d/proxy.conf`：

```ini
[Service]
Environment="HTTPS_PROXY=http://127.0.0.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1"
```

然后重启 Docker：

```bash
systemctl daemon-reload
systemctl restart docker
```

**方法 2：在海外机器拉取后传到国内服务器**

在已登录 ghcr.io 的海外机器上执行：

```bash
docker pull ghcr.io/yinsitanai/knowledget-backend:latest
```

```bash
docker save ghcr.io/yinsitanai/knowledget-backend:latest | gzip | ssh root@国内服务器IP 'gunzip | docker load'
```

frontend 镜像用同样的方法。导入后镜像名不变，国内服务器上直接 `docker compose up -d` 即可。

---

## 9. 问题排查体系

### 9.1 四条排查原则

1. **先读完整报错。** 看清五个要素：哪个命令、哪台机器、哪个系统用户、访问的哪个地址、报错关键词是什么。
2. **沿链路逐层确认，不跳层。** 不要凭感觉反复换 token，先确认地址，再确认资源存在，最后才查权限。
3. **一次只改一个变量。** 地址、token、执行用户、执行工具，每次只改一项，否则无法判断是哪一项起了作用。
4. **先脱离工具手动复现。** 面板部署失败，先在命令行 `docker pull`；工作流失败，先在本地用同一个 token 执行同样的 `git` 或 `curl` 命令。手动能成功，问题就在工具的配置上。

### 9.2 七层排查模型

任何"访问被拒绝"的问题，都可以拆解成一句话：**某个执行环境，以某个身份、用某种凭证，通过网络，对某个目标资源执行操作。** 逐层确认即可快速定位。

| 层   | 要回答的问题 | 检查方法 | 典型问题 |
| --- | --- | --- | --- |
| **L1 目标** | 访问的地址、仓库、标签到底是什么？ | `docker compose config --images`、`git remote -v`、工作流中的镜像名 | 命名空间写错、含大写字母、标签拼错 |
| **L2 存在** | 这个资源真的存在吗？ | 镜像 Package 页面、版本查询 API、仓库页面 | 镜像从未发布、只有 `main` 标签没有 `latest` |
| **L3 网络** | 能连上 github.com 和 ghcr.io 吗？ | `curl -sI https://ghcr.io/v2/` | 超时、TLS 握手失败 |
| **L4 身份** | 当前实际是以谁的身份访问？ | API 查询 `login`、`~/.docker/config.json`、`gh auth status` | 以为登录的是 A，实际是 B |
| **L5 凭证** | token 的类型、权限、有效期、SSO 是否正确？ | 查看 `x-oauth-scopes` 响应头、token 前缀 | fine-grained token、缺 `read:packages`、已过期 |
| **L6 授权** | 这个身份对这个资源有权限吗？ | 仓库 Collaborators、Package settings、`git ls-remote` | 不是协作者、镜像没有授权给该账号 |
| **L7 执行环境** | 实际执行操作的工具用的是这套凭证吗？ | 执行用户（root / sudo）、面板凭证、Secret 名称、checkout 残留凭证 | 命令行成功、面板失败 |

### 9.3 镜像拉取失败的快速定位

```mermaid
flowchart TD
  A["docker pull 失败"] --> B{"地址是否正确?命名空间 / 全小写 / 标签"}
  B -- 否 --> B1["修正地址"]
  B -- 是 --> C{"Package 页面能看到镜像和该标签?"}
  C -- 否 --> C1["检查 Actions 是否发布成功"]
  C -- 是 --> D{"curl -sI https://ghcr.io/v2/返回 401?"}
  D -- 否 --> D1["网络问题:配置代理或 save/load"]
  D -- 是 --> E{"token 是 classic且含 read:packages?"}
  E -- 否 --> E1["重新生成 classic token"]
  E -- 是 --> F{"该账号对镜像有权限?"}
  F -- 否 --> F1["加为仓库协作者或在 Manage access 授权"]
  F -- 是 --> G{"命令行成功而面板或脚本失败?"}
  G -- 是 --> G1["在工具中配置凭证检查执行用户"]
  G -- 否 --> H["docker logout 后重新 login 再试"]
```

### 9.4 标准排查命令集

**准备：把要检查的 token 放进临时变量**

```bash
read -s T
```

**L1 目标：确认实际访问的地址**

```bash
docker compose config --images
```

```bash
git remote -v
```

**L2 存在：确认镜像和标签存在**（这组命令需要 `read:packages` 权限，同时也在检验 L5 和 L6）

组织名下的镜像列表：

```bash
curl -s -H "Authorization: Bearer $T" "https://api.github.com/orgs/YinsitanAI/packages?package_type=container" | grep '"name"'
```

个人名下的镜像列表：

```bash
curl -s -H "Authorization: Bearer $T" "https://api.github.com/users/xianyupro/packages?package_type=container" | grep '"name"'
```

镜像的标签（个人镜像把 `orgs/YinsitanAI` 换成 `users/xianyupro`）：

```bash
curl -s -H "Authorization: Bearer $T" "https://api.github.com/orgs/YinsitanAI/packages/container/knowledget-backend/versions" | grep -A3 '"tags"'
```

**L3 网络：确认能连上**

```bash
curl -sI https://ghcr.io/v2/ | head -1
```

返回 `HTTP/2 401` 表示网络正常（401 是因为没带凭证，属于预期结果）；超时或无输出表示网络不通。

**L4 身份：确认 token 属于谁、本机登录的是谁**

```bash
curl -s -H "Authorization: Bearer $T" https://api.github.com/user | grep '"login"'
```

```bash
cat ~/.docker/config.json
```

输出的 `auths` 中应该有 `ghcr.io`。另外用 `whoami` 确认当前系统用户。

**L5 凭证：确认 token 类型和权限**

```bash
curl -sI -H "Authorization: Bearer $T" https://api.github.com/user | grep -i '^x-oauth-scopes'
```

| 结果  | 含义  |
| --- | --- |
| 输出 `x-oauth-scopes: read:packages, ...` | classic token，后面列出的就是它的权限 |
| 没有任何输出 | fine-grained token，**不能用于 ghcr.io** |

**L6 授权：直接测试能否访问**

代码仓库：

```bash
git ls-remote "https://x-access-token:$T@github.com/xianyupro/knowledgeT.git" | head -3
```

镜像（先用这个 token 执行 `docker login ghcr.io -u <token 所属用户名>`）：

```bash
docker pull ghcr.io/yinsitanai/knowledget-backend:latest
```

**L7 执行环境：对比命令行与工具**

- `docker` 和 `sudo docker` 使用的是不同的凭证文件；
- 面板要在自己的设置里单独配置凭证；
- Actions 中确认 Secret 名称完全一致（区分大小写）、确认失败步骤使用的是哪个 token（见 9.6 节）。

**清理**

```bash
unset T
```

### 9.5 报错关键词速查表

| 报错关键词 | 出现在哪里 | 所在层 | 常见原因 | 处理方法 |
| --- | --- | --- | --- | --- |
| `unauthorized` | docker pull、面板 | L4 / L5 / L7 | 凭证无效或过期；工具用的是旧凭证 | 重新 `docker login`；更新面板中的凭证 |
| `denied` | docker pull | L1 / L2 / L5 / L6 | 命名空间错误；镜像不存在；fine-grained token；账号无权限 | 按 L1 → L6 逐层检查 |
| `manifest unknown` | docker pull | L1 / L2 | 标签不存在（例如只有 `main` 或 `sha-xxx`） | 查询版本 API，改用存在的标签 |
| `repository name must be lowercase` | compose、构建 | L1  | 镜像名含大写字母 | 地址改为全小写 |
| `Bad credentials` | GitHub API | L5  | token 错误、过期或已撤销 | 重新生成 token |
| `Resource not accessible by personal access token` | GitHub API | L5  | 用 fine-grained token 调用了它不支持的接口（如 Packages） | 改用 classic token |
| `Resource protected by organization SAML enforcement` | API、git | L5  | token 未授权组织 SSO | token 列表中 Configure SSO → Authorize |
| `Repository not found` | git | L1 / L6 | 地址错误，或无权访问（私有仓库对无权限者显示为不存在） | 核对地址；请对方加协作者 |
| `Write access to repository not granted`（403） | git | L6 / L7 | 发出的 token 对该仓库无权限（常见于 fine-grained 跨所有者、checkout 残留凭证） | 分开指定 token；设置 `persist-credentials: false` |
| `The requested installation does not exist` | Actions 推送镜像 | L1 / L6 | 在 A 仓库的工作流中推送到 B 的命名空间 | 使用自动命名的工作流；禁用写死地址的工作流 |
| `without workflow scope` | git push | L5  | 推送内容修改了工作流文件，token 缺 `workflow` 权限 | token 勾选 `workflow` |
| `CONFLICT` / `Automatic merge failed` | 同步工作流 | —   | fork 和上游修改了同一处 | 按 6.7 节手动合并 |
| `required variable ... is missing a value` | docker compose | L1  | `.env` 缺少变量，或不在部署目录下执行 | 检查 `.env` 和当前目录 |
| `i/o timeout`、`TLS handshake timeout` | pull、login | L3  | 网络不通或太慢 | 按 8.6 节处理 |
| `failed to prepare project images for deploy` | 面板（如 Arcane） | L7  | 面板没有用上正确的仓库凭证 | 按第 10 章处理 |

### 9.6 GitHub Actions 排查步骤

**1. 找到失败的具体步骤**

打开失败的运行记录 → 左侧点击失败的 job → 展开显示红叉的步骤，阅读完整报错。

**2. 确认这一步使用的是哪个身份**

| 步骤  | 使用的凭证 |
| --- | --- |
| `actions/checkout` | 默认 `GITHUB_TOKEN`，或 `with.token` 指定的 token |
| `git fetch upstream` | `UPSTREAM_TOKEN`，未设置时用 `SYNC_TOKEN` |
| `git push origin` | `SYNC_TOKEN` |
| `docker/login-action` + `build-push-action` | `GITHUB_TOKEN`，只能推送到当前仓库所有者名下 |

**3. 检查 Secret**

确认 Secret 设置在**当前仓库**、名称完全一致（区分大小写）、token 没有过期。

**4. 检查仓库状态**

- fork 仓库的 Actions 是否已启用；
- 失败的是不是上游带过来的工作流（报错中出现别人的命名空间就是这种情况）；
- 定时任务只在 main 分支上运行。

**5. 检查触发链**

同步成功了但没有触发构建，说明推送用的是 `GITHUB_TOKEN`，要改用个人 token。

**6. 开启调试日志**

在运行记录页面点击 **Re-run jobs → Enable debug logging**，重新运行后日志会更详细。

**7. 在本地复现**

用同一个 token 在本地执行失败步骤中的 `git` 或 `curl` 命令（参考 9.4 节）。

### 9.7 部署排查步骤

按顺序执行，每一步通过后再进入下一步：

1. 在部署目录执行 `docker compose config --images`，确认地址正确（L1）。
2. 在浏览器中确认镜像和标签存在（L2）。
3. 执行 `curl -sI https://ghcr.io/v2/`，确认网络正常（L3）。
4. 以部署用户执行 `docker login`，然后手动 `docker pull`（L4 – L6）。
5. 手动拉取成功而部署失败，检查部署工具的凭证和执行用户（L7）。使用 Arcane 时见第 10 章。
6. 镜像拉取成功但容器启动失败，执行 `docker compose logs <服务名>`，检查 `.env`、端口占用（`ss -lntp | grep 3000`）和数据卷。

### 9.8 本项目实际问题复盘

以下是本项目搭建过程中遇到的真实问题，可以作为排查思路的参考：

| 现象  | 定位层 | 根本原因 | 解决方法 |
| --- | --- | --- | --- |
| 拉取 `ghcr.io/xianyupro/...` 返回 `denied`；Package 页面显示镜像源仓库是 `YinsitanAI/knowledgeT` | L1  | 镜像实际发布在 `yinsitanai` 命名空间下 | 改为正确的地址 |
| `docker login` 显示成功，但拉取任何镜像都返回 `denied`；API 返回 `Resource not accessible by personal access token`；查不到 `x-oauth-scopes` | L5  | 使用的是 fine-grained token | 换成 classic token，勾选 `read:packages` |
| 用 xianyupro 的 token 登录，却去拉取 yinsitanai 的镜像 | L4 / L6 | 登录身份与镜像命名空间的权限不匹配 | 用对该镜像有权限的本人账号登录 |
| fork 中的 Actions 报 `installation does not exist`，推送地址是 `ghcr.io/xianyupro/...:main` | L1 / L6 | 上游工作流把命名空间写死了 | 禁用该工作流，或改用自动命名的工作流 |
| 同步工作流在 `git fetch upstream` 时报 `403 Write access to repository not granted` | L6 / L7 | `SYNC_TOKEN` 没有上游权限，且 checkout 残留的凭证被一并发送 | 把成员账号加为上游协作者，或增加 `UPSTREAM_TOKEN` |
| 面板部署报 `unauthorized`，命令行报 `denied` | L7  | 面板使用自己保存的（旧）凭证，与命令行不同 | 在面板中更新仓库凭证 |
| 终端拉取正常，Arcane 部署报 `unauthorized` | L7  | Arcane 不读取终端的登录信息，其 Container Registries 中保存的凭证无效 | 把终端验证有效的用户名和 token 填入 Arcane（第 10 章） |
| 拉取 `...:latestt` 失败 | L1  | 标签拼写错误 | 核对标签 |

---

## 10. Arcane 面板部署专题

本章专门处理这种情况：**在终端 `docker pull` 一切正常，但在 Arcane 中部署同一个项目时拉取镜像失败。**

### 10.1 典型现象

Arcane 部署时报错：

```
failed to prepare project images for deploy: failed to pull image ghcr.io/<命名空间>/knowledget-backend:latest:
... Error response from daemon: error from registry: unauthorized unauthorized
```

而在同一台服务器的终端中执行：

```bash
docker pull ghcr.io/<命名空间>/knowledget-backend:latest
```

可以正常拉取。

### 10.2 原因分析

**1. 终端成功，说明问题只在第七层（执行环境）**

按 9.2 节的七层排查模型，终端能拉取，意味着前六层——地址、镜像是否存在、网络、身份、token 类型、权限——全部正常。剩下的只可能是：Arcane 执行拉取时用的凭证不对。

**2. Arcane 和终端使用的是两套独立的凭证**

|     | 终端 `docker` 命令 | Arcane |
| --- | --- | --- |
| 凭证来源 | `/root/.docker/config.json`（由 `docker login` 写入） | Arcane 自己的 **Container Registries** 设置 |
| 拉取方式 | docker 命令行读取凭证，再请求 Docker 守护进程拉取 | Arcane 通过 Docker API 请求守护进程拉取，凭证附带在请求中 |
| 互相影响 | 终端 `docker login` **不影响** Arcane | 在 Arcane 中配置凭证**不影响**终端 |

Docker 守护进程本身不保存任何仓库凭证，谁发起拉取请求，谁负责提供凭证。所以终端登录得再正确，Arcane 也用不上。

**3. 从报错关键词判断凭证状态**

| 报错  | 含义  |
| --- | --- |
| `unauthorized` | Arcane **发送了凭证，但凭证无效** |
| `denied` | 没有发送凭证（匿名访问），或凭证有效但没有该镜像的权限 |

Arcane 报的是 `unauthorized`，通常说明 Arcane 中保存的凭证本身有问题。

**4. 常见根本原因**

| 原因  | 说明  |
| --- | --- |
| Arcane 中保存的是旧 token | 例如早期配置的 fine-grained token，或已经过期、被撤销的 token。终端更新了 token，Arcane 里没有同步更新 |
| 用户名和 token 不匹配 | 用户名填了 A 账号，token 却是 B 账号生成的 |
| 存在多条 ghcr.io 记录 | Arcane 可能使用了其中错误的一条 |
| 仓库地址填写不规范 | 应填 `ghcr.io`，不要带路径或多余字符 |
| Arcane 自身的已知问题 | 部分版本在某些部署流程中不附带已配置的凭证，远程环境（Agent 模式）下也出现过拉取时不带凭证的问题 |

Arcane 官方仓库中的相关问题报告（可用于对照自己的情况）：

- [#1818](https://github.com/getarcaneapp/arcane/issues/1818)：Arcane 拉取报用户名或密码错误，同一台机器用 docker 命令拉取却成功
- [#3913](https://github.com/getarcaneapp/arcane/issues/3913)：已配置 ghcr.io 凭证，部署时仍拉取失败，先在 Images 页面手动拉取后部署才能继续
- [#1777](https://github.com/getarcaneapp/arcane/issues/1777)：远程环境下拉取私有镜像失败，日志显示 `hasAuth=false`，本地环境正常

### 10.3 修复步骤

**第 1 步：确认终端测试的是同一个地址**

从 Arcane 报错中复制完整的镜像地址，在终端拉取：

```bash
docker pull ghcr.io/<命名空间>/knowledget-backend:latest
```

```bash
docker pull ghcr.io/<命名空间>/knowledget-frontend:latest
```

两个都成功再继续。地址哪怕差一个字符，排查方向就完全不同。

**第 2 步：取出终端正在使用的用户名和 token**

终端中能用的这套凭证是确定有效的，把它原样填进 Arcane 最可靠。先查看保存的内容：

```bash
grep -A2 '"ghcr.io"' /root/.docker/config.json
```

会看到类似 `"auth": "eGlhbnl1cHJvOmdocF94eHh4..."` 的一行。复制引号内的内容解码：

```bash
echo 'eGlhbnl1cHJvOmdocF94eHh4...' | base64 -d
```

输出格式为 `用户名:token`。确认 token 以 `ghp_` 开头（classic token）。

> - 这一步会在屏幕上显示明文 token，完成后执行 `clear` 清屏。
> - 如果 `config.json` 中没有 `auth` 字段，而是 `credsStore`，说明凭证保存在系统的凭证管理器中，这时直接使用你当初生成的 token 即可。

**第 3 步：更新 Arcane 的仓库凭证**

打开 Arcane 的 **Container Registries** 设置：

1. **删除所有旧的 ghcr.io 记录**，只保留一条，避免 Arcane 选错。
2. **新建一条记录：**
  - 仓库地址：`ghcr.io`
  - 用户名：第 2 步输出中冒号前的部分
  - 密码 / Token：第 2 步输出中冒号后的部分
3. 保存。如果界面提供测试连接的功能，先测试一下。

**第 4 步：先在 Images 页面拉取，再部署**

1. 在 Arcane 的 **Images** 页面手动拉取 backend 和 frontend 两个镜像。
2. 拉取成功后，回到项目页面重新部署。

| 结果  | 说明  | 处理方法 |
| --- | --- | --- |
| Images 页面拉取失败 | Arcane 的凭证仍然不对 | 回到第 2、3 步核对 |
| Images 页面拉取成功，部署失败 | Arcane 部署流程没有附带凭证（#3913 类问题） | 先用 10.5 节的临时方案，并升级 Arcane 到最新版本 |
| 都成功 | 问题解决 | —   |

**第 5 步：仍然失败时，查看 Arcane 日志**

见 10.4 节。

### 10.4 查看 Arcane 日志

**1. 找到 Arcane 容器的名称**

```bash
docker ps --format '{{.Names}}' | grep -i arcane
```

**2. 查看拉取相关的日志**

把命令中的 `arcane` 换成上一步查到的容器名称：

```bash
docker logs arcane 2>&1 | grep -iE "imagepull|hasAuth|ghcr" | tail -20
```

**3. 根据日志判断**

| 日志内容 | 含义  | 处理方法 |
| --- | --- | --- |
| `hasAuth=false` | Arcane 没有附带凭证 | 检查仓库地址是否写成 `ghcr.io`；项目在远程环境（Agent）上时，升级 Arcane |
| `hasAuth=true`，仍报 `unauthorized` | 附带了凭证，但凭证错误 | 回到 10.3 节第 2、3 步，核对用户名和 token |
| 没有相关日志 | 日志级别太低 | 给 Arcane 容器添加环境变量 `LOG_LEVEL=debug`，重启后再部署一次 |

### 10.5 临时方案：从终端部署

急需先让服务运行起来时，可以在 Arcane 的项目目录中用终端部署，这样使用的是终端的凭证：

```bash
cd <Arcane 项目目录>
```

例如本项目为 `/home/Arcane/docker/knowledgeT`。然后执行：

```bash
docker compose pull
```

```bash
docker compose up -d
```

由于是在 Arcane 的项目目录中执行，项目名称一致，Arcane 界面中仍然可以看到和管理这些容器。

> 这只是临时方案。以后在 Arcane 中点击重新部署时，仍会遇到同样的问题，所以 10.3 节的凭证最终必须修好。

### 10.6 预防建议

- **两处凭证保持一致。** 服务器终端和 Arcane 使用同一个 classic token（只勾选 `read:packages`），更换 token 时两处同时更新。
- **只保留一条 ghcr.io 记录。** 不要在 Arcane 中堆积多条同一仓库的凭证。
- **token 到期前提前更换。** token 设置了有效期，到期后终端和 Arcane 会同时失效，提前记录到期日期。
- **改完凭证先验证。** 每次修改 Arcane 凭证后，先在 Images 页面拉取一次，确认成功再部署。
- **升级 Arcane 后回归测试。** 升级后在 Images 页面拉取一次私有镜像，确认凭证功能正常。

---

## 11. 附录

### 11.1 首次搭建验收清单

**代码**

- [ ] 双方都能 clone 主仓库
- [ ] 协作者能推送功能分支并创建 PR
- [ ] main 分支已保护，或团队已约定不直接推送 main
- [ ] `.env` 已加入 `.gitignore`

**Actions**

- [ ] 推送到 main 后"构建并发布镜像"运行成功
- [ ] Packages 页面出现 backend 和 frontend 两个镜像，标签包含 `latest` 和 `sha-xxxxxxx`
- [ ] （模式 B）手动运行"同步上游仓库"成功
- [ ] （模式 B）上游有新提交后，fork 自动同步并触发构建

**镜像权限**

- [ ] backend 和 frontend 都已勾选"继承仓库权限"，或已单独授权
- [ ] 部署者用自己的 classic token 能通过 API 查到这两个镜像

**部署**

- [ ] `docker compose config --images` 显示的地址正确
- [ ] `docker compose pull` 和 `docker compose up -d` 执行成功
- [ ] `docker compose ps` 显示两个容器都是 `Up`
- [ ] 如果使用 Arcane 等面板，已在面板中配置 ghcr.io 凭证，并在 Images 页面拉取验证成功（见第 10 章）

**安全**

- [ ] 没有长期使用他人的 token
- [ ] 所有 token 都设置了有效期，且权限最小
- [ ] 服务器上 `.env` 的权限为 600

### 11.2 常用命令速查

| 场景  | 命令  |
| --- | --- |
| 登录 ghcr.io | `docker login ghcr.io -u <你的用户名>` |
| 退出 ghcr.io | `docker logout ghcr.io` |
| 查看实际镜像地址 | `docker compose config --images` |
| 拉取并更新部署 | `docker compose pull` 然后 `docker compose up -d` |
| 查看容器状态 | `docker compose ps` |
| 查看日志 | `docker compose logs -f backend` |
| 查看 token 所属账号 | `curl -s -H "Authorization: Bearer $T" https://api.github.com/user \\| grep '"login"'` |
| 查看 token 类型和权限 | `curl -sI -H "Authorization: Bearer $T" https://api.github.com/user \\| grep -i '^x-oauth-scopes'` |
| 测试能否读取代码仓库 | `git ls-remote "https://x-access-token:$T@github.com/<所有者>/<仓库>.git"` |
| 测试 ghcr.io 网络 | `curl -sI https://ghcr.io/v2/ \\| head -1` |
| 创建功能分支 | `git switch -c feat/描述` |
| 创建 PR | `gh pr create --fill` |
| 发布正式版本 | `git tag v1.0.0` 然后 `git push origin v1.0.0` |
