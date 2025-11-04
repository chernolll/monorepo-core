# monorepo-core - 前端 Monorepo 学习实践项目

> 本项目仅用于学习前端 Monorepo 架构，基于 `pnpm Workspace + Git Submodules` 实现，无实际业务价值，核心目标是掌握 Monorepo 项目搭建、子模块管理、依赖统一等核心技能。

## 项目简介
- **架构选型**：pnpm Workspace（依赖管理） + Git Submodules（子模块独立仓库） + TurboRepo（可选，任务编排）
- **目录结构**：
  ```
  monorepo-core/          # 核心管理仓库（仅维护子模块关联和全局配置）
  ├── apps/               # 业务应用目录（每个应用是独立 Git 仓库，通过子模块嵌入）
  │   └── [your-app]/     # 示例：React/Next.js 等业务项目
  ├── packages/           # 共享包目录（独立 Git 仓库，通过子模块嵌入）
  │   ├── @my/axios/      # 示例：axios 二次封装共享包
  │   └── @my/ui/         # 示例：UI 组件共享包
  ├── pnpm-workspace.yaml # pnpm 工作区配置
  ├── package.json        # 根目录依赖（统一外部依赖版本）
  └── .gitmodules         # Git 子模块配置（自动生成）
  ```
- **核心特性**：
  - 子模块独立管理：`apps/` 和 `packages/` 下的内容均为独立 Git 仓库，支持单独开发和版本控制
  - 依赖统一管理：外部依赖（如 axios、react）通过根目录统一版本，避免版本不一致
  - 按需拉取：克隆核心仓库后，可根据需求拉取指定子模块

## 从零搭建流程（学习记录）
### 1. 环境准备
- 安装 pnpm：`npm install -g pnpm`
- 确保 Git 已配置（SSH 密钥/用户名邮箱）：用于子模块仓库关联

### 2. 核心仓库初始化
```bash
# 1. 创建核心仓库目录并初始化
mkdir monorepo-core && cd monorepo-core
git init
git remote add origin [你的核心仓库 Git 地址]

# 2. 初始化 pnpm 工作区
pnpm init -y
# 创建 pnpm-workspace.yaml（关联 apps 和 packages 目录）
cat > pnpm-workspace.yaml << EOF
packages:
  - "apps/*"
  - "packages/*"
EOF

# 3. 提交初始配置
git add pnpm-workspace.yaml package.json README.md
git commit -m "init: 初始化 monorepo 核心配置"
git push -u origin main
```

### 3. 子模块添加（独立仓库嵌入）
#### 3.1 准备独立仓库（前提）
先创建 3 类独立 Git 仓库（如 GitHub/GitLab）：
- 共享包仓库：`monorepo-packages`（对应本地 `packages/`）
- 业务应用仓库：`my-app`（对应本地 `apps/my-app/`）

#### 3.2 添加子模块到核心仓库
```bash
# 1. 添加 packages 子模块（共享包仓库）
git submodule add [你的 packages 仓库地址] packages

# 2. 添加 apps 下的业务应用子模块
git submodule add [你的 my-app 仓库地址] apps/my-app

# 3. 提交子模块配置（核心仓库仅记录子模块引用）
git add .gitmodules packages apps/my-app
git commit -m "feat: 添加 packages 和 my-app 子模块"
git push origin main
```

### 4. 依赖管理配置（统一版本）
#### 4.1 统一外部依赖（如 axios）
```bash
# 根目录安装外部依赖（-w 表示工作区共享）
pnpm add axios@1.7.7 -w
```
根目录 `package.json` 会自动添加：
```json
{
  "dependencies": {
    "axios": "1.7.7" // 所有子项目统一使用此版本
  }
}
```

#### 4.2 子项目引用外部依赖
```bash
# 给 packages/@my/axios 子项目安装共享的 axios（按目录路径过滤）
pnpm add 'axios@^1.7.7' -F packages/axios
```
- 子项目 `package.json` 中直接写版本号（由根目录统一维护，后续升级仅需修改根目录）
- 避免用 `workspace:*`（会混淆内部包和外部依赖，导致解析错误）

#### 4.3 子项目引用内部共享包（如 @my/ui）
```bash
# 给 apps/my-app 安装内部共享包 @my/ui（需确保 packages/ui 的 name 为 @my/ui）
pnpm add '@my/ui@workspace:*' -F apps/my-app
```

### 5. TurboRepo 配置（可选，任务编排）
```bash
# 根目录安装 TurboRepo
pnpm add -D turbo

# 根目录 package.json 添加脚本
cat >> package.json << EOF
"scripts": {
  "build": "turbo run build",
  "lint": "turbo run lint",
  "dev": "turbo run dev",
  "build:my-app": "turbo run build --filter=apps/my-app"
}
EOF

# 创建 turbo.json 配置任务规则
cat > turbo.json << EOF
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "lint": {
      "dependsOn": [],
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
EOF
```

## 项目拉取与使用
### 1. 克隆核心仓库
```bash
git clone [你的核心仓库 Git 地址]
cd monorepo-core
```

### 2. 按需拉取子模块
#### 场景 1：拉取所有子模块（完整项目）
```bash
# 初始化并拉取所有子模块
git submodule update --init --recursive
```

#### 场景 2：只拉取指定子模块（如仅需 packages + my-app）
```bash
# 初始化指定子模块
git submodule init packages apps/my-app
# 拉取子模块代码
git submodule update packages apps/my-app
```

### 3. 安装依赖
```bash
# 安装所有子项目依赖（自动关联根目录统一版本）
pnpm install
```

### 4. 常用命令
```bash
# 启动所有子项目的 dev 服务（Turbo 并行执行）
pnpm dev

# 构建所有子项目（增量执行，仅构建修改过的项目）
pnpm build

# 仅构建 apps/my-app 及其依赖的共享包
pnpm build:my-app

# 给指定子项目添加依赖（如给 apps/my-app 加 react）
pnpm add react -F apps/my-app
```

## 子模块日常维护
### 1. 修改子模块代码（如更新 packages/axios）
```bash
# 进入子模块目录
cd packages/axios

# 切换到主分支（子模块默认是 detached HEAD 状态）
git checkout main

# 开发修改后提交
git add .
git commit -m "feat: 优化 axios 请求拦截器"
git push origin main

# 返回核心仓库，提交子模块引用更新（关键！）
cd ../..
git add packages/axios
git commit -m "update: packages/axios 子模块到最新版本"
git push origin main
```

### 2. 同步他人修改的子模块代码
```bash
# 拉取核心仓库最新的子模块引用
git pull origin main

# 同步指定子模块到最新版本
git submodule update --remote packages/axios

# 或同步所有已初始化的子模块
git submodule update --remote
```

## 常见问题解决
1. **子模块添加报错「The following paths are ignored by .gitignore」**  
   - 原因：根目录 `.gitignore` 忽略了 `packages/` 或 `apps/`
   - 解决：删除 `.gitignore` 中对 `packages/` 和 `apps/` 的忽略规则

2. **pnpm 找不到子项目「No projects matched the filters」**  
   - 原因：子项目目录无 `package.json` 或 `pnpm-workspace.yaml` 未包含该目录
   - 解决：确保子项目有合法 `package.json`，且 `pnpm-workspace.yaml` 配置 `packages: ["apps/*", "packages/*"]`

3. **依赖引用报错「ERR_PNPM_WORKSPACE_PKG_NOT_FOUND」**  
   - 原因：外部依赖误用 `workspace:*` 导致 pnpm 解析为内部包
   - 解决：外部依赖直接写版本号（根目录统一维护），内部包才用 `workspace:*`

## 学习要点总结
1. **Monorepo 核心价值**：统一依赖管理、跨项目代码复用、标准化工程流程
2. **pnpm Workspace**：负责子项目依赖关联，通过 `pnpm-workspace.yaml` 定义项目范围
3. **Git Submodules**：实现子项目独立仓库管理，核心仓库仅维护版本引用
4. **依赖管理原则**：外部依赖根目录统一版本，内部包用 `workspace:*` 引用
5. **TurboRepo**：优化构建效率（并行执行、增量构建、缓存）

## 参考文档
- pnpm Workspace：https://pnpm.io/workspaces
- Git 子模块：https://git-scm.com/book/zh/v2/Git-工具-子模块
- TurboRepo 官方指南：https://turbo.build/repo/docs

---
> 本项目为学习实践产物，如有配置优化或问题，欢迎交流～