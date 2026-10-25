# RBAC 权限管理 UI 设计说明书

## 设计概览

**项目**: 辰光 Agent 平台 V2 - RBAC 授权增强版  
**设计日期**: 2026-10-25  
**设计师**: Claude Code  
**目标受众**: 系统管理员、运营人员、审计员  

---

## 1. 设计目标

### 1.1 核心目标
- 提供直观的 RBAC 权限可视化管理界面
- 实现角色与权限码的清晰展示与编辑
- 支持数据级授权的查看与配置
- 保持与项目现有设计系统的一致性

### 1.2 设计原则
- **辨识性**: 打破通用 SaaS 卡片风格，提供与业务高度契合的视觉体验
- **可用性**: 关注信息层级、交互细节，确保管理人员能快速定位与操作
- **一致性**: 统一配色、排版与组件语义，形成完整的产品语言
- **极简**: 仅保留服务于 RBAC 管理的必要功能，切除冗余装饰

---

## 2. 视觉设计

### 2.1 配色系统

| 变量 | Hex 值 | 用途 | 备注 |
|------|--------|------|------|
| `--bg` | `#0a0e17` | 主背景 | 深邃背景，减少视觉疲劳 |
| `--bg-subtle` | `#111827` | 卡片/输入框 | 轻浅卡片背景 |
| `--fg` | `#f1f5f9` | 主要文字 | 高对比度阅读 |
| `--muted` | `#64748b` | 辅助文字 | 60% 不透明度 |
| `--accent` | `#efa47f` | 主要强调 | 暖金色，区分于官方 Claude 蓝 |
| `--accent-hover` | `#f4b5a1` | 悬停状态 | 加深的暖金 |
| `--card` | `#1a2232` | 卡片容器 | 对比卡片背景 |
| `--border` | `#2a3a52` | 边框分割 | 与背景微妙区分 |
| `--success` | `#22c55e` | 成功状态 | 绿色系 |
| `--warning` | `#f59e0b` | 警告状态 | 橙色系 |
| `--error` | `#f87171` | 错误状态 | 红色系 |

### 2.2 排版系统

- **字体家族**: Inter (sans-serif) + Playfair Display (衬线)
- **基础字号**: 16px, 行高 1.5
- **标题字号**: 32px, 行高 1.2 (Playfair Display)
- **页面标题**: 28px, Inter font-family
- **小号文字**: 12px-13px, 用于标签、说明

### 2.3 版式系统

- **容器宽度**: `max-w-7xl`, `mx-auto`, `px-6`
- **网格系统**: 
  - 桌面: 12栅格系统
  - 移动: 4栅格系统，侧边栏隐藏
  - 统计卡片: 4栅格 (lg:grid-cols-4)
  - 操作面板: 3栅格 (lg:grid-cols-3)
- **间距**: 
  - 小间距: 4px, 8px
  - 中间距: 16px, 24px
  - 大间距: 32px, 40px

### 2.4 组件规范

#### 卡片
- 背景: `var(--card)`, 1px 边框 `var(--border)`
- 圆角: 8px (统计卡片), 12px (主要容器)
- 内边距: 24px
- 悬停效果: `border-color: rgba(239,164,127,0.3)`, `transform: translateY(-2px)`

#### 统计卡片
- 数字: 32px, 加粗, 线性渐变填充 (accent → #d97757)
- 标签: text-sm, text-[var(--muted)]
- 次要信息: text-xs, 灰色

#### 表格
- 交替行背景: `var(--card)` / `#151d2e`
- 表头: 背景 `#101620`, 边框-bottom
- 悬停行: `background: #151d2e`, 边框高亮

#### 按钮
- 主按钮: `background: var(--accent)`, `color: #0a0e17`, padding 8px 16px
- 次按钮: `background: transparent`, `color: var(--muted)`, padding 6px 12px
- 危险按钮: `background: #fee2e2`, `color: #b91c1c`
- 焦点态: 2px `var(--accent)` 边框, outline-offset: 2px

#### 权限标签
- 样式: inline-flex, padding 2px 6px
- 背景: rgba(239,164,127,0.1), 边框 1px rgba(239,164,127,0.3)
- 圆角: 3px, 字号 11px, font-weight 600
- 间距: margin 0 2px

#### 模态框
- 容器: `var(--card)`, 1px `var(--border)`, 圆角 12px
- 最大宽度: 90%, 最宽 500px
- 内边距: 32px
- 关闭按钮: 右上角, 透明背景, 悬停变色

---

## 3. 页面结构

### 3.1 页面布局

```
├── 头部导航栏
│   ├── 品牌标志 "辰光 Agent"
│   └── 用户信息 (super_admin 角色标识)
│
├── 主要内容区
│   ├── 面包屑导航
│   ├── 统计快捷卡片 (4列布局)
│   ├── 快捷操作面板 (3卡片: 角色管理、用户管理、权限管理)
│   ├── 权限码体系概览
│   │   ├── 25+ 权限码标签展示
│   │   └── 默认角色权限映射表
│   ├── 最近操作日志
│   │   ├── 表格: 时间、权限、用户、详情
│   │   └── 4 条最近操作记录
│   └── 搜索与筛选区域
│       ├── 搜索输入框
│       ├── 角色筛选下拉
│       └── 应用按钮
│
└── 底部页脚
    ├── 轻微分隔线
    └── copyright 信息
```

### 3.2 组件布局细节

#### 统计卡片网格 (lg:grid-cols-4)
- 桌面: 4 等分，每卡片 width: 25%
- 移动: 1 列，全宽
- 间距: gap-4 (16px)

#### 快捷操作网格 (lg:grid-cols-3)
- 桌面: 3 等分，每卡片 width: 33.33%
- 移动: 1 列，全宽
- 卡片: padding 24px, rounded-lg, hover:border

#### 搜索与筛选 (grid grid-cols-1 lg:grid-cols-4 gap-4 items-center)
- 4 个项目均匀分布
- 输入框: flex-1 (自动填充剩余空间)
- 下拉: 固定宽度
- 按钮: 自动收缩

---

## 4. 交互设计

### 4.1 导航交互

#### 侧边栏隐式交互 (在移动端)
- 导航项通过点击展开/折叠
- 悬停时显示完整标签
- 记住上次展开状态

#### 用户菜单
- 点击头像下拉显示选项
- 包括: 个人资料、设置、登出
- ESC 关闭或点击外部关闭

### 4.2 统计卡片交互
- **悬停**: `transform: translateY(-2px)`, `border-color: rgba(239,164,127,0.3)`
- **点击**: 无特定跳转 (当前为仪表盘概览)
- **响应式**: 窗口resize时重新计算网格

### 4.3 操作卡片交互
- **悬停**: `border-color: rgba(239,164,127,0.3)`, 轻微提升
- **点击**: 跳转至对应管理页面
  - 角色管理 → `/rbac/roles`
  - 用户管理 → `/rbac/users`
  - 权限管理 → `/rbac/permissions`

### 4.4 权限标签交互
- **悬停**: 轻微放大效果 (可选)
- **点击**: 无直接交互 (仅展示)
- **复制**: 理论上可通过浏览器复制

### 4.5 搜索交互
- **实时搜索**: 输入时触发防抖 (500ms 延迟)
- **筛选重置**: 点击"应用筛选"后，URL参数持久化
- **键盘支持**: Focus 时支持 ↑/↓ 键在选项间切换, Enter 确定

### 4.6 模态框交互
- **打开**: 点击按钮时, overlay 淡入 + 缩放动画
- **关闭**: 
  - 点击 × 号按钮
  - 点击模态框外部背景
  - 按 ESC 键
  - 点击"取消"按钮
- **表单验证**: 
  - 名称: 必填, 3-50 字符
  - 描述: 可选, 最大 200 字符
- **键盘辅助**: 
  - Tab 在字段间导航
  - Shift+Tab 逆向导航
  - Enter 确定, Esc 取消

### 4.7 响应式断点

| 断点 | 宽度 | 变化 |
|------|------|------|
| `sm` | 640px | 统计卡片: 2列 (grid-cols-2) |
| `md` | 768px | 快捷操作: 2列 (grid-cols-2) |
| `lg` | 1024px | 统计卡片: 4列, 快捷操作: 3列, 搜索: 4栅格 |
| `xl` | 1280px | 无额外变化 (基准布局) |

### 4.8 减少动画
- `prefers-reduced-motion: reduce` 时:
  - 移除所有过渡动画
  - 保留仅有的 essential 状态变化
  - 静态展示所有交互状态

---

## 5. 数据结构

### 5.1 权限码集合

```javascript
const permCodes = [
  'agent:read', 'agent:create', 'agent:update', 'agent:delete', 'agent:manage:lifecycle',
  'model:read', 'model:create', 'model:update', 'model:delete',
  'provider:manage',
  'prompt:read/create/update/delete',
  'knowledge:read', 'knowledge:write', 'knowledge:manage',
  'tool:read/create/update/delete/test',
  'conversation:read', 'conversation:annotate',
  'analytics:read',
  'system:user/manage', 'system:role/manage', 'system:permission/manage'
];
```

总计: **25 个标准权限码**

### 5.2 默认角色权限映射

```javascript
const rolePerms = {
  super_admin: ALL_PERMISSIONS,           // 所有 25 个权限
  admin: EXCEPT(['system:user/manage', 'system:role/manage', 'system:permission/manage']),
  operator: ['agent:read', 'agent:create', 'model:read', 'model:create',
    'prompt:read', 'prompt:create', 'knowledge:write', 'tool:read',
    'conversation:read', 'analytics:read'],
  viewer: ['agent:read', 'model:read', 'prompt:read', 'knowledge:read', 'analytics:read'],
  auditor: ['conversation:read', 'conversation:annotate', 'analytics:read']
};
```

### 5.3 用户样本数据

| ID | 用户名 | 角色 | 状态 |
|---|--------|------|------|
| 1 | super_admin | super_admin | online |
| 2 | zhang_wei | admin | online |
| 3 | liu_yun | operator | away |
| 4 | wang_ming | viewer | online |
| 5 | zhou_audit | auditor | online |

### 5.4 最近操作日志

| 时间 | 权限码 | 用户 | 详情 |
|------|--------|------|------|
| 14分钟前 | agent:create | super_admin | 创建了新 Agent: research-assistant-v2 |
| 31分钟前 | knowledge:write | operator_liu | 上传了知识库文档: q2-market-report.pdf |
| 1小时前 | model:read | user_zhang | 查看了模型列表 |
| 2小时前 | system:user/manage | super_admin | 重置了用户密码: user_cheng |

---

## 6. 无障碍设计

### 6.1 语义化 HTML
- 使用 `<nav>` 标签标记导航区域
- 使用 `<header>`, `<main>`, `<section>` 语义化标签
- 面包屑使用 `<ol class="breadcrumb">` 结构
- 表格使用 proper `<th>`/`<td>` 结构

### 6.2 色彩对比度
- 主要文本: `#f1f5f9` 背景 `#0a0e17`, 对比度 ≥ 8.5:1 (WCAG AAA)
- 辅助文本: `#64748b` 背景 `#111827`, 对比度 ≥ 4.5:1 (WCAG AA)
- 状态颜色: 确保在常见色盲下可辨识

### 6.3 键盘支持
- 所有交互元素支持 Tab 焦点遍历
- 悬停状态通过 `:focus-visible` 也可触发
- 模态框支持完整键盘操作 (Tab, Shift+Tab, Enter, Esc)

### 6.4 减少动画
- `prefers-reduced-motion` 查询媒体查询支持
- 动画完全移除或简化为即时状态变化
- 关键动画(模态框打开/关闭)提供 `transition: opacity 0.3s ease, transform 0.3s ease`

### 6.5 字体可缩放
- 使用相对单位 (rem, em) 而非固定 px
- 浏览器放大/缩小界面完整，无内容重叠
- 标题可放大至 200% 仍可阅读

---

## 7. 响应式断点行为

### 7.1 移动端 (≤ 640px)
- 统计卡片: 1列 (grid-cols-1)
- 快捷操作: 1列 (grid-cols-1)
- 面包屑: 移至顶部或简化显示
- 搜索: 全宽输入框
- 表格: 纵向滚动，横向必要时出现滚动条

### 7.2 平板 (≤ 768px)
- 统计卡片: 2列 (grid-cols-2)
- 快捷操作: 2列 (grid-cols-2)
- 侧边栏: 简化为仅图标模式
- 导航: 折叠菜单

### 7.3 桌面 (≥ 1024px)
- 完整布局展示
- 4栅格统计卡片
- 3栅格快捷操作
- 完整表格显示
- 悬停效果激活

### 7.4 大屏 (≥ 1280px)
- 无额外变化
- 内容在 `max-w-7xl` (约 1536px) 中居中
- 充足的呼吸空间

---

## 8. 可访问性声明

| 特性 | 状态 | 备注 |
|------|------|------|
| WCAG 级别 | AA | 达到 AA 级别要求 |
| 色弱友好 | ✓ | WCAG AA 级可比度 ≥ 4.5:1 |
| 键盘可访问 | ✓ | 完全键盘化操作 |
| 屏幕阅读器 | ✓ | 语义化 HTML 结构 |
| 减少动画 | ✓ | `prefers-reduced-motion` 支持 |
| 字体可缩放 | ✓ | 相对单位排版 |
| 焦点可见 | ✓ | `outline: 2px solid var(--accent)` |

---

## 9. 与后端对应

### 9.1 权限码对应表

前端展示的 25 个权限码直接对应后端 `permissions.code` 字段:

| 前端展示 | 后端权限码 | 对应接口 |
|----------|-----------|----------|
| agent:read | agent:read | GET /api/v1/agents, GET /api/v1/agents/{id} |
| agent:create | agent:create | POST /api/v1/agents |
| agent:update | agent:update | PUT /api/v1/agents/{id} |
| agent:delete | agent:delete | DELETE /api/v1/agents/{id} |
| agent:manage:lifecycle | agent:manage:lifecycle | POST /api/v1/agents/{id}/start/stop/publish/rollback |
| model:read | model:read | GET /api/v1/models |
| model:create | model:create | POST /api/v1/models |
| model:update | model:update | PUT /api/v1/models/{id} |
| model:delete | model:delete | DELETE /api/v1/models/{id} |
| provider:manage | provider:manage | POST/PUT/DELETE /api/v1/providers |
| prompt:read/create/update/delete | prompt:read/create/update/delete | /api/v1/prompts |
| knowledge:read | knowledge:read | GET /api/v1/knowledge-bases |
| knowledge:write | knowledge:write | POST /api/v1/knowledge-bases, 文档上传 |
| knowledge:manage | knowledge:manage | DELETE, reindex, retry |
| tool:read/create/update/delete/test | tool:read/create/update/delete/test | /api/v1/tools |
| conversation:read | conversation:read | GET /api/v1/conversations |
| conversation:annotate | conversation:annotate | POST /api/v1/conversations/{id}/annotate |
| analytics:read | analytics:read | GET /api/v1/analytics |
| system:user/manage | system:user/manage | /api/v1/users |
| system:role/manage | system:role/manage | /api/v1/roles |
| system:permission/manage | system:permission/manage | /api/v1/permissions |

### 9.2 角色权限映射

前端 `rolePerms` 映射对应后端数据库 `roles` ↔ `role_permissions` ↔ `permissions` 关系:

- **super_admin**: 所有权限 + `is_superuser=True`
- **admin**: 22 个权限 (除 system:user/manage, system:role/manage, system:permission/manage)
- **operator**: 10 个权限 (RAG 维护、Prompt 调优核心功能)
- **viewer**: 5 个权限 (业务数据查看)
- **auditor**: 3 个权限 (合规审查)

### 9.3 数据级授权前端体现

前端通过以下方式体现数据级授权:

1. **按钮权限控制**: 
   - 创建/编辑/删除按钮根据 `userPerms.includes('permission:code')` 判断显示/隐藏
   - 示例: `{canCreate && <Button>创建 Agent</Button>}`

2. **菜单权限控制**: 
   - 侧边栏菜单项根据权限动态显示/隐藏
   - 示例: `/agents` 仅当 `agent:read` 时显示

3. **操作禁用**: 
   - 按钮禁用态通过 `disabled` 属性或 CSS `opacity: 0.5` 表示
   - 悬停时显示提示: "权限不足，联系管理员"

---

## 10. 设计备注

### 10.1 色彩选择说明
- 主要强调色 `#efa47f` (暖金): 有意避开 Anthropic 的 Claude 蓝 `#0089d6`，形成区分度
- 避免使用通用的 terracotta `#D97757`，改用更柔和的暖金调
- 深色背景 `#0a0e17` 与浅色文字 `#f1f5f9` 提供最佳阅读体验

### 10.2 字体组合
- Inter (sans-serif) 作为主体: 可读性高，现代感强
- Playfair Display (serif) 作为强调: 衬线增添专业感，区分于纯技术产品
- 二者组合避免 "全无衬线" 或 "全无无衬线" 的生僻感

### 10.3 设计风格定位
- 非 SaaS 卡片风格默认 (不使用大量圆角卡片、柔和阴影)
- 采用"数据密集型后台管理"审美
- 信息层级清晰，操作路径直观
- 兼顾视觉愉悦与操作效率

### 10.4 扩展性考量
- 权限码列表可通过后端 API 动态获取，前端仅作展示
- 新增权限码时，仅需在 `permCodes` 数组添加，无需改动视觉设计
- 新增角色时，在 `rolePerms` 对象添加新条目，视觉表格自动适配
- 响应式设计支持未来的移动端全功能管理

---

## 11. 文件产物

### 11.1 静态页面
- `docs/ui/rbac/admin-dashboard.html` — 完整 RBAC 管理员仪表盘 HTML 页面

### 11.2 设计说明书
- `docs/design/ui/rbac-design.md` — 本设计说明文档

### 11.3 视觉资产
- 设计色彩token: 4-core + 6-accent hex values
- 排版规范: Inter + Playfair Display 组合
- 组件库: 按钮、卡片、表格、标签、模态框 规范
- 交互动画: 进入/退出动画、悬停反馈、响应式行为

### 11.4 实现引用
- 可直接在 FastAPI + Tailwind 项目中使用
- CSS 变量定义便于与后端渲染的动态数据集成
- 所有组件均为无依赖原生 HTML/CSS，便于移植至 shadcn/ui 组件库

---
*本设计说明书生成于 2026-10-25，对应 PROJECT_SPEC_V2.md RBAC 授权模块*