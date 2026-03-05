
## 1. 核心目标 (Context)
简述要做什么以及为什么。
- **目标**: 实现 [功能名称]，解决 [痛点]。
- **用户场景**: [用户角色] 在 [什么情况下] 需要 [执行什么操作]。

## 2. 技术栈约束 (Tech Stack)
防止 AI 乱用库或过时的语法。
- **语言**: TypeScript (Strict Mode)
- **框架**: Next.js 15 (App Router), Tailwind CSS
- **组件库**: 必须优先使用 `@/components/ui` (shadcn/ui)
- **数据库**: Prisma + PostgreSQL
- **禁止**: 禁止使用任何 Class-based Components，禁止安装未授权的第三方库。

## 3. 功能需求与验收标准 (User Stories & AC)
是 AI 最核心的工作依据。使用 REQ-XXX 编号方便引用。
可以参考AI-Native-PRD模版

### REQ-001: 用户登录表单
- **描述**: 提供邮箱/密码登录界面。
- **验收标准 (Acceptance Criteria)**:
  - [ ] 邮箱字段必须进行正则校验。
  - [ ] 密码长度不少于 8 位。
  - [ ] 登录失败需显示红色 Toast 提示 "凭据错误"。
  - [ ] 登录成功后跳转至 `/dashboard`。

### REQ-002: 数据列表展示
- **描述**: 从 API 获取并展示数据。
- **验收标准**:
  - [ ] 使用 React Query 进行状态管理。
  - [ ] 必须包含 Skeleton Loading 加载状态。
  - [ ] 支持分页（每页 20 条）。

## 4. 数据模型 (Data Model)
帮助 AI 理解数据库结构。
- **User**: { id, email, password_hash, role }
- **Post**: { id, title, content, authorId, createdAt }

## 5. UI/UX 规范 (Design Tokens)
- **颜色**: Primary: #3b82f6 (Blue-500)
- **布局**: 响应式设计，移动端优先，触控目标最小 44x44px。
- **交互**: 提交按钮在 Loading 状态下必须禁用。

## 6. 任务阶段定义 (Implementation Phases)
引导 AI 按顺序干活，避免一次改动太多导致崩溃。
- **Phase 1**: 定义数据库 Schema 并生成 Migration。
- **Phase 2**: 实现后端 API Route 及单元测试。
- **Phase 3**: 构建前端 UI 组件并连接 API。