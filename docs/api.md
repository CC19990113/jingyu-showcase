# 接口契约 · 鲸屿

> 这份文档回答一个问题：**前端和后端之间到底约定了什么。**
> 唯一权威源是 `packages/shared/src/schemas.ts`（入参校验）与 `types.ts`（出参形状）。
> 本文是给人读的说明 + 示例；两者冲突时以 `packages/shared` 为准，并回来改本文。

## 通用约定

- 基础路径：`/api/v1`，开发环境 `http://127.0.0.1:8892/api/v1`
- 认证：httpOnly cookie `dsh_sid`，值形如 `<sid>.<hmac>`。跨域请求必须带 `credentials: 'include'`
- 成功：`{ data: T }`；分页：`{ data: T[], pageInfo: { nextCursor, hasMore } }`
- 失败：`{ code, message, details? }`，`code` 取值见 `ERROR_CODES`
- 分页一律游标，不用 offset。`limit` 默认 20，最大 50
- 时间字段一律 ISO 8601 字符串（`2026-10-21T09:30:00.000Z`）
- 默认**所有接口都需要登录**，标了 `@Public` 的除外（下表「公开」列）
- 写接口有 Redis 限流，超限返回 429 `RATE_LIMITED`

---

## 一、认证

### `GET /auth/github` · 公开
302 到 GitHub 授权页。支持两个可选参数：
- `?next=/posts/xxx` —— 登录后回跳地址（只接受站内相对路径，含 `//` 或 `\` 一律丢弃，防开放重定向）
- `?invite=<code>` —— 邀请码。**主路径**：OAuth 回调时在同一个事务里核销邀请，用户创建即激活

邀请码无效时：用户照建但**不激活**，回跳 `/?invite_error=INVITE_EXPIRED`（或 `INVITE_INVALID` / `INVITE_EXHAUSTED`），由前端提示。

### `GET /auth/github/callback?code=&state=` · 公开
校验 state（一次性，Redis TTL 600s）→ 换 token → 拉资料 → 建/更新用户 → 建会话 → 写 cookie → 302 回 `WEB_ORIGIN + next`。

GitHub 用户名命中 `ADMIN_HANDLES` 的账号，首次登录即 `role=admin`、`trustLevel=4`。

### `GET /auth/google` / `GET /auth/google/callback` · 公开
同上，走 Google OIDC（`openid email profile`）。

### `POST /auth/logout`
销毁会话、清 cookie。返回 `{ data: { ok: true } }`。

### `GET /auth/me`
```json
{ "data": {
  "id": "clx...", "handle": "example", "displayName": "示例用户",
  "avatarUrl": "https://...", "trustLevel": 3, "isVerified": true,
  "bio": "写插件的", "status": "active", "isAdmin": false,
  "createdAt": "2026-10-01T00:00:00.000Z",
  "stats": { "postCount": 12, "commentCount": 40, "acceptedCount": 6, "likeReceivedCount": 210 },
  "devices": [{ "id": "d1", "label": "主力 Mac", "kernelVersion": "0.2.1-rc.2",
                "platform": "darwin-arm64", "fingerprint": "9f2a...", "verifiedAt": "..." }],
  "unreadNotificationCount": 3, "canCreateInvite": true,
  "activated": true
} }
```
未登录返回 401 `UNAUTHORIZED`，前端据此判断登录态。
`activated` 为 `false` 表示账号已创建但**尚未通过邀请激活**：只能读，写接口一律 403 `INVITE_REQUIRED`。

---

## 二、邀请

### `POST /invites/:code/redeem`
**补兑路径**，不是主路径。主路径是 `GET /auth/github?invite=<code>`（回调内同事务核销）。
这个接口用于：已经登录、但账号尚未激活的人（例如此前登录时没带邀请码，或邀请当时无效）。

校验顺序：不存在 `INVITE_INVALID` → 过期 `INVITE_EXPIRED` → 用尽 `INVITE_EXHAUSTED` → 已兑换过 `CONFLICT`。
成功：`{ data: Viewer }`（重建会话 cookie，返回最新的登录用户信息，前端可直接替换登录态）。

### `POST /invites` · 需 Lv3+
```json
{ "maxUses": 3, "expiresInDays": 30, "note": "给群友" }
```
→ `{ data: InviteView }`

### `GET /invites` · 管理员
→ `{ data: InviteView[], pageInfo }`

---

## 三、本地签名（已验证 DSH 用户徽章）

### `POST /verify/challenge`
```json
{ "data": {
  "challenge": "V1StGXR8_Z5jdHi6B-myT",
  "expiresAt": 1761100000000,
  "command": "node tools/dsh-verify.mjs sign V1StGXR8_Z5jdHi6B-myT"
} }
```
challenge 存 Redis，TTL 300 秒。

### `POST /verify/complete`
```json
{
  "challenge": "V1StGXR8_Z5jdHi6B-myT",
  "publicKey": "<base64(SPKI DER)>",
  "signature": "<base64(64 字节)>",
  "kernelVersion": "0.2.1-rc.2",
  "platform": "darwin-arm64",
  "timestamp": 1761100000123,
  "label": "主力 Mac"
}
```
服务端拼 `challenge|kernelVersion|platform|timestamp` 验签。失败 `SIGNATURE_INVALID`；challenge 过期 `CHALLENGE_EXPIRED`；已用过 `CHALLENGE_USED`；时间戳偏差 >600s 拒绝。
成功 → `{ data: VerifiedDevice }`。

### `DELETE /verify/devices/:id`
撤销自己的设备（`revokedAt` 置时间），不影响历史内容上的徽章快照。

### `GET /me/devices`
→ `{ data: VerifiedDevice[] }`，只返回未撤销设备。

> 安全边界：全流程不接触任何 DeepSeek API key，只用本机生成的 ed25519 密钥对。

---

## 四、帖子

### `GET /posts` · 公开
| 参数 | 说明 |
|---|---|
| `category` | 板块 slug |
| `tag` | 标签 slug |
| `author` | 作者 handle |
| `sort` | `latest` / `hot` / `unsolved`，默认 latest |
| `q` | 关键词（标题 + 正文 + 作者，trigram + 全文搜索） |
| `cursor` `limit` | 游标分页 |

→ `{ data: PostSummary[], pageInfo }`

**可见性**：只返回 `moderationStatus=approved`。带 `author=<自己>` 时额外包含自己的 `pending` / `rejected`。

### `GET /posts/:id` · 公开
→ `{ data: PostDetail }`，含 `bodyHtml`、`artifacts`、`viewer` 状态。未登录时 `viewer` 全 false。
`pending` 帖子仅作者与管理员可读，其他人 404。

### `POST /posts`
```json
{
  "categorySlug": "troubleshooting",
  "title": "工具调用进了死循环，日志里全是同一条 validation error",
  "bodyMd": "周三晚上跑一个批量改字段的活……",
  "tags": ["tool-schema", "loop"],
  "kernelVersion": "0.2.1-rc.2",
  "platform": "darwin-arm64",
  "model": "deepseek-flash",
  "artifacts": [
    { "kind": "patch", "name": "debug-tool-schema.patch",
      "content": "const good = { type: 'object', required: ['target'] };",
      "note": "可直接粘贴进插件源码" }
  ]
}
```
校验失败返回 400 + `details`（字段级）。
审核判定后可能返回 `moderationStatus: "pending"` —— 前端要显示「审核中」而不是当成失败。
限流：10 次/小时。

### `PATCH /posts/:id` · 作者或管理员
同 `POST` 字段，全部可选。**被驳回的帖子编辑后自动回到 pending**。

### `DELETE /posts/:id` · 作者或管理员
硬删除 + 审计留痕。

---

## 五、评论

### `GET /posts/:id/comments` · 公开
→ `{ data: CommentView[] }`，顶层评论数组，楼中楼在 `children`（只允许两层）。
只返回 `approved` + 当前用户自己的 `pending`。

### `POST /posts/:id/comments`
```json
{ "bodyMd": "就是这个问题。补充一条：`additionalProperties: false` 也有用。", "parentId": "clx..." }
```
`parentId` 指向同帖内的父评论；若父评论本身已是第二层，服务端拍平到其顶层父级。

### `POST /comments/:id/accept` · 仅帖子作者
帖子未解决时可用。采纳后：`post.isSolved=true`、`acceptedCommentId`、给回答者发通知、触发信任等级重算。
重复采纳 → `ALREADY_ACCEPTED`。

### `DELETE /comments/:id` · 作者或管理员

---

## 五半、GitHub 仓库预览

### `GET /github/repo?url=&withReadme=` · 公开
把 GitHub 仓库地址解析成一张卡片。**不需要登录、不需要额外 OAuth 权限、不使用任何 token** ——
服务端用匿名 GitHub API 拉公开仓库元数据，Redis 缓存 10 分钟。

| 参数 | 说明 |
|---|---|
| `url` | 完整地址 `https://github.com/o/r`、`github.com/o/r` 或 `o/r` 简写都认 |
| `withReadme` | `1` 时额外返回 README 摘要（发帖导入用，卡片不用） |

```json
{ "data": {
  "owner": "CC19990113", "repo": "dsh-plugin-agentflow",
  "fullName": "CC19990113/dsh-plugin-agentflow",
  "htmlUrl": "https://github.com/CC19990113/dsh-plugin-agentflow",
  "description": "Visual agent-orchestration canvas for DeepSeek Harness (dsh)…",
  "language": "JavaScript", "stars": 3, "forks": 0, "license": "MIT",
  "topics": ["dsh-plugin", "deepseek-harness", "agentflow", "…"],
  "pushedAt": "2026-10-21T…", "archived": false,
  "isDshPlugin": true,
  "readmeExcerpt": "…（仅 withReadme=1 时返回）"
} }
```

错误：仓库不存在 → 404；GitHub 匿名额度打满 → 429 `RATE_LIMITED`。
限流：每 IP 30 次 / 10 分钟（缓存命中不计数）。

**前端用法**：帖子正文里「独占一个段落」的仓库链接会自动渲染成卡片
（`components/GitHubRepoCard.tsx`），句子中间的行内链接保持原样。
发帖页「从 GitHub 导入」调 `withReadme=1` 生成草稿。

---

## 六、点赞与举报

### `PUT /posts/:id/like` / `DELETE /posts/:id/like`
### `PUT /comments/:id/like` / `DELETE /comments/:id/like`
返回 `{ data: { liked: true, likeCount: 42 } }`。
给自己点赞 → `CANNOT_LIKE_OWN`；重复点赞 → `ALREADY_LIKED`。限流 60 次/小时。

### `POST /reports`
```json
{ "targetType": "post", "targetId": "clx...", "reason": "spam", "detail": "带推广链接" }
```
`reason` 取值：`spam` / `abuse` / `wrong-place` / `plagiarism` / `other`。

---

## 七、元数据与搜索

### `GET /categories` · 公开 → `{ data: CategoryView[] }`
### `GET /tags?q=` · 公开 → `{ data: TagView[] }`
### `GET /search?q=` · 公开
→ `{ data: { posts: PostSummary[], users: UserBrief[], tags: TagView[] } }`，每组最多 5 条。

### `GET /users/:handle` · 公开 → `{ data: UserProfile }`
### `GET /users/:handle/posts` · 公开 → 该作者的 approved 帖子
### `PATCH /users/me`
```json
{ "displayName": "示例用户", "bio": "写插件的", "handle": "example" }
```
`handle` 只能设置一次，二次设置忽略并写注释说明。

---

## 八、通知

### `GET /notifications?cursor&limit` → `{ data: NotificationView[], pageInfo }`
### `POST /notifications/:id/read` → `{ data: { ok: true } }`
### `POST /notifications/read-all` → `{ data: { updated: 12 } }`

---

## 九、审核（moderator / admin）

### `GET /admin/moderation/queue?target=post|comment&status=pending&cursor&limit`
→ `{ data: ModerationQueueItem[], pageInfo }`
`reasons` 说明为什么进了队列：`["新用户"]`、`["命中敏感词: 内部链接"]`、`["近期被驳回 2 次"]`。

### `POST /admin/moderation/:type/:id/approve`
```json
{ "note": "看着没问题" }
```
### `POST /admin/moderation/:type/:id/reject`
```json
{ "reason": "缺少内核版本，且正文没有可复现步骤" }
```
理由必填（4–500 字），作者会在自己的帖子页看到它。

### `POST /admin/moderation/:type/:id/hide`
```json
{ "reason": "重复发帖" }
```

**状态机（只允许这四条边）**
```
pending → approved | rejected
approved → hidden
rejected → pending   （作者编辑后重提）
```
非法转换 → `INVALID_MODERATION_TRANSITION`。

每次动作同时写：内容状态 + `ModerationAction` + `AuditLog` + 给作者发 `Notification`。

### 敏感词管理
- `GET /admin/banned-words`
- `POST /admin/banned-words` `{ "pattern": "...", "kind": "exact|regex", "enabled": true }`
- `PATCH /admin/banned-words/:id`
- `DELETE /admin/banned-words/:id`

> 不做整表覆盖提交（一次误操作能把词库清空）。

---

## 十、后台其余

### `GET /admin/stats` → `{ data: AdminStats }`
```json
{ "pendingPosts": 3, "pendingComments": 7, "pendingReports": 1,
  "totalUsers": 1284, "totalPosts": 486, "newUsersToday": 12 }
```

### `GET /admin/users?q&status&role&cursor&limit` → `{ data: UserProfile[], pageInfo }`

### `POST /admin/users/:id/role` · 仅 admin
```json
{ "role": "moderator" }
```
`role` 取值 `user` / `moderator` / `admin`。
**不允许把最后一个 admin 降级**（返回 `CONFLICT`），否则站点会失去所有管理员。

### `POST /admin/users/:id/trust-level` · 仅 admin
```json
{ "trustLevel": 4 }
```
手工授予等级（0–4），用于人工兜底的 Lv4 元老。写 AuditLog。

### `POST /admin/users/:id/status`
```json
{ "status": "banned", "reason": "持续发广告" }
```
`status` 取值 `active` / `muted` / `banned`。
**改为 banned/muted 时立即调用 `SessionService.revokeAllByUser()` 踢下线**，不等会话过期。

### `GET /admin/reports?status&cursor&limit` → `{ data: ReportView[], pageInfo }`
### `POST /admin/reports/:id/resolve`
```json
{ "action": "hide", "reason": "确认是广告" }
```
`action`：`hide` / `delete` / `dismiss`。

### `GET /admin/audit-logs?actorId&targetType&action&cursor&limit` → `{ data: AuditLogView[], pageInfo }`

---

## 十一、错误码速查

| code | HTTP | 什么时候 |
|---|---|---|
| `UNAUTHORIZED` | 401 | 未登录 |
| `FORBIDDEN` / `ADMIN_ONLY` | 403 | 权限不够 |
| `ACCOUNT_BANNED` / `ACCOUNT_MUTED` | 403 | 账号被封 / 被禁言 |
| `VALIDATION_FAILED` | 400 | 入参没通过 zod，`details` 里有字段级错误 |
| `RATE_LIMITED` | 429 | 频率超限 |
| `INVITE_*` | 400/409 | 邀请码不存在 / 过期 / 用尽 / 重复 |
| `CHALLENGE_*` / `SIGNATURE_INVALID` | 400 | 签名验证失败 |
| `POST_NOT_FOUND` / `COMMENT_NOT_FOUND` | 404 | 内容不存在或无权查看 |
| `INVALID_MODERATION_TRANSITION` | 400 | 审核状态机不允许的跳转 |
| `INTERNAL` | 500 | 未捕获异常（不返回堆栈） |

---

## 十二、后台认证与账号管理

> 这一节的接口属于管理后台**自己的身份体系**，与前台的 `dsh_sid`（GitHub OAuth）互不认。基础路径仍是 `/api/v1`。

### 两套身份互不认

| | 前台 | 后台 |
|---|---|---|
| 登录方式 | GitHub / Google OAuth | 用户名 + 密码 |
| 会话存储 | Redis | PostgreSQL `AdminSession` 表 |
| Cookie | `dsh_sid`，14 天 | `dsh_admin_sid`，12 小时 |
| 适用接口 | `/auth/*` 与社区接口 | `/admin-auth/*`、`/admin/*` |

- 前台管理员（`GET /auth/me` 里 `isAdmin: true`）**进不了后台**：`/admin/*` 只校验后台会话，不读 `dsh_sid`。
- 后台账号不是社区 `User`，不出现在用户列表里，也不能发帖 / 评论 / 点赞。
- 两个 cookie 在同一 host 下可以共存，但服务端各验各的，互相不能替代。

### 没有注册接口

后台账号只有两个来源：

1. 已有后台账号在「账号管理」里创建（`POST /admin/accounts`）；
2. 首次部署时用根目录的 `pnpm admin:create` 直接在数据库里建（见 README「后台第一个账号」）。

### 角色与权限

`owner` > `admin` > `operator` > `viewer`。后台会话会被映射成前台的等价角色，再复用现有的 `@Roles` / `@AdminOnly`：

| 后台角色 | 等价角色 | 能力 |
|---|---|---|
| `owner` | `admin` | 全部后台能力，含账号管理 |
| `admin` | `admin` | 同上；账号管理接口同样放行 |
| `operator` | `moderator` | 审核、举报等 `@Roles('moderator')` 接口 |
| `viewer` | `user` | 只读；需要 moderator 的接口返回 403 `ADMIN_ONLY` |

账号管理接口（`/admin/accounts*`）要求映射后为 `admin`：**`owner` 与 `admin` 都能调用**，`operator` / `viewer` 返回 403。

> 代码注释写的是「只有 owner 能创建账号、改角色」，但当前实现没有这条判断，`admin` 角色同样能调用账号管理接口；以实际行为为准。

### `POST /admin-auth/login` · 公开

```json
{ "username": "boss", "password": "至少 12 位的密码" }
```

```json
{ "data": {
  "id": "clx...", "username": "boss", "displayName": "老板",
  "role": "owner", "mustChangePassword": false
} }
```

成功时 `Set-Cookie: dsh_admin_sid=<token>`：`HttpOnly`、`SameSite=Lax`、`Path=/`、有效期 12 小时（`NODE_ENV=production` 时带 `Secure`）。
数据库只存 `sha256(token)`，明文 token 只在这一刻出现。

失败：

- `401 UNAUTHORIZED` —— 用户名或密码不对（账号不存在时也会跑一次 scrypt，避免用响应时间探测账号）
- `403 FORBIDDEN` —— 账号 `status != 'active'`
- `429 RATE_LIMITED` —— 同一 IP + 用户名 15 分钟内失败 5 次后，后续失败请求被限流；登录成功会清零计数

用户名会 `trim` 并转小写，登录、失败都写 `AuditLog`。

### `POST /admin-auth/logout`

撤销当前 `dsh_admin_sid` 对应的数据库会话，并清掉 cookie：

```json
{ "data": { "ok": true } }
```

没有后台会话时同样返回成功（属于幂等清理）。

### `GET /admin-auth/me`

返回当前后台身份，形状同登录的 `data`。未登录 / 会话已失效 → `401 UNAUTHORIZED`，前端据此跳登录页。

```json
{ "data": {
  "id": "clx...", "username": "boss", "displayName": "老板",
  "role": "owner", "mustChangePassword": false
} }
```

### `POST /admin-auth/change-password`

```json
{ "oldPassword": "旧密码", "newPassword": "新密码（≥12 位）" }
```

改密成功后**撤销该账号的全部后台会话**并清掉当前 cookie，客户端必须重新登录：

```json
{ "data": { "ok": true } }
```

错误：`400 VALIDATION_FAILED`（旧密码不对 / 新密码不满足强度要求 / 新旧密码相同）；`401 UNAUTHORIZED`（没有后台会话）。

### `GET /admin-auth/sessions`

当前账号仍有效的会话（未撤销、未过期），按创建时间倒序：

```json
{ "data": [
  { "id": "clx...", "ip": "127.0.0.1", "userAgent": "Mozilla/5.0 …",
    "createdAt": "2026-10-22T01:00:00.000Z", "expiresAt": "2026-10-22T13:00:00.000Z" }
] }
```

### `DELETE /admin-auth/sessions/:id`

撤销自己的某个会话，返回 `{ "data": { "ok": true } }`。会话不存在、不属于自己或已失效 → `404 NOT_FOUND`。撤销当前正在使用的会话等价于退出登录（cookie 不会被这个接口清掉，但 token 立即失效）。

### `GET /admin/accounts`

返回全部后台账号（按 `createdAt` 升序）：

```json
{ "data": [ {
  "id": "clx...", "username": "boss", "displayName": "老板", "role": "owner",
  "status": "active", "mustChangePassword": false,
  "lastLoginAt": "2026-10-22T01:00:00.000Z",
  "createdAt": "2026-10-22T00:00:00.000Z", "isSelf": true
} ] }
```

### `POST /admin/accounts`

```json
{ "username": "alice", "displayName": "Alice", "password": "至少 12 位", "role": "operator" }
```

- `username`：小写字母、数字、下划线、连字符，3–32 位，且必须以字母或数字开头；服务端会 `trim` 并转小写
- `displayName`：1–40 位
- `password`：≥12 位，且不能是同一字符重复、不能以 `0123` / `1234` / `abcd` / `qwer` / `password` / `admin` 开头
- `role`：`owner` / `admin` / `operator` / `viewer`

返回新建的 `AdminAccountView`。新账号带 `mustChangePassword: true`（由后台前端引导首次登录改密，服务端不拦截请求）。

错误：`409 CONFLICT`（用户名已存在）；`400 VALIDATION_FAILED`；`403 ADMIN_ONLY`。

### `PATCH /admin/accounts/:id`

```json
{ "displayName": "Alice（运营）", "role": "operator", "status": "disabled" }
```

三个字段都可选，只传要改的。返回更新后的 `AdminAccountView`。

保护规则：

- **不能停用自己** → `400 VALIDATION_FAILED`「不能停用自己」
- **不能停用或降级最后一个可用 owner** → `409 CONFLICT`（统计时只算 `status = 'active'` 的其他 owner）
- `status` 改成 `disabled` 时，立即撤销该账号的全部后台会话
- `status` 只接受 `active` / `disabled`
- 账号不存在 → `404 NOT_FOUND`

### `POST /admin/accounts/:id/reset-password`

```json
{ "password": "可选，不传则服务端生成" }
```

不传 `password` 时服务端生成一个 20 位随机密码（去掉了容易看错的字符）：

```json
{ "data": { "password": "Xk7m..." } }
```

- 明文密码**只在这个响应里出现这一次**，服务端只存 scrypt 哈希
- 目标账号的 `mustChangePassword` 置为 `true`，并撤销其全部后台会话
- 密码不满足强度要求 → `400 VALIDATION_FAILED`；账号不存在 → `404 NOT_FOUND`

### `DELETE /admin/accounts/:id`

**刻意不支持删除**：硬删会让历史审计记录指向一个不存在的账号。要下线账号请用 `PATCH` 把 `status` 改成 `disabled`。

> 当前实现注意：这个路由抛的是普通 `Error`（只附加了 `status: 405`），会被全局异常过滤器当成未捕获异常，实际响应是 **500 `INTERNAL`**，不是 405。见「错误码速查」。
