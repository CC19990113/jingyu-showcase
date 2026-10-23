# 鲸屿 · 设计系统 v2

> 范围：**只有设计决策与实现工单**，本文件不含任何代码改动。
> 基准：`docs/prototype/index.html` + 现网 `http://127.0.0.1:5173`（前台）与 `5174`（后台）。
> 结论先行：现有深空暗色是**要保留的资产**，这一版不是换皮，是补三样东西——**亮色主题**、**为「可复现」而定的信息层级**、**把装饰性默认味砍掉**。

---

## 0. 方向与原则

### 0.1 一句话

**这不是一个「技术博客」，是一本公开的实验记录本。** 帖子 = 一次可复现的实测 + 一份可粘贴的产物；界面要为「我要不要信这条记录、我能不能直接抄」服务，而不是为「内容真好看」服务。

### 0.2 五条原则

| # | 原则 | 具体含义 | 反例（现在存在的问题） |
|---|---|---|---|
| 1 | **可复现优先** | 影响「能不能抄」的信息（内核 / 平台 / 模型）必须和标题同一视觉层，不能沉到页脚 | 环境 pill 现在在卡片最右下，和点赞数平级 |
| 2 | **等宽承担内容，不承担装饰** | mono 只用于「值」（版本号、路径、代码、计数）；标签永远用 sans | 反过来「小标签用等宽」正是 AI 生成页的默认味 |
| 3 | **形状分级** | 内容单元 = 描边 + 圆角 14；工具面板 = 圆角 10；右栏注记 = 没有形状 | 右栏 4 张卡、帖子卡、后台面板全用同一个壳 |
| 4 | **颜色是语义不是装饰** | 蓝 = 交互；青 = 已确认的事实；琥珀 = 缺省 / 版本风险；红 = 故障 / 驳回；板块色只出现在 3px 边条、板块标签、左导航圆点 | 状态徽章 5 种彩色 pill 堆在标题上方，颜色失去区分力 |
| 5 | **排版有纪律** | 4 档字重（400/500/600/700）、6 档字号；**中文不加字距**；数字统一 `tabular-nums` | CSS 里 `letter-spacing: 1.4px` 拍在中文小组标题上；字重用了 650 / 750 / 800 |

### 0.3 记忆点只给一个地方

> 只让一个东西高调：**「环境条 + 产物块」这套可复现语义**。
> 其余全部安静：无渐变 hero、无逐卡入场、无重复卡片壳、无装饰性光晕。

这条决定了后面所有取舍。看到任何「想加个效果」的冲动，先问它服务的是不是这一件事。

---

## 1. 色板

### 1.1 暗色主题（默认）

保留现有深空底，只做**职责拆分**：品牌色用于描边/图标，实心按钮单独一个填充色（因为白字压 `#4D6BFE` 只有 4.33:1，压 `#4560F2` 是 4.99:1，肉眼几乎看不出差别，但过 AA）。

| Token | 值 | 用途 | 对 `#121826` 对比度 |
|---|---|---|---|
| `--c-bg` | `#0A0D14` | 页面底 | — |
| `--c-bg-soft` | `#0E1320` | 环境条底、输入框内衬 | — |
| `--c-panel` | `#121826` | 卡片 / 面板 | — |
| `--c-panel-2` | `#161E2E` | 卡片 hover、弹出层 | — |
| `--c-panel-3` | `#1A2334` | 行内 code 底 | — |
| `--c-line` | `#1F2839` | 主描边 | — |
| `--c-line-soft` | `#18202F` | 分隔线、次级描边 | — |
| `--c-ink` | `#E8EDF7` | 正文 / 标题 | 15.10 |
| `--c-muted` | `#9FB0CB` | 摘要、次要文字 | 8.06 |
| `--c-faint` | `#77879F` | 元信息、标签 | 4.86 |
| `--c-prose` | `#D5DEEE` | 长文正文 | 12.5 |

`--c-faint` 4.86:1 是现有值，够 11px 小字用，别再往下调（现在还有 `#5D6B82` 3.29:1 在用，只能给 disabled）。

### 1.2 亮色主题

不是「白底 SaaS」，也不是「暖米色纸感」（后者是 AI 生成页第一号特征）。取的是**海雾**：冷调、微蓝、比纯白低一档，和暗色的深空是同一个色相家族的昼夜对偶。

| Token | 暗色 | 亮色 | 说明 |
|---|---|---|---|
| `--c-bg` | `#0A0D14` | `#EDF1F7` | 页面底：冷雾蓝灰 |
| `--c-bg-soft` | `#0E1320` | `#E4EAF3` | |
| `--c-panel` | `#121826` | `#FFFFFF` | 卡片底用纯白，靠页面底拉开层次 |
| `--c-panel-2` | `#161E2E` | `#F6F8FC` | |
| `--c-panel-3` | `#1A2334` | `#EEF2F8` | |
| `--c-line` | `#1F2839` | `#D6DEEE` | |
| `--c-line-soft` | `#18202F` | `#E6EBF4` | |
| `--c-ink` | `#E8EDF7` | `#0F1B2D` | 亮底 17.28 |
| `--c-muted` | `#9FB0CB` | `#4A5A73` | 亮底 6.17 |
| `--c-faint` | `#77879F` | `#5C6B85` | 亮底 4.75（11px 最小字也达标） |
| `--c-prose` | `#D5DEEE` | `#1B2740` | |
| `--c-topbar` | `rgba(10,13,20,.82)` | `rgba(255,255,255,.86)` | 顶栏毛玻璃底 |
| `--c-focus` | `#6E86FF` | `#3B55E6` | focus ring |
| `--c-selection` | `rgba(77,107,254,.34)` | `rgba(59,85,230,.20)` | |
| `--c-scroll` | `#253047` | `#C3CEE0` | 滚动条滑块 |
| `--c-scroll-hover` | `#33415F` | `#A9B8CF` | |
| `--c-on-active` | `#FFFFFF` | `#3B55E6` | 压在品牌**浅色 wash** 上的文字（`.navitem.on` / `.top-nav a.on` / `.mini-btn:hover`） |
| `--c-sweep` | `rgba(255,255,255,.05)` | `transparent` | 卡片光扫只在暗色存在 |
| `--c-lift` | `0 6px 20px rgba(0,0,0,.30)` | `0 6px 18px rgba(15,27,45,.10)` | hover 阴影 |

### 1.3 语义色

| 语义 | 暗色 | 亮色 | 用法 |
|---|---|---|---|
| 主色 / 信息 `--c-brand` | `#4D6BFE` | `#3B55E6` | 描边、图标、focus、强调；`antd.colorInfo` 也用它 |
| 主色填充 `--c-brand-fill` | `#4560F2` | `#3B55E6` | 实心按钮底（白字 4.99 / 5.79） |
| 主色文字 `--c-brand-2` | `#6E86FF` | `#2D4BD8` | 文字链接（5.50 / 5.96，`#4D6BFE` 当文字只有 4.09，不许当链接色） |
| 成功 `--c-green` | `#34D399` | `#15803D` | 已解决、通过、已采纳 |
| 警告 `--c-amber` | `#FBBF24` | `#B45309` | 待审、缺省环境、版本风险 |
| 危险 `--c-red` | `#F87171` | `#B91C1C` | 驳回、举报、表单错误 |
| 辅助强调 `--c-purple` | `#A78BFA` | `#7C3AED` | 深读板块、被引用/关联条目的次级强调 |
| 辅助青 `--c-teal` | `#2DD4BF` | `#0F766E` | **只表示「已确认的事实」**：已验证用户、当前内核、已填写且核实的平台 |

> **信息色刻意等于主色**：这套系统里「信息」和「品牌」是同一件事，不要造出第二支蓝。紫色是「辅助强调」，不是「信息」。

### 1.4 板块色：单一数据源 + 按主题推导

板块色的真源是数据库（seed 在 `packages/shared/src/constants.ts`，运营可在 `apps/backend` 的板块管理里用 `<input type="color">` 改）。所以**不能**在前端写一张 slug → 亮色 hex 的映射表（运营一改就失效）。

做法：`--acc`（DB 原色）只用于 **3px 边条 / 左导航圆点**；凡是**文字**，按主题做 OKLCH 亮度钳制：

```css
/* --acc 由 TSX 行内 style 写在卡片/标签的祖先上；这里只读不写，
   否则会覆盖掉继承来的值。缺省时退回品牌色。 */
.cat-tag {
  color: var(--acc, var(--color-brand));                                      /* 老浏览器降级 */
  color: oklch(from var(--acc, var(--color-brand)) max(l, 0.75) c h);         /* 暗色：只提亮太暗的 */
}
[data-theme='light'] .cat-tag {
  color: var(--acc, var(--color-brand));
  color: oklch(from var(--acc, var(--color-brand)) min(l, 0.52) c h);         /* 亮色：只压暗太亮的 */
}
```

> 已在实际 Chrome 中验证：`oklch(from …)` + `min()` / `max()` 可用；`--acc` 缺失时走 `var()` 兜底；`.cat-tag` 自身**不要**声明 `--acc`（行内样式在祖先上，本级声明会赢过继承，把板块色全变成品牌蓝）。
>
> 边条（`.post-card::before`、左导航圆点、后台板块列表的色块）在亮色下建议也做一次轻微压暗：`background: oklch(from var(--acc) min(l, 0.66) c h)`。3px 细条不需要满足文字对比度，但纯 `#FBBF24` 划在白底上「不在场」，压暗一档才看得见。

推导出来的实际值（**仅供校对，不要硬编码**）：

| 板块 | DB 原色 | 暗色文字（≥0.75） | 对比度 | 亮色文字（≤0.52） | 对白底 | 对页面底 `#EDF1F7` |
|---|---|---|---|---|---|---|
| 排障 | `#F87171` | `#FF7E7D` | 7.20 | `#B5333A` | 6.00 | 5.29 |
| 抄作业 | `#4D6BFE` | `#789FFF` | 6.93 | `#3B53E5` | 5.91 | 5.21 |
| 深读 | `#A78BFA` | `#B498FF` | 7.49 | `#7051BA` | 5.87 | 5.18 |
| 作品 | `#2DD4BF` | `#2DD4BF` | 9.53 | `#00806F` | 4.86 | 4.29 |
| 踩坑实测 | `#FBBF24` | `#FBBF24` | 10.62 | `#945C00` | 5.54 | 4.89 |
| 闲聊 | `#94A3B8` | `#A0AFC5` | 7.96 | `#5C6A7D` | 5.51 | 4.86 |

现状有多糟（白底对比度）：`#2DD4BF` **1.86**、`#FBBF24` **1.67**、`#F87171` **2.77**、`#A78BFA` **2.72**、`#94A3B8` **2.56**。全部不达标，这不是审美，是可读性。

**降级路径**：浏览器不支持相对颜色语法时落到 `color: var(--acc)`，亮色主题下黄/青板块标签会变淡。接受（Chrome 119+ / Safari 16.4+ / Firefox 128+ 都已支持）。

### 1.5 颜色的使用纪律

- 一个页面最多两种强调色在动（品牌蓝 + 一个语义色）。青只给「已验证」，不许拿来做装饰高亮。
- 所有 `rgba(77,107,254,.14)` 这类硬编码，一律换成 `color-mix(in oklab, var(--color-brand) 14%, transparent)`。**否则亮色主题下全部变成脏灰色**（现有 15 处）。
- 所有 `#fff` 文字（现有 8 处：`.skip-link` / `.top-nav a.on` / `.navitem.on` / `.chip.on` / `.post-card:hover .post-title` / `.prose h1-h3` / `.prose strong` / `.mini-btn:hover`）在亮色主题里大部分会直接消失。规则：**压在品牌填充上的白字保留**；**压在品牌浅色 wash 上的字改 `--c-on-active`**（暗 `#FFFFFF` / 亮 `#3B55E6`）；**标题和加粗改 `var(--color-ink)`**。
- 头像底色（`AVATAR_COLORS` 7 色 + 白字）现在最差的组合是白字压 `#FBBF24` = **1.67**。统一做 `background: oklch(from <原色> min(l, 0.55) c h)`，最差压到 4.3:1（首字是 `aria-hidden` 装饰，够用）。

---

## 2. 字体

### 2.1 两个家族

```css
--f-sans:
  "Inter", -apple-system, BlinkMacSystemFont, "Segoe UI",
  "PingFang SC", "Hiragino Sans GB", "Noto Sans SC", "Microsoft YaHei UI", "Microsoft YaHei",
  system-ui, sans-serif;

--f-mono:
  ui-monospace, "SF Mono", "JetBrains Mono", "Cascadia Mono", Menlo, Consolas, "Liberation Mono",
  "PingFang SC", "Microsoft YaHei", monospace;
```

**为什么不引中文字体**（这是决策，不是偷懒）：一个覆盖常用字的 Noto Sans SC 子集 3–8 MB；这个站的用户在自己机器上跑 agent，最反感的就是「打开个论坛先下 5 MB」。所以字体预算不花在「换一个中文脸」，花在两件更有用的事上：

1. **等宽栈的确定性**——环境条、版本号、patch 是内容本体，不是装饰。`--f-mono` 末尾**必须带 CJK fallback**，否则 Windows 上代码块里的中文注释会掉到 `NSimSun`，一眼廉价。
2. **字重 / 字号 / 字距纪律**——比换字体有效十倍。

sans 栈的顺序理由：Latin 优先（Inter → SF Pro Text → Segoe UI），中文随后（PingFang → 冬青 → Noto → 雅黑）。这样 Windows 上英文数字用 Segoe UI，中文用雅黑，不会两边都难看。

> 可选 P2：自托管 JetBrains Mono 的 Latin 子集（OFL 许可，约 30–60 KB woff2，400/500 两档）。做了以后版本号、patch 的辨识度会明显提升。不做也不影响本文件任何结论。

### 2.2 字号阶梯

| 角色 | px / line-height | 字重 | 字距 | 用在哪 |
|---|---|---|---|---|
| `.t-11` 角标 | 11 / 1.45 | 600 | 0 | 板块标签、状态徽章、环境条标签、计数 |
| `.t-12` 元信息 | 12 / 1.55 | 400 | 0 | 卡片页脚、右栏正文、表格表头 |
| `.t-13` 概要 | 13 / 1.65 | 400 | 0 | 卡片摘要、表单、后台表格正文 |
| `.t-14` UI 正文 | 14 / 1.70 | 400 | 0 | body 默认（沿用现有） |
| `.t-155` 卡片标题 | 15.5 / 1.55 | 600 | 0 | 信息流标题（最多 2 行） |
| `.t-prose` 长文 | 15 / 1.85 | 400 | 0 | 详情页正文（现有 14.5/1.92，略提字号、收行高） |
| `.t-h2` 长文小标题 | 18 / 1.5 | 600 | -0.01em | `.prose h2` |
| `.t-h1` 页面标题 | 22 / 1.4 | 600 | -0.01em | banner、后台页标题 |
| `.t-title` 详情标题 | 26 / 1.5 | 700 | -0.01em | 帖子详情；≤860 降 21 / 1.55 |
| `.t-code` 代码 | 12.5 / 1.75 | 400 | 0 | `pre`、行内 code |

移动端：`.t-title` → 21、`.t-h1` → 19、其余不变。

### 2.3 字重与字距

- **只有 400 / 500 / 600 / 700。** 现有 `650`（`.post-title`、`.prose strong`）、`750`（`.d-title`）、`800`（`.lv`）全部改掉。PingFang 只有 Regular/Medium/Semibold/Bold，650/750 走浏览器插值或合成，中文会糊边。
- **中文 `letter-spacing: 0`。** 大字标题允许 `-0.01em` 的收紧（这是 CJK 的负字距惯例，视觉上更整）。删除现有 `.brand-name` .5px、`.navgroup-title` 1.4px、`.rail-card h3` .9px、`.d-title` .2px、`.post-title` .1px。
- `text-transform: uppercase` 对中文是空操作，但对夹在中文里的英文标签会全部大写——**一律去掉**。

### 2.4 数字与代码

- 所有计数（回复/赞/浏览）、表格数字、右栏统计：`font-variant-numeric: tabular-nums`。
- 版本号、平台串、模型名、产物文件名：`font-family: var(--f-mono)`。**标签（内核/平台/模型）用 sans**，值用 mono——这条同时就是反「mono 当小标签」的默认味。
- 代码块语法色走 token：`--color-syn-key` / `--color-syn-str` / `--color-syn-comment`（亮色 `#1D4ED8` / `#0F766E` / `#5F6F88`），不再硬编码 `#7aa2ff`。
- 行内 code：底 `--color-panel-3`、描边 `--color-line`、文字 `--color-syn-str`，亮色主题下换成深青（`#0F766E`），保持「代码是青的」这个既有记忆点。

### 2.5 排版细节

- **行宽**：详情页整栏收到 **680px**，正文占满整栏 = 15px 中文约 45 字/行（CJK 的合理区间 35–48 字/行）。`pre` 允许横向滚动，**不为了代码把阅读列拉宽**——代码宽了，正文就永远读不完一行。
- 中文段落两端对齐会撕裂空格，**永远左对齐、ragged right，不用 `text-align: justify`**。
- 标题最多 2 行；超过 2 行截断（标题里堆整段报错原文是内容问题，不是布局问题——用 `-webkit-line-clamp: 2`）。
- **不用衬线**：本系统只有 sans + mono 两个家族（理由见 §5.3 第 1 条）。

---

## 3. 布局

### 3.1 三栏骨架

```
┌──────────────────────────────────────────────────────────────────────┐
│ topbar 60px  sticky     品牌 · 主导航 · 搜索 · [主题] · 通知 · 发帖   │
├──────────┬───────────────────────────────────────┬───────────────────┤
│ 216px    │ 中间流（fluid，1400 时约 792px）       │ 300px             │
│ 左导航   │                                       │ 右栏（注记，非卡片）│
│ 板块     │  masthead（无卡片，排版主导）          │ 社区规则           │
│ 我的     │  ─────────────────────────            │ 编辑精选（带序号） │
│ 当前内核 │  [最新][最热][待解决]        10 个帖子 │ 版本提示           │
│          │  ┌▏板块 状态                    内核┐ │ 成员               │
│          │  │▏ 标题                            │ │                    │
│          │  │▏ 摘要（2 行）                     │ │                    │
│          │  │▏ 作者 Lv  回复 赞 浏览           │ │                    │
│          │  └──────────────────────────────────┘ │                    │
└──────────┴───────────────────────────────────────┴───────────────────┘
   max-width 1400 · padding 22/24/80 · gap 22
```

**关键改动：右栏不再是卡片。** 三个理由：① 规则 / 榜单 / 统计是三种不同形态的内容，套同一个圆角框是把它们强行拉平；② 一整屏出现 5 个同圆角同阴影的盒子，就是「SaaS 卡片套件」这个 AI 特征；③ 去掉壳以后，信息流卡片才是页面上唯一「被托起来」的东西，层级立刻清楚。

```css
.rail-card {
  background: none;
  border: 0;
  border-radius: 0;
  padding: 0 0 18px;
  margin: 0;
}
.rail-card + .rail-card {
  border-top: 1px solid var(--c-line-soft);   /* 只用一条 hairline 分隔 */
  padding-top: 16px;
}
.rail-card h3 {           /* 删掉左边那条品牌色小竖条 */
  font-size: 12.5px; font-weight: 600; letter-spacing: 0;
  color: var(--c-muted); margin: 0 0 10px;
}
```

### 3.2 断点

| 视口 | 左栏 | 中栏 | 右栏 | 顶栏 |
|---|---|---|---|---|
| ≥ 1181 | 216 sticky | fluid | 300 sticky | 完整 |
| 861 – 1180 | 200 sticky | fluid | **收起为跨栏条带**，2 列 | 完整 |
| 641 – 860 | Drawer（已有） | 单列 | 条带 2 列 | 主导航隐藏（已有行为），主题按钮保留 |
| ≤ 640 | Drawer | 单列，padding 16/14 | 条带 1 列 | 主题按钮隐藏（`.hide-narrow`），改从 Drawer 进；搜索框靠现有 `flex-1 min-w-0` 自然收缩，不加新规则 |

右栏「跨栏条带」是纯 CSS，不动 TSX：

```css
@media (max-width: 1180px) {
  .app-layout { grid-template-columns: 200px minmax(0, 1fr); }
  .side-right {
    position: static;
    grid-column: 1 / -1;
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 0 26px;
  }
  .rail-card { padding-bottom: 14px; }
}
@media (max-width: 860px) {
  .app-layout, .app-layout.no-right { grid-template-columns: 1fr; }
  .side-right { grid-template-columns: repeat(2, minmax(0, 1fr)); }
}
@media (max-width: 640px) {
  .side-right { grid-template-columns: 1fr; }
  .rail-card + .rail-card { border-top: 0; padding-top: 0; }
}
```

> 注意 `.rail-card + .rail-card` 的顶部 hairline 在 2 列网格里会落在第 2、4 个上，几何上仍然成立（每个顶边都在上一行下面）。小屏单列时去掉。

### 3.3 卡片信息层级

```
┌▏[板块] [已解决] [审核中] 附产物3               内核 0.2.1-rc.2 ┐   ← 1. 资格行
│▏ 标题（15.5/1.55/600，最多 2 行）                              │   ← 2. 标题
│▏ 摘要（13/1.65/muted，最多 2 行）                              │   ← 4. 摘要
│▏ (头像) 林海 Lv4        回复 3  赞 2  浏览 40                  │   ← 5. 社交证据
└──────────────────────────────────────────────────────────────┘
   ▲ 3px 板块色条（唯一使用板块色的地方之一）
```

**为什么这么排：**

1. **标题最重要**。扫信息流时 90% 的注意力在标题；它决定「要不要点」。
2. **内核版本第二重要，所以从页脚搬到第一行右侧。** 这不是排版偏好，是内容属性：这个社区的核心承诺是「能抄」，而一条没写内核版本的「抄作业」是**负资产**。因此版本必须紧贴标题，和标题一起被读到；点赞数属于「别人怎么想」，属于页脚。
3. **板块色条 + 板块标签**回答「这是哪一类问题」，但只给一个色条和一个小标签，不做整块染色。
4. **摘要**是验证环节，2 行足够；超出就截断，不要 3 行——3 行会把页脚推出首屏节奏。
5. **互动数放最后**，图标 12px、颜色 `--c-faint`，只做弱信号。
6. **状态徽章最多 2 个用色**（已解决 = 绿、审核中 = 琥珀）。「附产物」不带颜色，改成中性 `📎 3` 样式的图标 + 数字。5 种彩色 pill 会变成标签池。

卡片本体保留 brief 指定的：左侧 3px 板块条、圆角 14、hover 抬升 2px。**但删掉「每张卡依次上浮入场」**（理由见 §5.4）。

### 3.4 详情页的阅读列

- 容器 `NarrowPage width={680}`（现在是默认 1000）。
- 标题 → 环境条 → 正文，三者在同一竖轴，左对齐。
- 环境条重做：`●` 已填 / `○` 未填，标签 sans、值 mono，未填用琥珀：

```
● 内核 0.2.1-rc.2    ● 平台 darwin-arm64    ○ 模型 未标注
```

这条把社区规则（报错帖必须带内核版本 + 平台）直接编进界面：**没填的人自己看得见**，不用版主去点。

### 3.5 前台 vs 后台：密度差异

| 维度 | 前台 `apps/web` | 后台 `apps/backend` |
|---|---|---|
| 阅读姿态 | 阅读感：宽松、留白大 | 工具感：一屏尽量多信息 |
| 基准字号 / 行高 | 14 / 1.70 | 13 / 1.60 |
| 面板圆角 | 14 | 10 |
| 面板内距 | 16–18 / 18–20 | 12–14 / 14–16 |
| 卡片间距 | 12 | 8 |
| 页标题 | 22（详情 26） | 20 |
| 表格 | 无 | antd `Table size="small"`，`cellPaddingBlockSM: 8`、`cellPaddingInlineSM: 10`，行高 ≈ 40 |
| 动效 | hover 抬升 2px（仅暗色有光扫） | 无抬升、无光扫，只有行 hover 底色 |
| 数字 | `tabular-nums` | `tabular-nums` + mono |
| 最大宽 | 1400 / 详情 680 | 1180 |
| 颜色预算 | 品牌 + 2 种语义色 | 语义色 4 档（待审 / 通过 / 驳回 / 举报）必须可区分 |
| 按钮 | 默认尺寸（36） | 工具条一律 `size="small"`（28） |

---

## 4. 主题切换实现

### 4.1 CSS 变量分层

**三层，各管一件事：**

1. `:root` / `[data-theme='light']` —— **原始 token**（`--c-*`），全站唯一改明暗的地方，**不放在任何 `@layer` 里**。
2. `@theme` —— 把原始 token 映射成 Tailwind 能生成工具类的名字（`--color-*`），值写 `var(--c-*)`。
3. `@layer components` —— 已有组件样式，继续用 `var(--color-*)`，**一个字符都不用改**。

```css
@layer theme, base, components, utilities;
@import 'tailwindcss/theme.css' layer(theme);
@import 'tailwindcss/utilities.css' layer(utilities);
@source './';

/* 1. 原始 token（unlayered，两个块，必须相邻且 dark 在前） */
:root { /* …§1.1 的 --c-* 全量 … */ }
[data-theme='light'] { /* …§1.2 的 --c-* 全量，同序 … */ }

/* 2. 映射 */
@theme {
  --color-bg: var(--c-bg);
  --color-bg-soft: var(--c-bg-soft);
  --color-panel: var(--c-panel);
  --color-panel-2: var(--c-panel-2);
  --color-panel-3: var(--c-panel-3);
  --color-line: var(--c-line);
  --color-line-soft: var(--c-line-soft);
  --color-ink: var(--c-ink);
  --color-muted: var(--c-muted);
  --color-faint: var(--c-faint);
  --color-brand: var(--c-brand);
  --color-brand-2: var(--c-brand-2);
  --color-teal: var(--c-teal);
  --color-amber: var(--c-amber);
  --color-red: var(--c-red);
  --color-green: var(--c-green);
  --color-purple: var(--c-purple);
  --color-code-bg: var(--c-code-bg);
  --color-code-fg: var(--c-code-fg);
  --color-syn-key: var(--c-syn-key);
  --color-syn-str: var(--c-syn-str);
  --color-syn-comment: var(--c-syn-comment);

  --font-sans: var(--f-sans);
  --font-mono: var(--f-mono);
}
```

**为什么用普通 `@theme` 而不是 `@theme inline`（这条很重要，已实测）：**

- 普通 `@theme`：会额外生成 `:root { --color-panel: var(--c-panel) }`，工具类输出 `var(--color-panel)`。**组件层里现存的一百多处 `var(--color-ink)` / `var(--color-panel)` 全部继续有效。**
- `@theme inline`：**不再生成 `--color-*` 变量**，只把值内联进工具类。一旦切过去，`@layer components` 里所有 `var(--color-*)` 立刻变成未定义，整页掉色。
- 两者都能响应 `[data-theme]`，因为 `data-theme` 挂在 `<html>` 上——和 `:root` 是同一个元素，`var()` 在同一个元素上完成替换。**`data-theme` 千万不要挂到 `<body>` 上**，那样普通 `@theme` 就失效了。
- 已验证：`bg-panel/40`、`border-brand/60` 这类透明度修饰符在 `var()` 值下仍然工作（Tailwind 会输出 `@supports (color: color-mix(in lab, red, red))` + `color-mix(in oklab, var(--color-panel) 40%, transparent)` 兜底）。

### 4.2 antd 6 怎么切

antd 6（当前 `6.6.7`）继续沿用 v5 的主题 API：`theme.darkAlgorithm` / `theme.defaultAlgorithm` + `ConfigProvider theme`。**不需要**换组件、不需要 `cssVar`。

关键变化：**主题对象从常量改成函数**，组件级覆盖也要跟着分主题，否则亮色下会出现白底白字（现在的 `Tabs.itemSelectedColor: '#FFFFFF'`、`Table.headerBg: '#0E1320'`、`Modal.contentBg: '#121826'` 就是三颗雷）。

新建 `apps/web/src/lib/antdTheme.ts`（后台同名文件，内容一致，只多一段表格密度）：

```ts
import { theme } from 'antd';
import type { ThemeConfig } from 'antd';
import type { ThemeMode } from './theme';

const SHARED = {
  fontFamily: 'var(--f-sans)',
  fontSize: 14,
  borderRadius: 10,
  borderRadiusLG: 14,
  controlHeight: 36,
} satisfies ThemeConfig['token'];

const DARK: ThemeConfig = {
  algorithm: theme.darkAlgorithm,
  token: {
    ...SHARED,
    colorPrimary: '#4560F2',
    colorInfo: '#4D6BFE',
    colorLink: '#6E86FF',
    colorSuccess: '#34D399',
    colorWarning: '#FBBF24',
    colorError: '#F87171',
    colorBgBase: '#0A0D14',
    colorBgLayout: '#0A0D14',
    colorBgContainer: '#121826',
    colorBgElevated: '#161E2E',
    colorBorder: '#1F2839',
    colorBorderSecondary: '#18202F',
    colorText: '#E8EDF7',
    colorTextSecondary: '#9FB0CB',
    colorTextTertiary: '#77879F',
    colorTextQuaternary: '#5D6B82',
  },
  components: {
    Button: { primaryShadow: '0 2px 14px rgba(77,107,254,.34)' },
    Tabs: { itemSelectedColor: '#FFFFFF' },
    Table: { headerBg: '#0E1320', rowHoverBg: '#161E2E' },
  },
};

const LIGHT: ThemeConfig = {
  algorithm: theme.defaultAlgorithm,
  token: {
    ...SHARED,
    colorPrimary: '#3B55E6',
    colorInfo: '#3B55E6',
    colorLink: '#2D4BD8',
    colorSuccess: '#15803D',
    colorWarning: '#B45309',
    colorError: '#B91C1C',
    colorBgBase: '#FFFFFF',
    colorBgLayout: '#EDF1F7',
    colorBgContainer: '#FFFFFF',
    colorBgElevated: '#FFFFFF',
    colorBorder: '#D6DEEE',
    colorBorderSecondary: '#E6EBF4',
    colorText: '#0F1B2D',
    colorTextSecondary: '#4A5A73',
    colorTextTertiary: '#5C6B85',
    colorTextQuaternary: '#8794A8',
  },
  components: {
    Button: { primaryShadow: '0 2px 10px rgba(59,85,230,.22)' },
    Tabs: { itemSelectedColor: '#0F1B2D' },
    Table: { headerBg: '#F4F7FC', rowHoverBg: '#F6F8FC' },
  },
};

export function buildAntdTheme(mode: ThemeMode): ThemeConfig {
  return mode === 'light' ? LIGHT : DARK;
}

/** 后台专用：在基础主题上叠表格密度与内容区底色。 */
export function buildAdminTheme(mode: ThemeMode): ThemeConfig {
  const base = buildAntdTheme(mode);
  return {
    ...base,
    token: { ...base.token, fontSize: 13 },
    components: {
      ...base.components,
      Table: {
        ...base.components?.Table,
        cellPaddingBlockSM: 8,
        cellPaddingInlineSM: 10,
        headerSplitColor: 'transparent',
      },
    },
  };
}
```

**删掉**：`Modal: { contentBg, headerBg }`（交给算法推导，硬编码一定会错主题）。

`fontFamily: 'var(--f-sans)'` 是有意的：antd 把 token 直接写进 CSS-in-JS 声明，`var()` 在其中合法，这样字体栈只有一份来源。

### 4.3 持久化与首屏不闪

新建 `apps/web/src/lib/theme.ts`（后台同名一份）：

```ts
import { useSyncExternalStore } from 'react';

export type ThemeMode = 'dark' | 'light';

const KEY = 'jingyu.theme';
const listeners = new Set<() => void>();

export function readStoredTheme(): ThemeMode {
  try {
    return localStorage.getItem(KEY) === 'light' ? 'light' : 'dark';
  } catch {
    return 'dark';
  }
}

export function getTheme(): ThemeMode {
  return document.documentElement.dataset.theme === 'light' ? 'light' : 'dark';
}

export function applyTheme(mode: ThemeMode): void {
  const root = document.documentElement;
  root.dataset.theme = mode;
  root.style.colorScheme = mode;
  const meta = document.querySelector('meta[name="theme-color"]');
  meta?.setAttribute('content', mode === 'light' ? '#EDF1F7' : '#0A0D14');
}

export function setTheme(mode: ThemeMode): void {
  try {
    localStorage.setItem(KEY, mode);
  } catch {
    /* 隐私模式 / 配额满：只在本次会话生效 */
  }
  applyTheme(mode);
  listeners.forEach((fn) => fn());
}

function subscribe(fn: () => void): () => void {
  listeners.add(fn);
  return () => listeners.delete(fn);
}

export function useThemeMode(): ThemeMode {
  return useSyncExternalStore(subscribe, getTheme, () => 'dark' as ThemeMode);
}

if (typeof window !== 'undefined') {
  window.addEventListener('storage', (event) => {
    if (event.key !== KEY) return;
    applyTheme(event.newValue === 'light' ? 'light' : 'dark');
    listeners.forEach((fn) => fn());
  });
}
```

`apps/web/index.html` 与 `apps/backend/index.html`，在 `<meta name="color-scheme">` 之后、模块脚本之前插**内联**脚本（内联才不会被延迟执行，`type="module"` 天然 defer，所以这一段一定先跑）：

```html
<meta name="color-scheme" content="dark light" />
<meta name="theme-color" content="#0A0D14" />
<script>
  (function () {
    try {
      var light = localStorage.getItem('jingyu.theme') === 'light';
      var mode = light ? 'light' : 'dark';
      var el = document.documentElement;
      el.dataset.theme = mode;
      el.style.colorScheme = mode;
      var meta = document.querySelector('meta[name="theme-color"]');
      if (meta) meta.setAttribute('content', light ? '#EDF1F7' : '#0A0D14');
    } catch (e) {
      document.documentElement.dataset.theme = 'dark';
    }
  })();
</script>
```

同时 `<html lang="zh-CN">` 保持不动（不要写死 `data-theme="dark"`，交给脚本；写了也能被覆盖，但会和 localStorage 形成两份真源）。

**为什么不会闪**：主题只影响 CSS 变量，变量在 `<html>` 的 dataset 上解析；内联脚本在 `<head>` 里、首帧之前完成赋值，body 第一帧就是正确色。antd 的样式是在 React 挂载后注入的，这里不存在「先暗后亮」。

**默认值**：首次访问固定 **dark**，不跟 `prefers-color-scheme`。理由：品牌默认 + 结果可预测（截图、评审、协作时不会因为系统设置各自成画）。若以后要加「跟随系统」，必须引入第三态，本期不做。

### 4.4 入口挂载

`main.tsx` 改成两层：外层建 QueryClient 与主题订阅，内层 `Shell` 负责 `ConfigProvider`。

```tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { App as AntdApp, ConfigProvider } from 'antd';
import zhCN from 'antd/locale/zh_CN';
import { BrowserRouter } from 'react-router-dom';
import { AuthProvider } from '@/stores/auth';
import { applyTheme, readStoredTheme, useThemeMode } from '@/lib/theme';
import { buildAntdTheme } from '@/lib/antdTheme';
import App from './App';
import './index.css';

applyTheme(readStoredTheme());

const queryClient = new QueryClient({ /* 保持现状 */ });

function Shell() {
  const mode = useThemeMode();
  return (
    <ConfigProvider locale={zhCN} theme={buildAntdTheme(mode)}>
      <AntdApp>
        <BrowserRouter>
          <AuthProvider>
            <App />
          </AuthProvider>
        </BrowserRouter>
      </AntdApp>
    </ConfigProvider>
  );
}

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <Shell />
    </QueryClientProvider>
  </StrictMode>,
);
```

后台用 `buildAdminTheme(mode)`，其余相同。

### 4.5 切换控件

`apps/web/src/components/ThemeToggle.tsx`（后台同名）：

```tsx
import { Button, Tooltip } from 'antd';
import { Moon, Sun } from 'lucide-react';
import { setTheme, useThemeMode } from '@/lib/theme';

export function ThemeToggle({ size = 'middle' }: { size?: 'small' | 'middle' }) {
  const mode = useThemeMode();
  const next = mode === 'dark' ? 'light' : 'dark';
  const label = next === 'light' ? '切到浅色主题' : '切到深色主题';
  return (
    <Tooltip title={label}>
      <Button
        type="text"
        size={size}
        aria-label={label}
        onClick={() => setTheme(next)}
        icon={mode === 'dark' ? <Sun size={16} aria-hidden="true" /> : <Moon size={16} aria-hidden="true" />}
      />
    </Tooltip>
  );
}
```

摆放：
- 前台顶栏：**搜索框之后、通知之前**（`Topbar.tsx`，非登录态也显示，放在「登录」左边）。
- ≤640 的顶栏：加 `.hide-narrow` 隐藏该按钮（`.hide-narrow` 需要在 `index.css` 的 `@media (max-width: 640px)` 里新写一条 `display: none !important`，对齐现有 `.hide-mobile` 的做法）。入口改到 `RootLayout` 的 Drawer（Drawer 在 ≤860 才有）顶部一行「外观」+ `ThemeToggle`。
- 后台：`AdminLayout.tsx` 标题行右侧，与副标题同处一行，`size="small"`。

文案规则（照 skill 的要求）：**按钮说它会让什么发生**——「切到浅色主题」，不是「浅色 / 深色」二选一的状态名。图标画的是**目标状态**（暗色下显示太阳）。

### 4.6 已知边界（写进来免得后面当 bug 查）

- `localStorage` 按 **origin** 隔离。前台 5173、后台 5174 是两个 origin，**主题选择不会互相同步**。生产环境若挂在同一 host（不同路径），会自动同步。不要为此加跨域同步。
- 切换是**瞬时**的，不加全局 `transition`。两套 token 之间不存在有意义的中间色，过渡只会让文字在某些时刻对比度不达标。
- 打印样式不覆盖，交给浏览器（`color-scheme` 已设）。

---

## 5. 自我批判

按 skill 的两遍流程：先出计划，再拿它和「任何技术社区我都会这么设计」的版本对比，改掉。

### 5.1 哪些是「给任何技术社区都会这么设计」的默认方案

老实说，**骨架和底色都是**，我不假装它是原创：

- 三栏 + 左分类 + 右信息栏，是 V2EX / 掘金 / 少数派的中文技术社区既成惯例。
- 深色底 + 一枚品牌蓝 + 卡片流，是 2019 以后所有开发者产品落地页的默认。
- 帖子卡「圆角 + 左边色条 + hover 抬升」，是 brief 直接指定的。
- 右栏「社区规则 / 热榜 / 统计」三件套，是论坛模板自带的。

这些不是我的设计，是 brief 的资产和约束。**我的工作空间在细节里**：颜色怎么分职责、信息谁先谁后、哪些装饰该删、亮色主题怎么不落进 SaaS 白。所以本文件的自评标准不是「这几栏有没有新意」，而是**「把这些细节拿掉，这个页面还认得出是鲸屿吗」**。

### 5.2 哪些是专门为「DSH 插件作者」这一群人造的

1. **环境条是一等公民，而且「缺省」是可见的。** `● 内核 0.2.1-rc.2` / `○ 模型 未标注`。这不是表单校验，是把这个社区的立站规则（报错帖必须带内核版本 + 平台）编码进视觉：**没填的人自己看得见，不用版主去点**。任何通用论坛都不会有这个。
2. **卡片第一行右侧放内核版本，而不是放热度。** 因为对这群人来说：「这条能不能抄」比「这条热不热」重要一个量级。这是基于内容属性做的层级判断，不是排版趣味。
3. **等宽字体承担内容而非装饰。** 版本号、平台串、patch、预设文件名全是内容本体；同时立一条反向规则（标签永远 sans），这正好避开「monospace 当小标签字体」这个 AI 默认。
4. **产物卡是页面上的第二个「重对象」**（第一个是帖子标题）。文件头结构 + 复制是一级交互，不是附件缩略图。
5. **后台是一等公民。** 审核队列 / 举报 / 敏感词 / 审计日志的词表直接出现在导航上——这个社区是邀请制 + 有内容治理的，后台不是「顺便做的 CRUD」。
6. **右栏去卡片化**，因为这里装的是「站务注记」而不是内容单元；这同时让信息流卡片成为页面上唯一被托起的东西。

### 5.3 五类「AI 生成味」怎么避

| # | 特征 | 本系统怎么处理 |
|---|---|---|
| 1 | 暖米色 `#F4F1EA` + 高对比衬线 + terracotta `#D97757` | **零衬线**（全站只有 sans + mono 两个家族，且 mono 是内容需要）；亮色底是冷雾 `#EDF1F7`，不是米色；强调色是深海蓝 + 青，没有 terracotta |
| 2 | 近黑 + 单一酸性绿/朱红 | 底色是 brief 指定的蓝黑 `#0A0D14`（不是中性 `#0B0B0B`）；强调分三层职责：蓝 = 交互、青 = 已验证事实、琥珀 = 缺省/风险；青被明确限制不能当装饰高亮，也不是 CTA 色 |
| 3 | 报纸版式：发丝线、零圆角、密排栏 | 保留 14px 圆角；hairline 只出现在「右栏注记之间」和「顶栏底边」两处，不做栏线、不做分栏密排 |
| 4 | SaaS 卡片套件：一切同圆角同阴影 + 渐变洗底 | 形状分级：帖子卡 14 描边、后台面板 10、右栏无形状、代码块 10；**删掉 body 的两层 radial 渐变**和 banner 的渐变 hero；阴影只在 hover 时出现（`--c-lift`），常驻无阴影 |
| 5 | 模板化 chrome：字距拉开的全大写 eyebrow、`·` 中缀元信息、`WORD — fragment`、`→` 后缀 | 删掉所有 `letter-spacing` 标签（中文加字距本来就是错的）；元信息用**标签 + 值**两段式（`内核 0.2.1-rc.2`）替代 `A · B · C`；删掉 `ArtifactCard` 里的 `· note` 拼接；链接不加字符箭头，方向感用 11px 图标且只在行尾 |

**关于标点与序号的一条自觉**：右栏「编辑精选」保留 1–5 的序号——因为那**确实是按热度排的序**；而「装得上、跑得通、抄得走」虽然读起来像三步，不做 01/02/03 编号，它是一句口号不是流程。

### 5.4 我砍掉了什么（改了什么、改成什么、为什么）

这是本文件最该被审的一节。

| # | 砍掉 / 改掉 | 改成 | 为什么 |
|---|---|---|---|
| 1 | **body 的两层 radial 渐变**（蓝 + 青光晕） | 单色底 `var(--c-bg)` | 装饰性渐变洗底是「AI 产品页」第一视觉特征；而且它让「深空」变成了「发光背景」，卡片反而显得平。删掉后暗色更沉、亮色更好做 |
| 2 | **每张卡片依次上浮入场**（`animation: rise` + `index * 34ms`） | 无入场动画 | 全站每个元素 fade-and-slide-up 是生成式页面的招牌动作；对一个「刷一下就为了看新帖」的社区，入场动画只会延迟可读时间。列表变动应当**立刻可见** |
| 3 | **中文标签上的字距**（`.navgroup-title` 1.4px、`.brand-name` .5px、`.d-title` .2px、`.post-title` .1px、`.rail-card h3` .9px） | 全部 `letter-spacing: 0`（大标题允许 -0.01em） | ① 字距拉开的 eyebrow 标签是模板化 chrome；② 中文加正字距把字排散，是排版错误不是风格 |
| 4 | **右栏 4 张同款圆角卡** | 去壳，平面注记 + 相邻 hairline 分隔 | 三类内容（规则/榜单/统计）套同一个壳，是「SaaS 卡片套件」特征；去掉后信息流卡片成为页面唯一被托起的对象，层级才成立 |
| 5 | **右栏 h3 前的品牌色小竖条 ×4** | 删除 | 竖条这个视觉符号已经被「板块色条」占用；同一个符号表达两种含义 = 没有含义。重复四次 = 装饰 |
| 6 | **banner 的渐变 hero**（120° 蓝青渐变 + 径向高光 + 品牌描边 + 圆角） | 无容器排版块：22px 标题 + 14px 说明 + 1px 底边 | hero 用渐变高光是默认解法；这个社区最特征的东西是**帖子本身**，banner 不该抢。去掉容器后，页面第一屏的重心回到信息流 |
| 7 | **5 种彩色状态 pill 堆在标题上方** | 只留 2 种用色（已解决 = 绿 / 审核中 = 琥珀），「附产物」改中性图标 + 数字 | 每个状态都用彩色 pill 会变成标签池，颜色失去区分力，读者最后只看到一排彩色方块 |
| 8 | **非标准字重 650 / 750 / 800** | 只 400 / 500 / 600 / 700 | PingFang 只有 4 档；650/750 触发插值或合成加粗，中文字形发糊、边缘发灰 |
| 9 | **`#fff` 硬编码文字**（8 处） | 分三类：压品牌填充上保留白；压浅色 wash 上改 `--c-on-active`；标题/加粗改 `var(--color-ink)` | 亮色主题下白字直接消失，这是硬故障不是审美 |
| 10 | **板块色直接当文字色** | 按主题做 OKLCH 亮度钳制（暗 ≥0.75 / 亮 ≤0.52） | 现有 6 色在浅底上是 1.67–2.77:1，全部不达标。数据源在 DB，所以用公式推导而不是写死映射表 |
| 11 | **亮色主题的光扫** | `--c-sweep: transparent`，只保留抬升 | brief 要的「hover 抬升 2px + 光扫」是暗色语境下的效果；白底上白色光扫不可见，换成蓝色光斑会变脏。**这是对 brief 的一处有意偏离，已在 §5.5 标注** |
| 12 | **`.post-card:hover .post-title { color: #fff }`** | 删除整条 | 暗色下标题从 `#E8EDF7` 到 `#fff` 的对比度只有 1.07:1，感知不到；亮色下则是白字消失。收益为零、风险为正 |

### 5.5 对 brief 的两处有意偏离（请 lead 拍板）

1. **亮色主题不做光扫**（见上表 #11）。如果坚持要，唯一不脏的做法是把光扫换成「描边变亮」而不是扫光；建议不要。
2. **右栏不再是卡片**（见上表 #4）。brief 里「卡片：左侧 3px 板块色条、圆角 14px、hover 抬升 2px + 光扫」描述的是**帖子卡**，我按其原意只保留在信息流，没有外推到右栏和后台。

---

## 6. 改造清单

### P0 —— 不做则亮色主题不成立

> 两份 `index.css`（`apps/web/src/index.css` 与 `apps/backend/src/index.css`）**目前 910 行逐字节相同**。它们是两个独立应用、独立部署，**不做跨包引用**，改动必须两边同步打；建议在文件头加一行注释 `/* 设计系统 v2 · token 层与 apps/web/src/index.css 保持一致 */`。

| # | 文件 | 位置 | 改法 |
|---|---|---|---|
| P0-1 | `apps/web/src/index.css`、`apps/backend/src/index.css` | 第 15–38 行 `@theme` 之前 | 插入 `:root` 与 `[data-theme='light']` 两个 unlayered token 块（§1.1 / §1.2 全量，含 `--f-sans` / `--f-mono`） |
| P0-2 | 同上 | `@theme` 块 | 17 个颜色值改 `var(--c-*)`（现有 15–38 行的全部颜色项）；新增 `--color-code-bg/-fg/-syn-key/-syn-str/-syn-comment`；`--font-sans/--font-mono` 改为 `var(--f-sans)` / `var(--f-mono)`。**保持是 `@theme`，不要写 `inline`**（§4.1） |
| P0-3 | 同上 | `body`（58–69 行） | `background` 去掉两层 radial-gradient，改 `background: var(--c-bg)` |
| P0-4 | 同上 | 全文 | 29 处硬编码 `rgba(...)` → `color-mix(in oklab, var(--color-*) N%, transparent)`。分布：brand 15 / green 6 / teal 4 / amber 2 / white 1（`--c-sweep`）/ topbar 底 1 |
| P0-5 | 同上 | 152 / 222 / 293 / 375 / 434 / 635 / 663 / 739 行的 `#fff` | 按 §5.4 #9 分类替换；新增 `--c-on-active`（暗 `#FFFFFF` / 亮 `#3B55E6`） |
| P0-6 | 同上 | `.topbar` | `background: var(--c-topbar)`；`.d-title` 字号 25→26 / 行高 1.42→1.5 / 字重 750→700 / 字距 0；≤860 时 21 |
| P0-7 | 同上 | `.prose` | 15px / 1.85，颜色 `var(--c-prose)`；`h1–h3`、`strong` 用 `var(--color-ink)`、字重 600；`pre` 用 `--color-code-*`；`.k/.s/.c` 用 `--color-syn-*` |
| P0-8 | 同上 | `.prose code:not(pre code)` | 底 `--color-panel-3`、字 `--color-syn-str` |
| P0-9 | 同上 | `::-webkit-scrollbar-thumb` / `::selection` / `:focus-visible` | 改用 `--c-scroll`、`--c-scroll-hover`、`--c-selection`、`--c-focus` |
| P0-10 | `apps/web/src/lib/theme.ts`、`apps/backend/src/lib/theme.ts` | 新建 | 照抄 §4.3 |
| P0-11 | `apps/web/src/lib/antdTheme.ts`、`apps/backend/src/lib/antdTheme.ts` | 新建 | 前台用 `buildAntdTheme`，后台用 `buildAdminTheme`（§4.2） |
| P0-12 | `apps/web/index.html`、`apps/backend/index.html` | `<head>` | 加 `theme-color` meta + 内联主题脚本（§4.3）；`color-scheme` meta 改 `dark light`；**不要**在 `<html>` 写死 `data-theme` |
| P0-13 | `apps/web/src/main.tsx`、`apps/backend/src/main.tsx` | 全文 | 删掉 `const antdTheme`（含 `Modal` / `Tabs` / `Table` 硬编码）；按 §4.4 拆出 `Shell`；前台传 `buildAntdTheme(mode)`，后台传 `buildAdminTheme(mode)` |
| P0-14 | `apps/web/src/components/ThemeToggle.tsx`、`apps/backend/src/components/ThemeToggle.tsx` | 新建 | 照抄 §4.5 |

### P1 —— 视觉与层级

| # | 文件 | 改法 |
|---|---|---|
| P1-1 | `apps/web/src/components/PostCard.tsx` | 环境 pill 从页脚移到**第一行**（`ml-auto`）；删掉 `style` 里的 `animationDelay`；标题 `font-[650]` → `font-semibold`；未标环境时给 pill 加 `data-missing` |
| P1-2 | `apps/web/src/components/Badges.tsx` | 8 处行内 `rgba(...)` → Tailwind token 类（`text-teal border-teal/30 bg-teal/10` 等）；`ArtifactHint` 去掉彩色，改中性 `text-faint`；`DemoNotice` 同改 |
| P1-3 | `apps/web/src/index.css` 的 `.post-card` | 删 `animation: rise …`；`::after` 背景用 `var(--c-sweep)`，并在 `[data-theme='light']` 下 `opacity: 0`；hover `border-color` 改 `color-mix(in oklab, var(--color-brand) 44%, transparent)` |
| P1-4 | `apps/web/src/index.css` 的 `.navitem.on` / `.top-nav a.on` / `.navgroup-title` / `.chip` | 字距归零；`.on` 状态文字 `var(--c-on-active)`、底色 `color-mix(in oklab, var(--color-brand) 14%, transparent)` |
| P1-5 | `apps/web/src/index.css` 的 `.rail-card` / `.rail-card h3` | 按 §3.1 去卡片化；删除 `h3::before` 竖条 |
| P1-6 | `apps/web/src/index.css` 的 `.banner` | 去掉渐变 / 圆角 / 描边 / `::after`；变 masthead（20px 内距、1px 底边、标题 22/1.4/600、说明 13.5/1.75、`max-width: 46ch`） |
| P1-7 | `apps/web/src/index.css` 的 `.envbar` + `.env-pill` | 按 §3.4 加 `●/○` 标记（`:not([data-missing])` / `[data-missing]` 两套），标签 sans、值 mono |
| P1-8 | `apps/web/src/pages/PostDetailPage.tsx` | `<NarrowPage>` → `<NarrowPage width={680}>`；环境条每一项加 `data-missing` |
| P1-9 | `apps/web/src/index.css` / `apps/backend/src/index.css` 媒体查询 | 替换 §3.2 的三段；`1180` 右栏变跨栏条带，`860` 单列，新增 `640` |
| P1-10 | `apps/web/src/components/Topbar.tsx` | 搜索框后插入 `<ThemeToggle />`；品牌 SVG 的两个渐变 stop 保持硬编码（logo 允许固定色，两个主题都用） |
| P1-11 | `apps/web/src/components/RootLayout.tsx` | Drawer 顶部加一行「外观」+ `<ThemeToggle />` |
| P1-12 | `apps/backend/src/components/AdminLayout.tsx` | 标题行右侧放 `<ThemeToggle size="small" />`；`h1` 22→20；导航 `.on` 的 `text-white!` → `text-[var(--c-on-active)]`；`bg-[rgba(77,107,254,.14)]` → `bg-brand/14`（已实测 Tailwind 4 支持任意百分比透明度） |
| P1-13 | `apps/backend/src/**/*.tsx` | 14 处 `style={{ color: '#F87171' }}` → `className="text-red"`；1 处 `#2DD4BF`（`Badges.tsx`）→ `text-teal`；9 处行内 `rgba(...)` → token 类 |
| P1-14 | `apps/backend/src/pages/admin/*.tsx`（4 个用 Table 的页面） | `Table` 加 `size="small"`；数字列加 `tabular-nums` |
| P1-15 | `apps/backend/src/index.css` | 追加后台密度块：`body { font-size: 13px; line-height: 1.6 }`；`.panel` / `.admin-stat` 圆角 14→10、内距 16/18→12/14；删 `.admin-stat:hover` 的描边高亮 |
| P1-16 | `apps/web/src/components/ArtifactCard.tsx` | 删掉 `· {note}` 的字面中缀，改 `gap` 分隔；「一键装」文案改「怎么用」（它实际只做复制，按钮文案要说真话） |
| P1-17 | `apps/web/src/components/UserAvatar.tsx`（前后台各一份） | `background: oklch(from <c> min(l, 0.55) c h)`，白字最低 4.3:1 |

### P2 —— 可选增强

| # | 改动 | 价值 |
|---|---|---|
| P2-1 | 自托管 JetBrains Mono Latin 子集（400/500，woff2，约 30–60 KB），`@font-face` + `--f-mono` 前置 | 版本号 / patch 辨识度明显提升；不做也不影响正确性 |
| P2-2 | `CategoriesPage` 的主题色输入框旁加两枚预览 chip（暗底 / 亮底各一枚，实时用 §1.4 公式渲染） | 运营改色时立刻看见亮色下是否可读 |
| P2-3 | 详情页环境条点击可复制整串 `dsh 0.2.1-rc.2 / darwin-arm64 / deepseek-v3.2` | 报障时把环境贴给别人是高频动作 |

### 6.4 明确不要做（红线）

- 不换组件库、不手写控件（antd 6 是约束）；不上 CSS-in-JS；不动 Vite / Tailwind 构建。
- 不引入中文 webfont 或图标字体（`lucide-react` 已在用）。
- 不用 `prefers-color-scheme` 自动跟随，除非同时做「跟随系统」第三态。
- 不给「附产物」这类非状态信息造彩色 pill。
- 不改 `packages/shared` 的 `DEFAULT_CATEGORIES` 颜色值（真源是 DB，前端只负责推导显示色）。

---

## 7. 验收清单

实现者交活前自查，逐条打勾：

1. **亮色无残影**：亮色下全局搜索 `#fff` / `text-white` / `#0b0f18` / `#d5deee` / `rgba(77,107,254`，除了 logo 的渐变 stop，不应有命中。
2. **不闪**：DevTools Network 限速 Slow 4G + 勾掉缓存，`localStorage` 设 `light` 后强刷，第一帧就是浅色（录制或用 Performance 面板看首帧截图）。
3. **持久化**：切到浅色 → 刷新 → 仍是浅色；关标签重开 → 仍是浅色；`localStorage.getItem('jingyu.theme') === 'light'`。
4. **antd 跟随**：亮色下打开 Modal / Table / Tabs / Dropdown / Drawer 各一个，无「深底浅字」或「白底白字」。
5. **对比度**（用 DevTools 取色器核）：
   - 正文 `--c-ink` 对面板 ≥ 7:1；`--c-muted` ≥ 4.5:1；`--c-faint` ≥ 4.5:1（两套主题）。
   - 六个板块标签在**两套主题下**都 ≥ 4.5:1。
   - 主按钮白字：暗色 4.99、亮色 5.79。
6. **层级**：卡片第一行能同时看到板块、状态、内核版本；页脚只剩作者、等级、三个计数。
7. **断点**：在 1440 / 1181 / 1180 / 860 / 640 / 375 六个宽度各截一张，右栏不出现「消失」（1180 以下是变条带，不是 `display: none`）。
8. **键盘**：Tab 能到主题开关，focus ring 两套主题都清晰；`prefers-reduced-motion` 下无动画。
9. **两份 CSS 一致**：`diff apps/web/src/index.css apps/backend/src/index.css` 只应存在后台密度块的差异。
10. **动效纪律**：全站只剩三类动效——hover 抬升、光扫（仅暗色）、antd 自身的展开/淡入。没有任何入场动画。

---

## 附：本次设计计划与自我批判的先后顺序（留档）

1. **计划**：读 brief → 确定「实验记录本」这一内容属性 → 出 token（色 / 字 / 布局 / 原则）。
2. **对照**：把计划里每一条问一遍「换成理财社区 / 摄影社区 我还会这么设计吗」——右栏卡片壳、banner hero、逐卡入场、字距标签、5 色 pill 都是「会」，所以改；环境条、版本前置、mono 承担内容、右栏去壳、板块色推导是「不会」，所以留。
3. **动手**：以上第 2 步的结论写进 §5，最终形态见 §1–§4，落地工单见 §6。

---

## 8. 实现反馈修订（两位实现者实测后回的，已采纳）

本节由实现阶段的实测发现汇总，**优先级高于前文正文**。前文保留原样是为了留下决策痕迹，实际施工以本节为准。

### 8.1 antd 6 会注入 unlayered 的锚点 reset（§4.4 漏了）

antd 6 会往页面注入一条 **不在 `@layer` 里**的规则：

```css
:where(.css-xxxx) a { color: var(--ant-color-link); background-color: transparent }
```

unlayered 优先级高于任何 `@layer utilities`，于是**锚点上所有 Tailwind 颜色与底色被压掉**。症状伪装成「Tailwind 没生效」：卡片底色透明、导航选中态 wash 消失。

**正确修法（两个应用统一用这个）**：

```tsx
<AntdApp component={false}>
```

去掉那个带 `css-*` 类名的包裹 div，`:where(.css-…) a` 失去匹配对象，整条规则失效，组件层与 utilities 层一次性恢复。`useApp()` 的 message 仍可用，antd 组件各自带 `css-var-*`，主题变量不受影响。

比逐选择器写 unlayered 兜底彻底 —— 后者每加一个锚点卡片就要补一条。

**唯一残留**：antd 按钮的 focus ring（亮色 `colorPrimaryBorder` ≈ 1.6:1）也是 unlayered，仍需 `!important` 压回 `--c-focus`。

### 8.2 §3.2 的 1180 断点会重叠

左栏 `position: sticky` 挂在 grid item 上时，Chrome 拿整个 grid 容器当包含块，条带进第 2 行后左导航会滑下来盖住「社区规则」（实测 nav bottom 939 / rail top 689）。

**修法**：sticky 移到拉伸后的 `<aside>` 内部 `<nav>` 上。

### 8.3 §7.1 与 §1.5 自相矛盾

§7.1 说「除 logo 不得出现 `#fff`」，§1.5 又说「压品牌填充的白字保留」。加一个 token 解决：

```css
--c-on-fill: #FFFFFF;     /* 品牌实心填充上的字 */
```

需要白字的只有实心品牌按钮/徽章，其余一律用 `--c-ink` / `--c-on-active`。

### 8.4 §2.2 字号阶梯漏了 12.5

§3.1 的 `.rail-card h3` 与 `.t-code` 自己就在用 12.5px。补进阶梯。

### 8.5 46ch 对中文只有约 23 字/行

banner 说明会出现 2 字孤行。**保留 46ch**（对中文已经是上限），补 `text-wrap: pretty`。

### 8.6 §1.4 的板块标签对比度与 §7.5 冲突

§1.4 自己写着作品板块标签在页面底是 4.29，而 §7.5 要求 ≥4.5。**以 §7.5 为准**，实现时把 wash 降到 8% 或把文字色再压深（`oklch` 的 `l` 减 0.03）。

### 8.7 亮色下「语义色压自身 wash」的小徽章普遍不达标

实测（实现阶段）：已解决绿 4.20、已采纳绿 4.27、审核中琥珀 4.39、Lv4 琥珀 4.01、左导航选中 4.20。

根因：这些颜色对纯白有 5.02–5.79，是 wash 把它们拉下去的。

**统一处理**：亮色下这类 wash 压到 **8%**（导航选中 10%），仍不够就压深文字色。目标一律 ≥4.5:1。

### 8.8 antd 主色的实际取值

antd 6 生成的 `--ant-color-primary` 实测是 `#3E55D1`（不是 seed 的 `#4560F2`）。暗色主按钮白字 ≈ 6.2:1，比 §1.3 写的 4.99 更高 —— 结论不受影响，但文案对不上，以实测为准。

---

## 9. 第二轮审查修订（2026-10-22）

> 触发：请助手「用 frontend-design 那套方法把样式再审一遍」。
> 方法：Chrome DevTools MCP 连 5173（前台）/ 5174（后台），暗色 + 亮色各跑一遍**自动对比度审计**——遍历每个含直接文本的节点，取 `color` 与向上回溯到的第一个不透明背景，按 WCAG 算比值；再对 1920 / 1440 / 375 三个宽度截图看行长、触控盒、溢出。

### 9.1 修掉的（都是实测出来的，不是观感）

| # | 位置 | 原值 | 改为 | 实测 |
|---|---|---|---|---|
| 1 | 亮色 `--c-green` | `#15803D` | `#14703A` | 页面底 4.42 → **5.44**（白卡 6.16） |
| 2 | 亮色 `--c-amber` | `#B45309` | `#A84A08` | 4.43 → **5.08**（白卡 5.76） |
| 3 | `.ant-badge` 未读数 | antd 推导 `#D66363` | `#D92C2C` + 白字 | 3.61 → **4.83** |
| 4 | `.avatar` OKLCH 钳制 | `min(l, .55)` | `min(l, .52)` | 青色头像白字 4.33 → **4.86** |
| 5 | `.post-row-kicker` | 12px `faint` | 12px `muted` | 整行灰到「扫读等于没有」 |
| 6 | `.post-row-env b` / `.env-pill` | 11px `faint` / `muted` | 11.5px `ink` / `muted` | 内核版本是「能不能抄」前置条件，不该比点赞数还淡 |
| 7 | `.artifact-strip` | 42 高 + 1px 边框 + 圆角 9 | 42 高、**去边框** + 圆角 9 | 卡片已有描边，条再套框 = 框里画框 |
| 8 | `.copy-artifact-btn` | 72×44、圆角 9 | 桌面 34 高、圆角 9，`pointer: coarse` 下 44 | 按钮和条等高、层次消失 |
| 9 | `.jump-artifacts`（详情页） | 44 高 + 描边 + 7% wash | 44 高、**去描边** + 9% wash、12.5px | 它只是锚点，不是主 CTA |
| 10 | 前台 `formatDateTime` | `2026/10/22 15:59` | `2026年10月22日 15:59` | 清单 §5「日期是人话」 |
| 11 | 后台 `formatDateTime` | `2026/10/22 15:59` | `2026-10-22 15:59` | 密集表格要定长可对齐；**刻意与前台不同**，别强行统一 |

改完复测：暗色 / 亮色**不达标文本节点均为 0**（原暗 1、亮 8，其中 5 条是头像误报）。

### 9.2 查过但决定不动的

- **列表 1920 下摘要 70 字/行**：列表是扫读 + 2 行截断，不是连续阅读；加 `max-width` 反而让三栏右缘不齐。详情页实测 34 字/行，不动。
- **`post-card` / `post-row` 混排（卡间距 11px、行靠发丝线）**：这是信息编码（要动手 vs 读的东西 / 已归档），不是节奏失误。
- **圆角仍有 4/5/6/9/10/14/999**：控件层已收敛到 9（产物条 / 复制按钮 / 跳转按钮 / 输入框 / 导航项），4~6 是徽章与行内 code，14 是内容卡，10 是后台面板。**不再 token 化**——再抽一层 `--r-*` 收益小于维护成本。

### 9.3 留给 lead 拍板的

1. **顶栏 5 个分类是硬编码**（`Topbar.tsx` 的 `TOP_NAV`），左栏是 DB 驱动 7 类。运营在后台新增/改名板块，顶栏不会跟着变，且顶栏缺「作品」「闲聊」。两条路：顶栏也吃 `useCategories()`（超 7 个就折叠），或顶栏只留「首页 / 搜索 / 我的」，分类全交给左栏。
2. **后台操作列每行 3 个描边按钮**（改角色 / 停用 / 重置密码），表格右侧偏重。可降为「主操作按钮 + 「⋯」下拉」，但会多一次点击，取决于后台日均使用频次。

---

## 10. 第三轮：用 impeccable 做终审（2026-10-22）

> 触发：装上 impeccable / huashu-design / interaction-design / gpt-taste / gsap-core / flutter-animating-apps 六个 skill 后，按 impeccable 的 `critique` 流程重审一遍。
> 方法：**dual-agent** —— A 做设计评审（Nielsen 十项 + 花叔六维），B 跑确定性检测器 + 浏览器叠加层（1440×900 / 375×812 × 暗亮 四组合）。两份互不可见，最后合成。
> 结果：Nielsen **27/40**，花叔六维 **6.3/10**（概念 7/10，未触发封顶）。设计特异性判定「介于之间」——产品语义是 DSH 的，视觉母题仍可换到任何开发者社区。

### 10.1 检测器结论

| 通道 | 命中 | 判定后真问题 |
|---|---|---|
| CLI `detect` | 3 条（side-tab ×2、broken-image ×1） | **0 条**（2 条 brief 资产、1 条正则字面量误报） |
| 浏览器叠加层 | 1440 暗 8 / 1440 亮 8 / 375 各 10 | **2 条**（low-contrast、unbalanced-heading） |

**项目级 ignore 已写 5 条**（`side-tab` / `dark-glow` / `bounce-easing` / `overused-font` / `broken-image`，都带理由）。CLI 复跑 **0 条**。
**注意：ignore 只作用于 CLI，浏览器叠加层不读 `.impeccable/config.json`** —— 每次 critique 仍会看到那 6 类，属于预期噪声。

### 10.2 修掉的

| # | 问题 | 原值 → 现值 |
|---|---|---|
| 1 | `.navitem.on > .cnt` 亮色对比度 | faint 压 10% wash = **4.141** → `--c-muted` = **5.377** |
| 2 | `.post-card--settled` 降级过度 | 底 30% → **56%**；标题 muted/500 → **ink/600**；摘要 faint → **muted** |
| 3 | 卡片变体没有日期 | settled / artifact 卡现在也渲染 `<time>`（原来只有 request 变体有） |
| 4 | `.env-pill` 只有裸版本号 | 改「内核 0.2.1」：标签 sans、值 mono，亮色 faint 4.46 的坑一并绕开 |
| 5 | `entryKind()` 顺序 | 产物判断提到 deep-read / lounge 之前 —— 带产物的深读不再退成索引行、产物入口不再消失 |
| 6 | ≤1180 右栏掉到页脚 | 改 `grid-template-areas` 提到信息流上方（原实测 y=2336）；900+ 三列压缩到 ~190px |
| 7 | 错误态缺失 | 新增 `ErrorState` + `LoadMoreError`，6 个页面接 `isError`；404 与 5xx 分开 |
| 8 | 生产把网关 502 当演示数据 | `isOffline` 里 `isBackendNotReady` 只在 DEV 降级；生产交给 ErrorState |

### 10.3 试过但撤掉的

`text-wrap: pretty` 给标题/摘要修移动端孤行 —— **在 `-webkit-line-clamp` 下完全不生效**（实测末行宽度不变），已回滚，避免留下「已修复」的错觉。`unbalanced-heading` 在 clamp 场景目前无解，接受。

### 10.4 这轮没做的（留给下一轮）

1. **首屏保证一个「能直接抄」的对象**：仍是「最新」排序直出，第一条 artifact 卡要滚一屏。可拆「最新 / 能直接抄」两个入口，或在 masthead 下加紧凑产物条。
2. **顶栏导航数据驱动**：`TOP_NAV` 硬编码 5 项，左栏 DB 7 项，缺「作品」「闲聊」。
3. **空态给 action**：FeedPage / SearchPage 的 `EmptyState` 仍没传 `action`。
4. **`.post-row` 整行 `<a>` 的可访问名 378 字符**：应复用卡片的「标题链接 + ::after」结构。

---

## 11. 第四轮：换暗色 + 亮色列表分块（2026-10-22）

> 触发：「暗色的颜色设置还是不太行，换一套风格」+「亮色主题不容易区分列表的每一项」。

### 11.1 诊断

| | 现状 | 问题 |
|---|---|---|
| 暗色 | 底 `#0A0D14`、卡片 `#121826` | 底接近纯黑，卡片只比底亮 **1.09**，东西浮不起来；通体蓝调偏冷 |
| 亮色 | 列表项靠一条 `#E6EBF4` 发丝线分隔 | 那条线压在页面底 `#EDF1F7` 上只有 **1.06** 对比 —— 等于没画 |

### 11.2 暗色：换「深蓝灰」（方案 B）

保留品牌蓝家族，整体上提一档；三选一里另外两套（炭灰 `#15171B`、墨绿 `#101614`）已留档在 `docs/screenshots/review/theme-A/B/C-*.png`。

| Token | 旧 | 新 |
|---|---|---|
| `--c-bg` | `#0A0D14` | `#0F1520` |
| `--c-bg-soft` | `#0E1320` | `#141C29` |
| `--c-panel` | `#121826` | `#182130` |
| `--c-panel-2` / `-3` | `#161E2E` / `#1A2334` | `#1E2939` / `#243146` |
| `--c-line` / `--c-line-soft` | `#1F2839` / `#18202F` | `#2A3547` / `#202B3B` |
| `--c-ink` / `-muted` / `-faint` | `#E8EDF7` / `#9FB0CB` / `#77879F` | `#E9EEF7` / `#A3B2C9` / `#7D8CA3` |
| `--c-code-bg` / `-fg` | `#0B0F18` / `#B9C7DE` | `#121A27` / `#C3D0E4` |

同步改的：两份 `antdTheme.ts` 的 DARK（`colorBgBase/Layout/Container/Elevated`、`colorBorder*`、`colorText*`、`Table.headerBg/rowHoverBg`）、`theme.ts` 的 `theme-color`、两个 `index.html` 的内联防闪脚本与 favicon、后台 `CategoryColorPreview` 的模拟底色。

实测：卡片/底 **1.13**，分隔线 **1.28**，文字 ink **13.89** / muted **7.52** / faint **4.74**，全站对比度审计 **0 处不达标**。

### 11.3 亮色：列表项改成独立白卡

- `--c-line` `#D6DEEE` → `#B9C8DC`，`--c-line-soft` `#E6EBF4` → `#CBD6E6`（页面底 **1.30** / 白卡 **1.47**）
- `[data-theme='light'] .post-row`：白底 + `10px` 圆角 + `8px` 间距 + `1px` 描边 + `0 1px 2px rgba(15,27,45,.04)` 微阴影；hover 描边加深、阴影抬一档
- **`.post-row--archived` 不参与**：已解决且无产物的行保持透明（它已经退成索引，不该占一张卡的分量）
- 形状分级没破：卡片本体仍是 14px 圆角 + 3px 板块条，索引行是 10px 圆角、无边条

**暗色不加这层底** —— 暗底的卡片与页面已经有 1.13 的层次，行再抬一档会碎；两套主题在这里刻意不同构。

### 11.4 留档

三套暗色候选与前后对比图在 `docs/screenshots/review/`：`theme-A-charcoal.png`、`theme-B-slate.png`、`theme-C-forest.png`、`final-dark-1440.png`、`final-light-1440.png`。
