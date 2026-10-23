# 鲸屿 · 架构设计

> 版本：v1.0（首版）

本文档回答三个问题：**鲸屿由哪些进程和模块组成、一次请求从浏览器到数据库经过哪些层、以及为什么在若干候选方案里选了现在这一套。** 它写给需要接手部署、排查线上问题或评估改造成本的工程师。数据库字段级设计见 [database.md](./database.md)，认证细节见 [auth.md](./auth.md)，接口契约见 [api.md](./api.md)。

---

## 1. 总览

鲸屿是一个单机部署的前后端分离应用。前端是 React SPA，后端是 NestJS 单体 API，PostgreSQL 存业务数据，Redis 承担会话、限流、短缓存和 BullMQ 队列。生产环境四个容器（web 静态站点、api、postgres、redis）跑在同一台 2C2G 机器上，由 Caddy 做 TLS 与反代。

技术栈（已定，不在本文档重新选型）：

| 层 | 选型 | 版本基线 |
| --- | --- | --- |
| 前端框架 | React | 19 |
| 构建 | Vite | 8 |
| 语言 | TypeScript | 7 |
| 样式 | Tailwind CSS | 4 |
| 组件库 | antd | 6 |
| 路由 | React Router | 7（data router） |
| 数据请求 | TanStack Query | 5 |
| 后端框架 | NestJS | 11 |
| ORM | Prisma | 6 |
| 数据库 | PostgreSQL | 17 |
| 缓存/队列 | Redis + BullMQ | 7 / 5 |
| 包管理 | pnpm workspace | 10 |

---

## 2. 分层图

```
┌─────────────────────────────────────────────────────────────────────┐
│ 浏览器                                                               │
│  React 19 SPA · React Router · TanStack Query · antd 6 · Tailwind 4 │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ HTTPS / JSON（cookie: dsh_sid）
┌───────────────────────────────▼─────────────────────────────────────┐
│ 接入层  Caddy（生产） / Vite dev server proxy（开发）                 │
│  TLS 终止 · 静态资源 · /api/* → api:8892 · gzip/brotli               │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────────┐
│ 应用层  NestJS（apps/api）                                           │
│                                                                     │
│  全局中间件   request-id · pino 日志 · body 上限 · Origin 校验        │
│  全局守卫     SessionGuard → StatusGuard → ActivationGuard →         │
│               TrustGuard → RateLimitGuard                            │
│  全局拦截器   ResponseEnvelope · CacheInterceptor（读缓存）           │
│  全局过滤器   HttpExceptionFilter（统一错误码）                       │
│                                                                     │
│  业务模块     auth │ users │ invites │ posts │ comments │ artifacts  │
│               categories │ tags │ likes │ search │ notifications     │
│               moderation │ reports │ admin │ verify │ health         │
└───────┬───────────────────────────────┬─────────────────────────────┘
        │ Prisma Client                 │ ioredis / BullMQ
┌───────▼──────────────┐        ┌───────▼─────────────────────────────┐
│ 数据层  PostgreSQL 17 │        │ 状态层  Redis 7                      │
│  18 张表 · GIN/trgm   │        │ session · 限流 · 短缓存 · 队列       │
│  事务 · 部分唯一索引   │        │ BullMQ: trust-recalc/notification/   │
└──────────────────────┘        │         view-flush                   │
                                └─────────────────────────────────────┘
```

说明两点：

- **没有独立 worker 进程。** BullMQ 的 worker 在 api 进程内注册，队列并发设为 2。2C2G 下多一个 Node 进程约多 80–120 MB 常驻内存，不值得；任务本身都是低吞吐的（见第 6 节）。未来任务变重时再把 worker 拆成独立容器，队列契约不变。
- **没有独立的搜索服务。** 首版搜索直接走 PostgreSQL 的 `tsvector` + `pg_trgm`，不引入 Elasticsearch/Meilisearch。理由见第 8.4 节。

---

## 3. 模块职责

### 3.1 后端（apps/api）

| 模块 | 职责 | 主要依赖 |
| --- | --- | --- |
| `auth` | OAuth 发起/回调、state 票据、账号解析、激活判定、session 签发/撤销、当前用户注入 | Redis、Prisma、GitHub/Google |
| `users` | 个人主页、资料更新、信任等级重算、禁言/封禁状态 | Prisma、BullMQ |
| `invites` | 邀请码生成、列表、预校验、事务内核销、写 `InviteUse`、激活判定 | Prisma、Redis |
| `posts` | 发帖、编辑、删除、详情、信息流、待解决、置顶、环境条字段 | Prisma、Redis 缓存 |
| `artifacts` | 产物块增删改、排序、复制内容读取 | Prisma |
| `comments` | 评论、楼中楼、采纳答案、评论列表、删除 | Prisma、BullMQ |
| `likes` | 帖子/评论点赞与取消、计数维护 | Prisma |
| `categories` / `tags` | 板块与标签的读取与管理端写操作 | Prisma、Redis 缓存 |
| `search` | 搜索查询构造（FTS + trigram）、结果高亮拼接 | Prisma `$queryRaw` |
| `notifications` | 通知写入（队列消费者）、列表、已读 | Prisma、BullMQ |
| `moderation` | 自动判定、审核队列、通过/驳回/隐藏/删除、敏感词管理、审计 | Prisma、Redis 缓存 |
| `reports` | 举报提交、举报队列、处理结果 | Prisma |
| `admin` | 后台聚合接口：用户管理、板块管理、统计 | 以上模块 |
| `verify` | ed25519 challenge 签发、签名校验、设备绑定 | Redis、Node `crypto` |
| `health` | `/health` 存活与依赖探针（DB/Redis） | Prisma、Redis |
| `common` | 守卫、拦截器、过滤器、DTO 基类、分页、错误码 | — |

模块边界规则：跨模块只通过 Service 接口调用，不直接访问对方的 Repository；禁止在 Controller 里写 Prisma 查询。审核判定只允许 `moderation` 模块调用，发帖/评论模块通过 `ModerationService.evaluate()` 拿判定结果，避免规则散落。

### 3.2 前端（apps/web）

| 目录 | 职责 |
| --- | --- |
| `app/router` | 路由表：`/`、`/c/:slug`、`/p/:id`、`/new`、`/p/:id/edit`、`/u/:handle`、`/login`、`/join/:code`、`/notifications`、`/search`、`/admin/*` |
| `features/posts` | 信息流、详情、编辑器、产物块编辑、环境条 |
| `features/comments` | 评论树、回复、采纳 |
| `features/auth` | 登录页、邀请落地页、session 状态 |
| `features/admin` | 审核队列、举报、用户、板块、敏感词、审计 |
| `lib/api` | fetch 封装：错误码映射、401 跳登录、查询串构造 |
| `lib/query` | TanStack Query 的 queryKey 约定与失效策略 |
| `components/ui` | 与业务无关的展示组件（Markdown、复制按钮、等级徽标） |
| `styles` | Tailwind 入口与设计 token（颜色、间距、暗色） |

前端不做任何权限判断的「假放行」：所有按钮的可见性只是体验优化，最终以服务端 403 为准。

### 3.3 共享包（packages/shared）

只放**跨端且无副作用**的内容：DTO 类型与 Zod schema、枚举常量（板块 slug、产物类型、审核状态、错误码）、URL 构造器、Markdown 渲染的公共配置。不放假服务端逻辑，不 import Node 内置模块。

---

## 4. 请求链路

### 4.1 读链路（信息流）

```
GET /api/v1/posts?sort=hot&category=troubleshooting&cursor=<opaque>
  → Caddy → api:8892
  → request-id 中间件
  → SessionGuard：读 cookie dsh_sid → Redis GET sess:{sid}
                   命中则把 { userId, role } 挂到 req.user
                   （等级不存 session，需要时从 user:profile 缓存读取）
                   未命中按匿名处理（读接口允许匿名）
  → RateLimitGuard：匿名按 IP，登录按 userId
  → CacheInterceptor：查 Redis cache:feed:hot:{hash}
                   命中 → 直接返回，跳过后面的数据库
  → PostsService.list()
       → Prisma：WHERE moderationStatus='approved' AND status='open'
                  ORDER BY ... LIMIT 20
       → 若 sort=hot：$queryRaw 计算热度分（或读物化的热度缓存）
  → 组装 DTO（作者、板块、标签、产物数量、环境条）
  → 写回 Redis TTL 60s（只缓存 approved 且非个性化结果）
  → 返回 { data: { items, nextCursor } }
```

缓存只用于**匿名可见的公共列表**与板块/标签/站点配置。任何带 `viewer` 差异的结果（含 pending、含点赞状态）不缓存，避免越权泄漏。

### 4.2 写链路（发帖）

```
POST /api/v1/posts  { title, bodyMd, categorySlug, tags, kernelVersion, platform, model, artifacts }
  → SessionGuard（必须登录）
  → StatusGuard：Redis GET user:status:{userId}；banned → 401，muted 且写操作 → 403
  → ActivationGuard：写接口校验 activated:{userId}；未激活 → 403 INVITE_REQUIRED
  → TrustGuard：按路由元数据要求最低等级（发帖 Lv0，发邀请 Lv3，后台 Lv4）
  → RateLimitGuard：发帖 10 次/小时/用户
  → ValidationPipe（Zod DTO）：标题长度、正文大小、产物数量与单块大小、环境条格式
  → PostsService.create()
       1. 事务开始
       2. 读 SiteSetting（进程内缓存 300s + Redis）拿审核阈值
       3. ModerationService.evaluate()：按顺序判定，得出 approved/pending 与命中原因
       4. INSERT Post / Artifact[] / PostTag[]
       5. 若 pending：INSERT ModerationAction(action=auto_pending, reason=...)
       6. 事务提交
       7. 事务外：BullMQ 入队 notification（通知被 @ 的人与作者，审核结果另行入队）
       8. 失效 cache:feed:*（按前缀删除或版本号递增）
  → 返回 201 { postId, moderationStatus }
```

写操作不跨队列完成核心逻辑：审核判定、帖子落库都在同一事务内完成，队列只做通知、浏览数落库、等级重算等**可补偿**的旁路工作。

### 4.3 审核动作链路

```
POST /api/v1/admin/moderation/:type/:id/approve
  → SessionGuard → StatusGuard → AdminGuard（ADMIN_HANDLES 或 role）
  → ModerationService.transition(id, 'approve', reason?)
       → 事务内 SELECT ... FOR UPDATE 锁行
       → 校验状态机（pending → approved）
       → UPDATE Post SET moderationStatus='approved'
       → INSERT ModerationAction
       → INSERT AuditLog
       → 事务提交
  → 失效帖子详情缓存、信息流缓存、作者等级相关缓存
  → 返回更新后的状态
```

状态机校验必须在数据库事务内、行锁下完成，避免两个管理员同时操作导致 `approved → approved` 或覆盖驳回。

### 4.4 认证链路

登录与设备签名链路较长，单独在 [auth.md](./auth.md) 展开。这里只给边界：`/api/v1/auth/github`、`/auth/google` 与各自的 callback 公开；`/auth/me`、`/auth/logout`、`/verify/challenge`、`/verify/complete` 要求登录。认证相关接口使用更严格的限流（见第 5 节）。

### 4.5 错误约定

| HTTP | 错误码 | 触发 |
| --- | --- | --- |
| 400 | `VALIDATION_FAILED` / `INVALID_MODERATION_TRANSITION` | DTO 校验失败 / 审核状态机不允许的跳转 |
| 400/409 | `INVITE_INVALID` / `INVITE_EXPIRED` / `INVITE_EXHAUSTED` | 邀请码不存在 / 过期 / 用尽 |
| 401 | `UNAUTHORIZED` / `ACCOUNT_BANNED` | 无 session / 被封禁 |
| 403 | `FORBIDDEN` / `ADMIN_ONLY` / `ACCOUNT_MUTED` / `INVITE_REQUIRED` | 权限不足 / 禁言 / 未激活账号执行写操作 |
| 404 | `POST_NOT_FOUND` / `COMMENT_NOT_FOUND` | 目标不存在或对当前用户不可见 |
| 409 | `ALREADY_LIKED` / `ALREADY_ACCEPTED` | 重复点赞 / 重复采纳同一评论 |
| 429 | `RATE_LIMITED` | 触发限流，`Retry-After` 给出秒数 |
| 500 | `INTERNAL` | 未预期错误，日志含 request-id |

对不可见内容统一用 404 而不是 403，避免泄漏「这里有一篇待审核的帖子」。

---

## 5. Redis 用途

Redis 在鲸屿不是「顺手加个缓存」，它承担四类不可替代的职责：**可撤销会话、跨请求限流、一次性票据、队列**。Key 规范统一为 `<域>:<对象>:<标识>`，全部带 TTL（队列除外）。

| Key | 类型 | TTL | 用途 | 失效方式 |
| --- | --- | --- | --- | --- |
| `sess:{sid}` | string(JSON) | 7 天，滑动续期 | 会话：`{userId, role, createdAt, lastSeenAt}`；cookie 值带 HMAC 签名 | 登出/封禁删除；每次请求 `EXPIRE` 续期 |
| `user:sessions:{userId}` | set | 8 天 | 用户全部 sid，用于「封禁即踢」 | 登出时移除单个；封禁时整集合删除 |
| `oauth:state:{state}` | string(JSON) | 10 分钟 | OAuth state 票据：`{provider, invite?, next, verifier?}` | 回调成功后 `GETDEL`，一次性 |
| `rate:{scope}:{key}:{window}` | counter | 窗口长度 | 限流：scope 为路由组，key 为用户/IP 哈希 | 自然过期 |
| `verify:ch:{sha256}` | string | 300 秒 | ed25519 challenge：值为 userId | `GETDEL`，一次性 |
| `verify:used:{sha256}` | string | 600 秒 | challenge 已成功使用的标记，用于区分「重放」与「过期」 | 自然过期 |
| `user:status:{userId}` | string | 300 秒 | 用户状态缓存（active/muted/banned） | 管理动作后主动删 |
| `activated:{userId}` | string | 已激活永久 / 未激活 60 秒 | 邀请激活判定缓存：`1` 已激活，`0` 未激活 | 兑换成功后写 `1`；管理员视为已激活 |
| `user:profile:{userId}` | string(JSON) | 60 秒 | 等级与展示资料缓存（权限判定用，不存 session） | 等级/资料变更后删 |
| `cache:categories` | string(JSON) | 60 秒 | 板块列表（含计数） | 板块管理写操作后删 |
| `cache:site-settings` | string(JSON) | 300 秒 | 审核阈值、敏感词开关、置顶配置 | 配置更新后删 |
| `cache:feed:{sort}:{filtersHash}` | string(JSON) | 60 秒 | 匿名信息流第一页 | 发帖/审核动作后按前缀清理 |
| `view:{postId}:{ipHash}` | string | 1800 秒 | 浏览去重（30 分钟） | 自然过期 |
| `bull:{queue}:*` | BullMQ | 由队列管理 | 任务与锁 | 队列自身 |

约定：

- 所有 key 前缀集中在 `packages/shared` 的常量里，禁止业务代码手写字符串。
- **Redis 的 `maxmemory-policy` 必须是 `noeviction` 或 `volatile-lru`，绝不能用 `allkeys-lru`**：session 被随机淘汰会让用户无规律掉线，且难以排查。内存不够时先扩内存或缩短缓存 TTL。
- Redis 不做持久化真相：即使 flush 全部，用户会重新登录、缓存重建，数据库数据不受影响。生产开启 `appendonly yes`（everysec）即可，不做 RDB 备份策略。
- 多实例部署时 session 天然共享；若未来要广播缓存失效，再引入 Redis pub/sub，首版单实例不需要。

---

## 6. 异步任务（BullMQ）

| 队列 | 触发 | 任务内容 | 重试 | 并发 |
| --- | --- | --- | --- | --- |
| `trust-recalc` | 每小时 cron（`0 * * * *`）+ 每次采纳/审核通过后单用户入队 | 按 [auth.md](./auth.md) 第 4 节规则重算 `trustLevel` | 3 次，指数退避 | 1 |
| `notification` | 评论、回复、采纳、审核结果、举报处理 | 写 `Notification` 行；首版不发邮件不发推送 | 5 次 | 2 |
| `view-flush` | 详情页每次浏览 | 聚合 `view:{postId}` 计数，每 10 秒批量 `UPDATE Post SET viewCount = viewCount + n` | 3 次 | 1 |
| `cache-purge` | 审核动作、板块变更 | 清理信息流与详情缓存前缀 | 3 次 | 1 |

任务幂等要求：`notification` 用 `(userId, type, targetId)` 在应用层去重（同一事件不重复通知）；`view-flush` 允许少量丢失，不允许重复加太多（每次刷写前 `GETDEL` 计数）。失败任务保留在 failed 集合，由管理员通过 `pnpm --filter api queue:retry` 手动重跑，首版不做自动死信处理。

---

## 7. 部署拓扑

### 7.1 生产（单机 Docker Compose）

```
                       Internet
                          │ 443
                 ┌────────▼─────────┐
                 │ caddy            │  TLS（自动证书）/ 静态资源
                 │ 反代 /api → api  │  gzip · 安全响应头
                 └───┬──────────┬───┘
                     │          │
              /assets│          │/api
                     │          │
              ┌──────▼─────┐ ┌──▼──────────────┐
              │ web        │ │ api             │
              │ nginx:alpine│ │ node:22-alpine  │
              │ 静态 dist  │ │ NestJS + BullMQ │
              └────────────┘ │ :8892（仅内网） │
                             └──┬───────────┬──┘
                                │5432       │6379
                        ┌───────▼─────┐ ┌───▼─────────┐
                        │ postgres:17 │ │ redis:7     │
                        │ volume: pg  │ │ AOF everysec│
                        │ 端口不对外  │ │ 端口不对外  │
                        └─────────────┘ └─────────────┘
```

- 只有 caddy 暴露 80/443；api、postgres、redis 只在 compose 内部网络可达。
- `docker-compose.yml` 的 `prod` profile 已包含 postgres、redis、api（8892）、web、backend 与 **caddy**：caddy 是唯一对外暴露 80/443 的容器，负责 TLS、按域名路由（前台 / 后台两个站点）与 `/api/*` 反代。web 与 backend 只在内部网络可达。
- 开发环境的端口映射是另一套：api `8892`、postgres `5433`（宿主）→ 5432（容器）、web dev server `5173`。**不要把开发端口写进生产 compose。**
- 数据卷：`pgdata`（必须备份）、`redisdata`（可选）、`caddy_data`（证书，必须保留）。
- 启动顺序：postgres 健康检查通过 → api 执行 `prisma migrate deploy`（compose 的 entrypoint 脚本）→ api 启动 → caddy 就绪。

### 7.2 资源预算（2C2G）

| 组件 | 内存上限 | 说明 |
| --- | --- | --- |
| PostgreSQL | 512 MB（`shared_buffers=256MB`, `work_mem=8MB`, `max_connections=50`） | 主要内存占用者 |
| api（Node） | 512 MB（`--max-old-space-size=384`） | 常驻约 150–250 MB |
| Redis | 256 MB（`maxmemory 192mb`） | session + 缓存 + 队列 |
| caddy + web | 128 MB | 静态文件几乎不占内存 |
| 系统余量 | ~600 MB | 留出突发与 page cache |

并发预期：首版日活 500、峰值 QPS < 30，单 api 实例足够。真正的瓶颈会先出现在数据库连接数与慢查询上，而不是 Node 的事件循环。

### 7.3 健康检查与运维

- `GET /health`：进程存活（不查依赖）；`GET /health/ready`：`SELECT 1` + `PING`，供 compose healthcheck。
- 日志：pino JSON 输出到 stdout，由 docker 的 `json-file` 驱动限制 `max-size=20m, max-file=3`；每条日志带 `requestId`、`userId`（脱敏）、`route`、`durationMs`。
- 备份：每日 03:00 cron 执行 `tools/backup.sh`，容器内 `pg_dump -Fc` 输出到 `~/backups/jingyu-<时间戳>.dump` 并附 `.sha256`，保留 7 份；用 `tools/restore-db.sh` 恢复到临时库并比对 `Post`/`Comment`/`User` 三表条数。cron、恢复步骤与演练记录见 [ops.md](./ops.md)。
- 升级：`git pull → docker compose build api web → docker compose up -d`；迁移只允许向前兼容（先加列/表，后删旧列），保证回滚镜像仍可运行。
- 监控：首版不引入 Prometheus。用 `docker stats` + 每日一次 `/health/ready` 巡检 + PostgreSQL 慢查询日志（`log_min_duration_statement=500ms`）足够。

---

## 8. 选型理由

### 8.1 为什么后端用 NestJS，而不是 Go

- **类型一致性**：DTO、枚举、错误码放在 `packages/shared`，前后端共用一份 TypeScript 定义，Prisma 生成的类型直接进 Service 签名。Go 方案需要额外维护一套结构体与 JSON tag，接口一改就要两边同步，首版人力不值得。
- **权限矩阵的表达力**：鲸屿的路由几乎都带「登录 + 状态 + 等级 + 限流」四层前置条件。Nest 的 Guard 链 + `SetMetadata` 正好把这件事声明式化（`@MinTrustLevel(3)`、`@RateLimit('post', 5, 600)`），审核/后台模块能复用同一套守卫；Go 需要自己搭中间件与上下文传递。
- **队列与 ORM 生态**：BullMQ 是 Node 原生；Prisma 的迁移与类型生成在 Nest 里是成熟路径。Go 的队列与 ORM 不是不能做，但需要更多胶水。
- **性能不是瓶颈**：单机日活 500、峰值 QPS < 30，Node 的吞吐足够；Go 的优势（低内存、高并发）在这个量级换不来收益，反而增加跨语言协作成本。如果未来要抗 10 倍流量，先做的是加缓存和读写分离，而不是换语言。
- **代价**：Node 常驻内存比 Go 高（约 150–250 MB vs 30–60 MB）；CPU 密集任务（如大文本全文索引）不放在 Node 里做，交给 PostgreSQL。这两点在 2C2G 的内存预算内可接受。

### 8.2 为什么自研，而不是直接用 Discourse

- **核心差异不是「论坛」，是「产物」**：产物块需要独立表、独立编辑器、独立渲染与复制语义、按类型过滤；环境条需要参与搜索与排序。Discourse 的 topic/post 模型以富文本为中心，插件可以加自定义字段，但编辑器、序列化器、搜索索引都要改，升级时持续冲突。
- **审核模型不同**：鲸屿的自动判定顺序、驳回后重提、近 30 天驳回计数、Lv0 全量进队列，是产品级规则；Discourse 的标记/审核队列是另一套语义，硬套会让审核员在两个心智模型之间切换。
- **资源**：Discourse 需要 Ruby、Ember 构建、Sidekiq、Redis、PostgreSQL，单机 2C2G 属于官方不推荐的下限；本站的部署目标是「一台小机器跑全部」。
- **定制前端**：信息流、板块色、产物卡片的视觉是产品差异的一部分，Discourse 的主题系统能改但要跟上游模板耦合；自研 React 前端改起来没有约束。
- **代价**：自研要自己承担反垃圾、权限、通知、搜索、迁移这些「论坛基础设施」。首版的应对是缩小范围（不做邮件/图片/私信/AI 审核），把复杂度压在可控的 18 张表内。

### 8.3 为什么数据库用 PostgreSQL 17，而不是 MySQL

- **jsonb + GIN**：`SiteSetting.value`、`Notification.payload`、`AuditLog.detail` 都需要存半结构化数据并按内容查询；PostgreSQL 的 jsonb 索引与操作符（`@>`、`?`）比 MySQL 的 JSON 函数成熟。
- **部分唯一索引**：每个帖子只能有一个被采纳答案，PostgreSQL 用 `CREATE UNIQUE INDEX ... WHERE "isAccepted"` 一行解决；MySQL 没有部分索引，只能靠触发器或应用层事务兜底。
- **搜索能力**：`to_tsvector('simple', ...)` + GIN、`pg_trgm` 的 `%`/`similarity()` 都在核心内；MySQL 的 FULLTEXT 对中文与组合查询更受限。首版不引入独立搜索引擎，这一点是硬需求。
- **约束与查询表达力**：CHECK、EXCLUDE、CTE、窗口函数、`RETURNING`（邀请码核销靠 `UPDATE ... WHERE usedCount < maxUses RETURNING` 一行原子完成）都更顺手。
- **代价**：PostgreSQL 的 VACUUM 与连接数管理需要一点运维知识（已通过 `max_connections=50` 和 Prisma 连接池上限约束）；团队若更熟 MySQL，学习成本存在，但 SQL 层面差异不大。

### 8.4 为什么搜索不引入 Elasticsearch / Meilisearch

首版内容量预计 < 1 万帖，PostgreSQL 的 GIN 索引在这个量级上预期能维持几十毫秒级的响应（以真实数据压测为准，不预先承诺 SLA）。中文没有内置分词，采用两级策略加一条升级路径：优先 `pg_trgm` 对标题与作者做模糊匹配（对中文按三元组切分，无需词典），正文用 `to_tsvector('simple', ...)` 做 token 匹配并配合 `ILIKE` 兜底；命中率仍不足时再引入 `zhparser` 扩展（升级步骤见 [database.md](./database.md) 4.3）。引入独立搜索引擎意味着多一个进程、一份数据同步逻辑和一套运维负担，在 2C2G 上不划算。相关索引见 [database.md](./database.md) 第 3 节。

### 8.5 其它已定选择

- **Prisma 而非 TypeORM**：迁移可读、类型生成稳定、`$queryRaw` 能兜住搜索与热榜这类复杂 SQL；TypeORM 的装饰器实体在 Nest 里更「传统」，但迁移与类型体验弱于 Prisma。
- **antd 6 + Tailwind 4 并存**：antd 承担后台的表单、表格、弹窗、通知等重组件；Tailwind 承担前台布局、间距、暗色 token。两者通过 CSS 变量对齐色板，避免出现两套视觉语言。不引入第二套 UI 库。
- **TanStack Query 而非手写 fetch 状态**：信息流分页、通知轮询、详情缓存失效都能声明式表达；配合 `queryKey` 约定（`['feed', sort, filters]`、`['post', id]`），审核动作后精准失效。
- **Cookie session 而非 JWT**：需要「封禁即时生效」「登出即撤销」，服务端会话是唯一可靠方案；JWT 撤销要维护黑名单，反而更复杂。跨域需求不存在（同域部署），cookie 的 CSRF 风险用 `SameSite=Lax` + Origin 校验 + 写接口非简单请求控制。
- **Nest 单体而非微服务**：18 张表、一个团队、一台机器，拆分只会增加部署与调试成本。模块边界已经按未来可拆的方式划好，真要拆时按 `auth / content / moderation` 三块走。

---

## 9. 已知风险与应对

| 风险 | 影响 | 应对 |
| --- | --- | --- |
| 中文搜索命中差 | 用户找不到帖子 | 先上 `pg_trgm` + 标题加权；每周看零结果查询词，必要时启用 `zhparser` |
| Lv0 审核队列积压 | 新人体验差、审核员疲劳 | 敏感词精确匹配优先；Lv0 发帖限流；队列按时间正序 + 批量通过 |
| 2C2G 内存吃紧 | OOM 杀进程 | 固定各容器内存上限；Redis `noeviction`；关闭不必要的 Nest 模块（如 Swagger 生产不挂载） |
| 邀请码外泄/倒卖 | 垃圾账号 | 邀请码有次数与有效期；Lv3 才能生成；核销记录进审计日志；异常核销可整批停用 |
| 单机故障 | 全站不可用 | 每日 `pg_dump` 到本机 `~/backups`（`tools/backup.sh`，保留 7 份）；接受小时级恢复；不承诺 SLA。备份尚未异地，整机磁盘故障仍可能丢数据 |
| OAuth provider 故障 | 无法登录 | GitHub 与 Google 双通道；已登录用户不受影响（session 在 Redis） |
| 队列任务堆积 | 通知延迟、浏览数滞后 | worker 与 api 同进程但独立并发；failed 任务可手动重跑；`view-flush` 可丢弃 |
