# 辰光Agent平台 V2 — 项目说明书 (RBAC 授权增强版)

> **版本**: V2.0  
> **更新日期**: 2026-03-20  
> **基于**: PROJECT_SPEC.md (V1)  
> **本次更新**: 补全 RBAC 授权模块，实现基于权限的接口访问控制

---

## 零、本次更新概览 / Changelog

| 内容 | 说明 |
|------|------|
| **新增** | RBAC 授权中间件与依赖注入 |
| **新增** | 权限码体系与自动生成机制 |
| **新增** | 业务模块接口权限控制 |
| **变更** | `get_current_user` 扩展为 `get_current_user_with_perms` |
| **变更** | 所有业务模块 API 增加权限校验装饰器 |
| **新增** | 权限码枚举与管理 API |
| **新增** | 数据级授权（资源归属检查） |

---

## 一、项目概述

**辰光Agent平台**是一款企业级 AI Agent 管理平台，原 V1 版本已实现用户认证（JWT）、验证码、用户/角色/权限的 CRUD 管理，但**缺少接口授权** — 即所有已登录用户可以访问所有接口，无法对角色分配不同的数据访问权限。

**V2 版本核心目标**：在 V1 认证体系之上，**补全 RBAC 授权功能**，实现"谁能登录"与"登录后能干什么"的分离，使得不同角色（如管理员、运营者、普通用户）对平台功能具有细粒度的访问控制。

### 技术栈回顾

| 层 | 技术 |
|----|------|
| 后端 | Python 3.13 + FastAPI + SQLAlchemy (async) + asyncmy |
| 前端 | React 19 + TypeScript + Vite + Tailwind CSS + shadcn/ui |
| 数据库 | MySQL 8.4 (Docker) |
| 缓存 | Redis (验证码存储) |
| 对象存储 | MinIO (文档/文件上传) |
| 迁移工具 | Alembic |

---

## 二、RBAC 授权架构详解

### 2.1 角色-权限模型回顾

V1 已实现的数据库关联关系：

```
Users ──M:N──> Roles ──M:N──> Permissions
       (user_roles)     (role_permissions)
```

- **`users`** 表：用户基本信息 (`username`, `email`, `is_active`, `is_superuser`)
- **`roles`** 表：角色定义 (`code`, `name`, `description`)
- **`permissions`** 表：权限定义 (`code`, `name`, `description`)
- **`user_roles`** 关联表：用户-角色绑定
- **`role_permissions`** 关联表：角色-权限绑定
- `is_superuser` 字段作为**超级管理员**标记，绕过所有权限检查

### 2.2 V2 新增：权限码体系

#### 2.2.1 权限码命名规范

采用 `resource:action` 格式，便于维护和扩展：

| 权限码 | 说明 | 对应接口 |
|--------|------|----------|
| `agent:read` | 查看 Agent | GET `/api/v1/agents`, GET `/api/v1/agents/{id}` |
| `agent:create` | 创建 Agent | POST `/api/v1/agents` |
| `agent:update` | 更新 Agent | PUT `/api/v1/agents/{id}` |
| `agent:delete` | 删除 Agent | DELETE `/api/v1/agents/{id}` |
| `agent:manage:lifecycle` | 管理 Agent 生命周期（启动/停止/发布/回滚） | POST `/api/v1/agents/{id}/start/stop/publish/rollback` |
| `model:read` | 查看模型 | GET `/api/v1/models` |
| `model:create` | 创建模型 | POST `/api/v1/models` |
| `model:update` | 更新模型 | PUT `/api/v1/models/{id}` |
| `model:delete` | 删除模型 | DELETE `/api/v1/models/{id}` |
| `provider:manage` | 管理供应商 | POST/PUT/DELETE `/api/v1/providers` |
| `prompt:read/create/update/delete` | Prompt 管理 | `/api/v1/prompts` |
| `knowledge:read` | 查看知识库 | GET `/api/v1/knowledge-bases` |
| `knowledge:write` | 写入知识库（创建/上传文档） | POST `/api/v1/knowledge-bases`, 文档上传 |
| `knowledge:manage` | 管理知识库（删除/重索引/重试） | DELETE, reindex, retry |
| `tool:read/create/update/delete/test` | 工具管理 | `/api/v1/tools` |
| `conversation:read` | 查看对话 | GET `/api/v1/conversations` |
| `conversation:annotate` | 标注对话 | POST `/api/v1/conversations/{id}/annotate` |
| `analytics:read` | 查看分析数据 | GET `/api/v1/analytics` |
| `system:user/manage` | 用户管理 | `/api/v1/users` |
| `system:role/manage` | 角色管理 | `/api/v1/roles` |
| `system:permission/manage` | 权限管理 | `/api/v1/permissions` |

#### 2.2.2 默认角色设计

| 角色 Code | 角色名称 | 拥有权限 | 说明 |
|-----------|---------|----------|------|
| `super_admin` | 超级管理员 | **所有权限** | 系统初始化角色，`is_superuser=True` |
| `admin` | 管理员 | 除用户/角色/权限管理外的所有权限 | 日常运营管理 |
| `operator` | 运营者 | Agent/模型/Prompt/知识库/工具的只读+部分写入 | RAG 知识维护、Prompt 调优 |
| `viewer` | 查看者 | Agent/Prompt/知识库/分析的只读 | 业务数据查看 |
| `auditor` | 审计员 | 对话查看、审计日志查看 | 合规审查 |

### 2.3 授权中间件与依赖注入

#### 2.3.1 新的依赖函数

在 `src/core/deps.py` 中补充：

```python
from src.modules.user.model import User
from src.core.exceptions import BizException

async def get_current_user_with_perms(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    """在 get_current_user 基础上，预加载 roles + permissions 关联"""
    user = await get_current_user(token, db)
    # lazy="selectin" 已配置，user.roles 和 role.permissions 会自动加载
    return user

def check_permission(required_perm: str):
    """
    权限检查装饰器/依赖
    用法：current_user: User = Depends(check_permission("agent:create"))
    """
    async def permission_dependency(
        current_user: User = Depends(get_current_user_with_perms),
    ) -> User:
        # 超级管理员直接放行
        if current_user.is_superuser:
            return current_user
        # 收集用户所有权限码
        user_perm_codes = set()
        for role in current_user.roles:
            for perm in role.permissions:
                user_perm_codes.add(perm.code)
        # 检查权限
        if required_perm not in user_perm_codes:
            raise BizException(code=403, message=f"权限不足，需要权限: {required_perm}")
        return current_user
    return permission_dependency
```

#### 2.3.2 装饰器式用法

```python
from src.core.deps import check_permission

@router.post("", response_model=ResponseSchema[AgentRead])
async def create_agent(
    data: AgentCreate,
    svc: AgentService = Depends(get_agent_service),
    _: User = Depends(check_permission("agent:create")),
):
    result = await svc.create_agent(data, current_user=...)
    return ResponseSchema(data=result)
```

### 2.4 数据级授权

部分接口需要**资源归属检查**，例如用户只能编辑自己创建的 Agent，不能编辑他人创建的 Agent（除非拥有管理员权限）：

```python
# 在 Service 层实现
async def update_agent(self, agent_id: int, data: AgentUpdate, current_user: User) -> AgentRead:
    agent = await self.repo.get_by_id(agent_id)
    if not agent:
        raise BizException(code=45001, message="Agent 不存在")
    # 数据级授权：非管理员只能修改自己的 Agent
    if not current_user.is_superuser and agent.created_by != current_user.username:
        raise BizException(code=403, message="无权修改其他用户的 Agent")
    ...
```

### 2.5 权限自动加载优化

为避免每次请求都查询数据库，采用以下优化：

1. **JWT token 中嵌入用户角色列表**，减少一次数据库查询
2. **Redis 缓存用户-权限映射**：`user_perms:{user_id}` → 权限码集合，TTL 30 分钟
3. **角色/权限变更时清除缓存**：调用 `user_roles` 或 `role_permissions` 更新接口后，清除受影响用户的缓存

---

## 三、后端架构详解（补充）

### 3.1 新增文件

| 文件 | 用途 |
|------|------|
| `src/modules/permission/model.py` | 补充 `Permission.code` 枚举定义 |
| `src/modules/permission/router.py` | 补充 `/api/v1/permissions/defaults` 初始化默认权限 |
| `src/core/authz.py` | 新增：RBAC 授权工具类（`PermissionChecker`, `require_perms`） |
| `src/middlewares/authz.py` | 新增：权限检查中间件（可选，用于日志审计） |

### 3.2 需修改的文件

| 文件 | 修改内容 |
|------|---------|
| `src/core/deps.py` | 补充 `get_current_user_with_perms`, `check_permission` |
| `src/modules/*/api.py` | 所有业务模块 API 增加 `Depends(check_permission(...))` |
| `src/modules/agent/service.py` | 补充 `current_user` 参数，实现数据级授权 |
| `src/modules/*/service.py` | 同上，所有业务模块 Service 接受 `current_user` |
| `src/main.py` | 补充权限初始化（数据库迁移后自动填充默认权限） |

### 3.3 API 路由汇总（含权限要求）

| 前缀 | 模块 | 接口 | 权限要求 |
|------|------|------|---------|
| `/api/v1/auth` | 认证 | POST /login | 公开 |
| `/api/v1/captcha` | 验证码 | GET /, POST / | 公开 |
| `/api/v1/users` | 用户 | GET /me | 登录 (认证) |
| `/api/v1/users` | 用户 | CRUD + /search | `system:user/manage` |
| `/api/v1/users/{id}/roles` | 用户 | PUT | `system:user/manage` |
| `/api/v1/roles` | 角色 | CRUD + /search | `system:role/manage` |
| `/api/v1/roles/{id}/permissions` | 角色 | PUT | `system:role/manage` |
| `/api/v1/permissions` | 权限 | CRUD + /search | `system:permission/manage` |
| `/api/v1/permissions/defaults` | 权限 | GET /defaults | `system:permission/manage` |
| `/api/v1/agents` | Agent | CRUD | `agent:read/create/update/delete` |
| `/api/v1/agents/{id}/start/stop` | Agent | 生命周期 | `agent:manage:lifecycle` |
| `/api/v1/agents/{id}/publish/rollback` | Agent | 版本管理 | `agent:manage:lifecycle` |
| `/api/v1/models` | 模型 | CRUD | `model:read/create/update/delete` |
| `/api/v1/providers` | 供应商 | CRUD | `provider:manage` |
| `/api/v1/prompts` | Prompt | CRUD + 版本 | `prompt:read/create/update/delete/publish` |
| `/api/v1/knowledge-bases` | 知识库 | CRUD + 文档 | `knowledge:read/write/manage` |
| `/api/v1/tools` | 工具 | CRUD + 测试 | `tool:read/create/update/delete/test` |
| `/api/v1/conversations` | 对话 | 查看/标注/导出 | `conversation:read/annotate` |
| `/api/v1/analytics` | 分析 | 概览/费用/评估 | `analytics:read` |

---

## 四、前端架构（补充）

> **原则**: 前端不参与授权决策，授权由后端 `403` 响应驱动。前端仅做**菜单权限隐藏**与**操作按钮禁用**。

### 4.1 菜单权限控制

在 `DashboardLayout.tsx` 中，根据用户角色动态隐藏菜单项：

```typescript
const menuPermissions: Record<string, string[]> = {
  '/agents': ['agent:read'],
  '/models': ['model:read'],
  '/prompts': ['prompt:read'],
  '/knowledge': ['knowledge:read'],
  '/tools': ['tool:read'],
  '/conversations': ['conversation:read'],
  '/analytics': ['analytics:read'],
  '/system/users': ['system:user/manage'],
  '/system/roles': ['system:role/manage'],
  '/system/permissions': ['system:permission/manage'],
}

// 在 Layout 中判断
const userPerms = await authService.getUserPermissions()
{userPerms.includes('agent:read') && <MenuItem path="/agents">Agent管理</MenuItem>}
```

### 4.2 操作按钮权限控制

页面内的创建/编辑/删除按钮根据权限判断是否显示：

```typescript
const canCreate = userPerms.includes('agent:create')
{canCreate && <Button Create Agent</Button>}
```

### 4.3 API 客户端 403 处理

在 `apiClient` 中统一处理 403 响应：

```typescript
if (res.status === 403) {
  message.error('权限不足，无法执行此操作')
  // 可选：跳转到无权限提示页
}
```

### 4.4 用户权限接口

新增接口获取当前用户权限列表：

```typescript
// GET /api/v1/users/me/permissions
authService.getUserPermissions(): Promise<string[]>
```

---

## 五、数据库迁移

### 5.1 迁移内容

1. **初始化默认权限** (在 `alembic/versions/` 中添加迁移脚本)
   ```sql
   INSERT INTO permissions (code, name, description) VALUES
     ('agent:read', '查看 Agent', '浏览 Agent 列表和详情'),
     ('agent:create', '创建 Agent', '创建新的 Agent'),
     ('agent:update', '更新 Agent', '编辑 Agent 配置'),
     ('agent:delete', '删除 Agent', '删除 Agent'),
     ('agent:manage:lifecycle', '管理 Agent 生命周期', '启动/停止/发布/回滚'),
     ... (所有权限码)
   ```

2. **初始化默认角色** (在迁移脚本中)
   ```sql
   INSERT INTO roles (code, name, description) VALUES
     ('super_admin', '超级管理员', '系统所有权限'),
     ('admin', '管理员', '管理平台所有功能'),
     ('operator', '运营者', '知识库维护、Prompt 调优'),
     ('viewer', '查看者', '业务数据查看'),
     ('auditor', '审计员', '对话查看、审计日志');

   -- 绑定默认角色权限
   INSERT INTO role_permissions (role_id, permission_id) VALUES
     (1, 1), (1, 2), ... -- super_admin 拥有所有
     (2, 1), (2, 3), ... -- admin 拥有除用户/角色/权限管理外的所有
   ```

3. **为超级管理员初始化**
   ```sql
   -- 将 is_superuser=True 的用户绑定到 super_admin 角色
   INSERT INTO user_roles (user_id, role_id)
   SELECT id, 1 FROM users WHERE is_superuser = TRUE;
   ```

### 5.2 迁移步骤

```bash
alembic revision --autogenerate -m "add RBAC default permissions and roles"
alembic upgrade head
# 运行初始化脚本填充默认数据
python src/scripts/init_rbac.py
```

---

## 六、安全与审计

### 6.1 权限变更审计

- **操作日志**：记录角色权限变更操作（谁、什么时候、变更了什么）
- **审计表**：`audit_logs` 表记录 `action`, `module`, `target_id`, `user_id`, `ip_address`, `detail`
- **触发场景**：角色权限分配、用户角色变更、权限删除

### 6.2 越权访问防范

| 场景 | 防范措施 |
|------|---------|
| 用户尝试访问无权限接口 | 后端返回 403，前端提示 |
| 用户尝试修改他人资源 | Service 层检查 `created_by` |
| 权限码枚举攻击 | 随机化部分接口响应时间 |
| 权限缓存穿透 | Redis 缓存 + 数据库二次验证 |

### 6.3 敏感操作二次确认

- **删除 Agent/知识库/用户**：前端弹窗二次确认
- **发布/Publish 操作**：要求至少 `operator` 角色
- **角色权限修改**：要求 `admin` 或以上角色

---

## 七、开发规范补充

### 7.1 后端 — 权限注解规范

```python
# 标准 API 写法
@router.get("", response_model=ResponseSchema[PageResult[AgentRead]])
async def list_agents(
    params: PageParams = Depends(),
    svc: AgentService = Depends(get_agent_service),
    _: User = Depends(check_permission("agent:read")),
):
    ...
```

### 7.2 Service 层 — 接收 current_user

```python
# Service 方法增加 current_user 参数，用于数据级授权
async def create_agent(self, data: AgentCreate, current_user: User) -> AgentRead:
    agent = Agent(
        ...,
        created_by=current_user.username,  # 记录创建者
    )
```

### 7.3 前端 — 权限判断工具

```typescript
// services/auth.ts 补充
export const authService = {
  ...
  getUserPermissions: (): string[] => {
    return tokenStorage.getPermissions() // 从 token 或缓存获取
  },
  hasPermission: (perm: string): boolean => {
    return authService.getUserPermissions().includes(perm)
  },
  hasAnyPermission: (perms: string[]): boolean => {
    return perms.some(p => authService.getUserPermissions().includes(p))
  },
}
```

---

## 八、测试指南（补充）

### 8.1 后端权限测试

```python
# test/test_rbac.py
async def test_permission_forbidden(client, admin_token):
    """普通用户无法访问 admin 接口"""
    response = await client.get("/api/v1/users/search", 
                                headers={"Authorization": f"Bearer {user_token}"})
    assert response.status_code == 200  # 业务状态码
    assert response.json()["code"] == 403

async def test_data_level_auth(client, user_token):
    """用户无法修改他人 Agent"""
    response = await client.put("/api/v1/agents/2", 
                                headers={"Authorization": f"Bearer {user_token}"},
                                json={...})
    assert response.json()["code"] == 403
```

### 8.2 Mock 权限测试

在前端 Mock 模式下，`authService.getUserPermissions()` 返回固定权限列表，便于 UI 测试。

---

## 九、数据模型速查（更新）

### 9.1 新增字段

| 表名 | 字段 | 类型 | 说明 |
|------|------|------|------|
| `users` | `is_superuser` | bool | 是否超级管理员（已有） |
| `permissions` | `code` | varchar(100) | 权限码（已有） |

### 9.2 关系回顾

```
Users ──M:N──> Roles ──M:N──> Permissions
       (user_roles)     (role_permissions)
```

- `User.roles` → `List[Role]` (lazy="selectin")
- `Role.permissions` → `List[Permission]` (lazy="selectin")
- `Permission` → 无反向关系（单向）

---

## 十、环境搭建（无变更）

参考 V1 文档，命令相同：

```bash
# 后端
uvicorn src.main:app --port 8080 --reload

# 前端
cd app && npm run dev
```

---

## 十一、设计文档索引

| 文件 | 内容 |
|------|------|
| `docs/PROJECT_SPEC.md` | V1 项目说明书 (原版) |
| `docs/PROJECT_SPEC_V2.md` | V2 项目说明书 (RBAC 授权增强) ← 本文档 |
| `app/docs/00-架构总览.md` | 前端架构设计 |
| `app/docs/09-系统管理.md` | 系统管理页面设计 |
| `app/docs/10-API接口规范.md` | API 接口规范 |

---

## 十二、待实现功能清单 (RBAC 授权)

### 12.1 已就绪

- ✅ 用户认证 (JWT + Captcha)
- ✅ 用户/角色/权限 CRUD
- ✅ 用户-角色分配
- ✅ 角色-权限分配

### 12.2 需实现 (V2 新增)

- ❌ **授权中间件**: `check_permission` 依赖
- ❌ **权限码体系**: 默认权限码定义与初始化
- ❌ **默认角色**: 5 个默认角色创建
- ❌ **接口授权**: 所有业务模块 API 增加权限校验
- ❌ **数据级授权**: `created_by` 归属检查
- ❌ **权限查询接口**: GET `/users/me/permissions`
- ❌ **前端菜单控制**: 根据权限动态显示菜单
- ❌ **审计日志**: 记录权限变更操作
- ❌ **权限缓存**: Redis 用户-权限映射缓存
