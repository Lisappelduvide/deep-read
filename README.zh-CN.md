# DeepRead

简体中文 | [English](./README.md)

DeepRead 是一个自托管的、按账号隔离的英文文章阅读工作区。它集成了基于块的文章编辑器、支持分页与高亮/查词的阅读器、生词本，以及可选的 AI 辅助翻译与分析。内容存储在本地 SQLite 中；账号由管理员创建，并可选择启用共享访客账号。

该应用是 React 单页前端，由 Express API 提供服务。

> 本文档描述的行为均已对照当前源码核验。完整生产运维手册参见 [DEPLOYMENT.md](./DEPLOYMENT.md)。

## 目录

- [功能特性](#功能特性)
- [技术栈](#技术栈)
- [项目结构](#项目结构)
- [环境要求](#环境要求)
- [安装](#安装)
- [本地开发](#本地开发)
- [生产部署](#生产部署)
- [配置说明](#配置说明)
- [用户角色与权限](#用户角色与权限)
- [数据存储](#数据存储)
- [常见问题与已知限制](#常见问题与已知限制)

## 功能特性

| 领域 | 代码中的实际实现 |
|---|---|
| 文章库 | 使用块编辑器创建和编辑文章（`text` / `image` 块）。支持粘贴 HTML、拖入或粘贴图片、导入 `.txt` 文件，并记录来源 URL/作者元数据。可浏览、按标题或标签搜索、排序和打标签（每页 10 条）。 |
| 阅读器 | 按每页 1,800 个字符分页，保留图片块，恢复每篇文章上次阅读的页码，调整字号（12–32 px），并切换亮色/暗色主题。 |
| 标注 | 点击单词即可查词；选中文本可保存高亮/下划线及笔记；自动强调生词本中已有的单词。高亮内容提供全局与单篇两种汇总视图。 |
| 生词本 | 保存单词时可记录释义、音标、词性、例句、文章上下文以及间隔重复（SRS）调度字段。支持搜索、排序和按文章分组。提供仅管理员可用的 Eudic 同步接口。 |
| 翻译与分析 | 通过 Baidu AI 文本翻译实现逐页翻译，通过 OpenAI、Gemini 或本地回退方案实现句子级分析。结果缓存于 SQLite，可按文章清除。这些功能需要外部 API 凭据。 |
| 媒体 | 上传图片会转换为 WebP 并生成缩略图，按账号目录存储，且只能通过归属校验后的路由访问。远程图片 URL 会经过校验（仅 http/https、屏蔽私网地址、限制 10 MB）。 |
| 管理 | 概览统计，创建/启用/停用/删除账号，重置密码，导出账号数据，以及切换访客访问开关。 |
| 可移植性 | 用户可将自身数据导出为 JSON。项目提供数据库备份、恢复、导入和迁移脚本。 |

## 技术栈

| 层次 | 组件 |
|---|---|
| 前端 | React 19、React Router 7、Vite 8、lucide-react、axios、oxlint |
| 后端 | Node.js（部署要求 22+）、Express 5、better-sqlite3、sharp、dotenv、cors |
| 数据库 | SQLite（WAL 模式）；Admin 与 Guest 使用独立的数据库文件 |
| 运维 | PM2、Nginx、systemd、Ubuntu；不提供容器镜像 |

## 项目结构

```text
deep-read/
├── src/                        # React 应用
│   ├── api/                    # 前端 API 封装
│   ├── components/             # article/、reader/ 与通用 UI 组件
│   ├── pages/                  # 路由页面（文章库、阅读器、生词本、管理等）
│   ├── routing/access.js       # 基于角色的导航与重定向
│   └── styles/                 # 全局与页面样式
├── backend/
│   ├── server.js               # Express API、认证、媒体路由、静态 SPA 托管
│   ├── db.js                   # SQLite 连接、表结构与迁移辅助逻辑
│   ├── scripts/                # 迁移、管理员初始化、备份、恢复
│   └── .env.example            # 开发环境配置模板
├── deploy/
│   ├── install.sh              # 首次生产安装
│   ├── update.sh               # 生产更新
│   ├── backup-db.sh / restore-db.sh / import-db.sh / migrate.sh / check.sh
│   ├── migrations/             # 账号归属相关的 SQL 迁移
│   ├── ecosystem.config.cjs    # PM2 进程定义
│   └── nginx.conf              # 反向代理配置
├── data/                       # 本地 SQLite 数据库（已 gitignore）
├── uploads/                    # 本地媒体文件（已 gitignore）
├── .env.example                # 面向生产的配置模板
├── package.json                # 前端脚本与共享依赖
├── vite.config.js              # 开发服务器对 /api 和 /uploads 的代理
└── DEPLOYMENT.md               # 详细部署与运维指南
```

## 环境要求

- **Node.js 22 或更高版本。** `deploy/install.sh` 会拒绝更低版本，且 Vite 8 要求 Node 22.13+ 或 24+。
- 部署与迁移脚本需要 **npm**、**Git** 以及 SQLite 命令行工具（`sqlite3`）。
- 生产环境：一台具备 `sudo` 的 Ubuntu 主机，并安装 PM2、Nginx 和 systemd。`deploy/install.sh` 会安装系统包和 PM2，且要求以非 root 的 `ubuntu` 用户运行。
- 翻译、AI 分析、词典查询和 Eudic 同步需要可选的外部账号。缺少相应凭据时，这些功能不可用。

## 安装

```bash
git clone https://github.com/After2thougt/deep-read.git
cd deep-read

npm ci
(cd backend && npm ci)

cp backend/.env.example .env
```

启动前请编辑 `.env`。服务端先加载仓库根目录的 `.env`，再加载 `backend/.env`；已设置的值不会被覆盖。实际被代码读取的变量参见[配置说明](#配置说明)。

准备数据库结构并创建第一位管理员：

```bash
npm --prefix backend run db:migrate
npm --prefix backend run db:bootstrap-administrator
```

`db:migrate` 会对配置的 Admin 数据库应用账号归属迁移。`db:bootstrap-administrator` 是交互式命令，会询问用户名和密码；若已存在管理员，则会拒绝执行。`.env` 中的 `AUTH_USERNAME` / `AUTH_PASSWORD` 组合在启动时创建的是**普通账号**，而不是管理员。

## 本地开发

在两个终端分别运行 API 和 Vite 开发服务器：

```bash
# 终端 1 —— API，监听 http://127.0.0.1:3000
npm --prefix backend run dev
```

```bash
# 终端 2 —— 前端，监听 http://localhost:5173
npm run dev
```

Vite 会把 `/api` 和 `/uploads` 代理到 `http://127.0.0.1:3000`。常用检查命令：

```bash
curl http://127.0.0.1:3000/api/health    # {"status":"ok"}
npm run lint                             # oxlint
npm run build                            # 生产构建输出到 dist/
npm run preview                          # 预览构建后的前端
```

如果不需要自动重启，可用 `npm start` 代替 watch 命令运行 API。API 默认绑定 `127.0.0.1:3000`，并在生产模式下托管 `dist/` 且提供 SPA 回退。

## 生产部署

所有部署脚本都必须以非 root 用户从 `/opt/deepread/app` 目录运行。路径在脚本中定义，无法通过命令行参数修改。

```bash
# 首次安装（Ubuntu 主机）
bash deploy/install.sh

# 从指定分支应用更新（默认 main）
bash deploy/update.sh [branch]

# 运维操作
bash deploy/check.sh
bash deploy/backup-db.sh
bash deploy/restore-db.sh [backup-file]
bash deploy/migrate.sh

# 等效的 npm 脚本
npm run db:backup
npm run db:restore -- /path/to/backup.db
```

`install.sh` 会安装系统包和 PM2，配置 Nginx 与 systemd，准备 `.env`，用 `npm ci` 安装两套依赖树，备份数据库，执行迁移，构建前端，启动名为 `deepread` 的 PM2 进程，并执行健康检查。它**要求 `/data/deepread/app.db` 处已存在 Admin 数据库**；脚本不会创建或替换该数据库。

`update.sh` 会校验环境、要求工作区干净、通过 `git pull --ff-only` 拉取代码、重装依赖、备份数据库、执行迁移、重新构建前端、以更新后的环境重启 PM2，并检查健康端点。脚本没有自动回滚：如果更新在迁移之后失败，需要手动恢复数据库备份。

系统可能安装了 `deepread-deploy.timer` 单元，用于每五分钟检查 GitHub 并调用 `update.sh main`。请在主机上执行 `systemctl status deepread-deploy.timer` 查看状态；若该定时器已禁用或停止，请手动运行 `update.sh`。

| 运行时路径 | 用途 |
|---|---|
| `/opt/deepread/app` | 应用代码目录 |
| `/data/deepread/app.db` | Admin 数据库 |
| `/data/deepread/guest.db` | Guest 数据库 |
| `/data/deepread/backups` | 数据库备份（保留最近 30 份） |
| `/data/deepread/uploads` | 上传的媒体文件 |
| `/data/deepread/logs` | PM2 日志（`deepread-out.log`、`deepread-error.log`） |

Nginx 监听 80 端口，并把 `/api/`、`/uploads/` 以及所有 SPA 路由代理到 `127.0.0.1:3000`。PM2 根据 `deploy/ecosystem.config.cjs` 运行名为 `deepread` 的进程。

## 配置说明

此处不包含任何密钥值。请复制模板并填入自己的配置。

| 变量 | 用途 | 必填 / 默认值 |
|---|---|---|
| `NODE_ENV` | 运行模式；启用生产检查并限制 CORS | `development` 或 `production` |
| `HOST`, `PORT` | API 绑定地址与端口 | `127.0.0.1`、`3000` |
| `DATABASE_PATH` | Admin SQLite 数据库 | 默认 `<repo>/data/app.db`；生产脚本要求 `/data/deepread/app.db` |
| `GUEST_DATABASE_PATH` | Guest SQLite 数据库 | 默认位于 Admin 数据库同级的 `guest.db`；生产脚本要求 `/data/deepread/guest.db` |
| `AUTH_USERNAME`, `AUTH_PASSWORD` | 启动时初始化普通账号 | `NODE_ENV=production` 时必填 |
| `AUTH_SESSION_SECRET` | 为会话 Cookie 签名 | `NODE_ENV=production` 时必填 |
| `GUEST_USERNAME`, `GUEST_PASSWORD` | 共享访客登录凭据 | 启用访客登录时需要 |
| `CORS_ORIGIN` | 允许的跨域来源 | 默认为空；生产环境禁用跨域请求 |
| `UPLOADS_PATH` | 媒体根目录 | 默认 `<repo>/uploads` |
| `BAIDU_TRANSLATE_APPID`, `BAIDU_AI_TRANSLATE_API_KEY` | 翻译服务 | 使用 `/api/translate` 时需要 |
| `ANALYSIS_PROVIDER`, `ANALYSIS_PROMPT_VERSION` | 分析提供方（`openai`、`gemini`、`fallback`）与提示词版本 | 默认提示词版本 `v4` |
| `OPENAI_API_KEY`, `OPENAI_API_BASE`, `OPENAI_API_MODE`, `OPENAI_MODEL` | OpenAI 兼容的分析服务 | 若配置了其他提供方，则为可选 |
| `GOOGLE_GEMINI_API_KEY`, `GOOGLE_GEMINI_MODEL` | Gemini 分析 | 若配置了其他提供方，则为可选 |
| `MERRIAM_WEBSTER_LEARNERS_KEY` | 词典查询 | 可选 |
| `EUDIC_TOKEN` | Eudic 生词同步 | 可选 |
| `IMAGE_DOWNLOAD_PROXY` | 下载远程图片时使用的代理 | 可选 |
| `LEGACY_OWNER` | 归属迁移脚本使用的旧账号所有者 | `deploy/migrate.sh` 中默认为 `carolc` |

模板中的 `ADMIN_KEY`、`TRANSLATE_PROVIDER` 和 `TRANSLATE_API_KEY` 项**当前服务端不会读取**。翻译路由始终使用 Baidu；代码中存在腾讯相关配置，但该路由不会选用它。

会话保存在进程内存中，通过已签名的 `HttpOnly`、`SameSite=Lax` Cookie（`deepread_session`）传递，有效期为 14 天。API 重启后会话即失效，且不会在多个进程间共享。登录尝试按 IP 限流为每 15 分钟 10 次。

## 用户角色与权限

| 角色 | 创建方式 | 能力 |
|---|---|---|
| 管理员（Administrator） | 通过交互式 `db:bootstrap-administrator` 命令创建 | 管理概览，创建/启用/停用/重置/删除/导出账号，切换访客访问，Eudic 同步，以及正常的阅读功能。界面将管理员限制在首页和用户管理页。 |
| 普通用户（User） | 由管理员通过 `/api/admin/users` 创建 | 管理自己的文章、标签、生词、高亮、主题与字号设置，自助修改密码以及导出 JSON。 |
| 访客（Guest） | 访客访问开启时，可从独立的 Guest 数据库使用共享登录 | 访问共享的访客数据空间；无法自助修改密码。关闭访客访问会使所有访客会话失效。 |

归属过滤由后端强制执行：普通用户和管理员的请求按其账号 ID 与数据库限定范围，访客会话则使用 Guest 数据库。前端的角色检查只控制导航；`/api/admin/*` 接口会独立校验管理员权限。密码以加盐 `scrypt` 哈希存储。自助修改密码要求至少 8 个字符；管理员创建或重置的密码要求至少 12 个字符。

## 数据存储

- **数据库：** SQLite，启用 WAL 模式与外键。Admin 数据库保存账号及账号所属内容；Guest 数据库保存共享的访客空间。启动时会检查必需的表结构与归属迁移。
- **数据表：** `auth_users`、`articles`、`article_blocks`、`tags`、`article_tags`、`vocabulary`、`article_analyses`、`article_translation_cache`、`guest_access_settings` 和 `migrations`。归属列与约束由 `deploy/migrations/` 下的两个迁移脚本应用。
- **媒体：** `UPLOADS_PATH`（默认 `<repo>/uploads`）包含 `users/<userId>/` 和 `guest/<userId>/` 目录，存放临时上传文件、每篇文章的 WebP 图片及缩略图。读取媒体需要有效会话、归属路径以及数据库中的引用记录。
- **浏览器状态：** 主题和阅读器字号存储在 `localStorage` 中；草稿与每篇文章的阅读状态按数据库和账号 ID 划分命名空间。
- **备份：** `deploy/backup-db.sh` 通过 WAL 安全快照创建备份并执行完整性检查，保留最近 30 份。`deploy/restore-db.sh` 会停止 PM2、创建恢复前备份、替换数据库、清理 WAL/SHM 文件，并重新执行健康检查。

## 常见问题与已知限制

**启动失败并提示“Admin database schema is not ready”。** 请先备份 Admin 数据库，然后运行 `npm --prefix backend run db:migrate`。在归属迁移和 `role` / `is_active` 列都存在之前，API 会拒绝监听端口。

**`AUTH_USERNAME` 账号无法打开管理页面。** 它是普通账号。请使用 `npm --prefix backend run db:bootstrap-administrator` 创建管理员。

**访客无法登录。** 请设置 `GUEST_USERNAME` 和 `GUEST_PASSWORD`，确认访客账号已存在，并在管理端用户管理页面核实访客访问已开启。

**翻译或分析没有反应。** 这些功能会调用外部服务。请为翻译配置 Baidu 密钥，并为分析配置 OpenAI 或 Gemini 提供方之一。结果按文章和页码缓存。

**更新失败。** 部署脚本不会自动回滚。请使用 `bash deploy/restore-db.sh [backup-file]` 恢复最近的备份，然后重新运行 `bash deploy/update.sh main`。

**当前限制。** 项目没有自助注册或密码找回流程；“Forgot password?” 链接未接到任何路由。本仓库没有 URL 抓取或正文提取接口——来源 URL 仅作为元数据存储。没有文章级文本转语音；只有单词发音使用浏览器语音 API。`package.json` 中未定义测试脚本；可使用 `npm run lint` 和 `npm run build` 进行验证。
