# 项目启动注意事项

## 1. 环境准备

- **Python 3.14+**: 项目使用 Python 3.14.7 运行。确保已安装并添加到系统 PATH。
- **Node.js v22+**: 前端使用 Node.js v24.20.0。确保已安装并可通过 `node` 和 `npm` 命令访问。
- **Git**: 项目已配置 Git 仓库，所有更改需提交时使用 Git 命令。

## 2. 虚拟环境与依赖

- 项目根目录下有 `.venv` 虚拟环境，基于当前机器的 Python 3.14.7 创建。
- **关键依赖问题修复**（已在本次启动过程中完成）：
  - `requirements.txt` 原始编码为 UTF-16，需转换为 UTF-8 后才能被 pip 正确读取。
  - `pydantic-core` 缺失 C 扩展 `_pydantic_core`，需重新安装或升级至兼容版本（`2.41.5`）。
  - `cffi` 版本不兼容导致 `argon2` 导入失败，需强制重新安装并重新编译（`pip install --force-reinstall --no-binary :all: cffi`）。
  - `pillow` 缺失 C 扩展 `_imaging`，重新安装 pillow 即可解决。
  - `captcha` 包需单独安装。

## 3. 数据库初始化

- 运行 `bootstrap_db.py` 初始化 SQLite 数据库（`chenguang.db`）。
- 该脚本会自动建表、灌入演示数据并标记 Alembic 迁移版本为最新。
- 数据库使用 SQLite，本地免 Docker 开发，无需额外数据库服务。

## 4. 后端启动

- 使用 `.venv\Scripts\python.exe -m uvicorn src.main:app --host 127.0.0.1 --port 8080` 启动。
- 后端健康检查地址：`http://127.0.0.1:8080/health`
- API 文档：`http://localhost:8080/docs`
- 默认登录账号：`admin / 123456`
- 后端依赖模块包括：FastAPI、SQLAlchemy async、fakeredis（模拟 Redis）、MinIO 客户端。

## 5. 前端启动

- 进入 `app/` 目录，运行 `npm.cmd run dev` 启动 Vite 开发服务器。
- 前端访问地址：`http://localhost:5173`
- 前端使用 Vue/React 19 框架，TailwindCSS 进行样式化。

## 6. 已知问题与排查

| 问题 | 解决方案 |
|------|----------|
| `requirements.txt` UTF-16 编码导致 pip 失败 | 转换为 UTF-8：`iconv -f UTF-16 -t UTF-8 requirements.txt > requirements_utf8.txt` |
| `pydantic-core` 缺失 C 扩展 | 重新安装：`pip install --force-reinstall "pydantic-core==2.41.5"` |
| `cffi` 版本冲突导致 `argon2` 导入失败 | 强制重新编译：`pip install --force-reinstall --no-binary :all: cffi` |
| `Pillow` 缺失 `_imaging` 模块 | 重新安装 pillow：`pip uninstall -y pillow && pip install pillow` |
| `captcha` 模块未找到 | 安装：`pip install captcha` |
| 端口 8080/5173 被占用 | 关闭占用端口的进程，或修改 start.bat 中的端口配置 |

## 7. 项目结构要点

- **后端 (`/src`)**: FastAPI + Python 3.13
  - `core/` - 共享基础设施（配置、DI、异常、日志）
  - `infra/` - 数据库（SQLAlchemy async + asyncmy）、Redis 缓存、MinIO 客户端
  - `modules/` - 功能模块（user、auth、captcha、permission、role 等）
- **前端 (`/app`)**: React 19 + TypeScript + Vite
  - `src/pages/` - 模块级路由页面
  - `src/services/` - API 层（可通过 `VITE_USE_MOCK` 切换 mock/真实调用）
- **数据库**: MySQL 8.4 通过 Docker Compose 运行（本地开发使用 SQLite 替代）
- **迁移**: 使用 Alembic，13 个迁移文件在 `/alembic/versions/`

## 8. 快速启动流程

```bash
# 1. 确保 Python 环境和虚拟环境就绪
python -m venv .venv
.\.venv\Scripts\activate

# 2. 转换 requirements.txt 编码（如果需要）
iconv -f UTF-16 -t UTF-8 requirements.txt > requirements_utf8.txt

# 3. 安装后端依赖
pip install -r requirements.txt

# 4. 重关键依赖
pip install --force-reinstall "pydantic-core==2.41.5"
pip install --force-reinstall --no-binary :all: cffi
pip uninstall -y pillow && pip install pillow
pip install captcha

# 5. 初始化数据库
python bootstrap_db.py

# 6. 启动后端
.\.venv\Scripts\python.exe -m uvicorn src.main:app --host 127.0.0.1 --port 8080

# 7. 启动前端
cd app
npm.cmd run dev
```

---

**启动时间预估**: 首次完整启动约需 5-8 分钟（包括依赖安装、数据库初始化和服务启动）。

**联系方式**: 如遇问题，请检查控制台输出的错误日志，或查阅 `docs/` 目录下的各模块说明文档。