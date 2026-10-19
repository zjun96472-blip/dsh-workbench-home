# dsh-workbench-home

个人 DeepSeek Harness 工作台的**「工作台首页 + 试作页」独立包**。装上并重启后左侧多两个入口：

- **「工作台」**：顶部大标题与常驻状态条 + 三个统计（技能数 / 本地插件数 / 阶段数）+ **能力库卡片墙**
  （中文名、图标、一句话说明、`/` 菜单状态、点开看怎么用，带搜索与分类筛选）+ 「下一阶段」三块占位；
- **「试作」**：一页纸的设计样本（深墨蓝底 + 一束光 + 稿纸网格），不接任何数据，用来判断审美方向。

**页面只开这两页**：行 config 的 `pages` 里 `home: true` / `audition: true`，
其余三项（`memory` / `planBook` / `skillLibrary`）显式 `false` —— 关掉的页面**连侧边栏图标都不占**。

主机半边**没有功能**，只做一件事：把 `pages.*` 通过官方 `webserver/index-inject` 注入成
页面全局量 `__DSH_WORKBENCH__`（客户端拿不到 row config，只能走这个官方通道）。

## 装

```bash
dsh plugin --profile <profile> add /abs/path/to/dsh-workbench-home
dsh plugin --profile <profile> add github:<owner>/dsh-workbench-home
```

装完**必须重启一次**：主机插件代码与客户端 bundle 都是启动时读的。

前置：

| 需要什么 | 说明 |
| --- | --- |
| **pnpm** | `dsh plugin add` 内部转发给 pnpm。 |
| Web profile | 客户端半边要 `webServer`（`dsh-web-app`）。没有 web 的 profile 也装得下，只是没有页面。 |
| `$DSH_HOME` | 技能目录按官方口径解析（config 的 `dshHome` > `$DSH_HOME` > `<安装根>/data` > `~/.dsh`），启动日志会打印最终选了哪个。 |
| 技能库（可选） | `$DSH_HOME/skills/`。没有就显示空状态，不报错。 |
| Node | 本包 `dependencies` 为空。 |

## config

```yaml
- id: workbench-home
  name: dsh-workbench-home
  config:
    dshHome: ""
    pages:
      home: true
      memory: false
      audition: true
      planBook: false
      skillLibrary: false
```

**改完 config 要重启才生效。**

## 页面快照

工作台页里「这个技能是干什么的」来自**构建期烘进 `lib/client.js` 的快照**
（客户端读不到文件内容，只能注入）。本包出厂时快照是**空的**（不含任何个人内容），跑一次就灌自己的：

```bash
DSH_HOME=/path/to/data node tools/refresh-snapshots.mjs --skills   # 只刷技能
```

页面会用实时目录名跟快照做体检，数量对不上会提示重跑 —— 不假装信息是最新的。

## 别同时装的东西

### ① 本套其它三个拆分包（**不要同时装**）

`dsh-workbench-memory` / `dsh-plan-book` / `dsh-skill-library` 与本包**共用同一份客户端代码**，
靠各自下发的 `pages` 开关决定注册哪几页。两个一起装会撞在**同一个全局量**上 —— 实测表现（隔离实例，两个包同装）：

- `index.html` 里 `__DSH_WORKBENCH__` 被赋值**两次**（后装的包在后），浏览器按顺序执行 → **后一条生效**，
  先装那个包的 `pages` 全部失效。实测把 `dsh-workbench-memory` 与本包同装时生效的是本包的
  `{"home":true,"memory":false,"audition":true,...}` → 记忆库那一页**一格都不注册、侧边栏入口不出现**。
- 两个客户端 bundle 都会下发，且读**同一份**生效 config → 同一个插槽 id/key 被两个插件**各注册一次**
  （实测：两个 bundle 各注册 4 处，id/key 完全同名：`sidebar.panellist` 与 `main` 的 `workbench-home` / `workbench-audition`）。
  官方 `dsh-client-ui-sidebar` 把 `entriesOfSlot("sidebar.panellist")` 直接列成侧边栏项，
  所以这种重复会变成重复入口；真实浏览器里的最终渲染**我没有实测**（本会话无浏览器控制）。

**要哪个就装哪个**；确实要一次拿全部，用合集形态 `dsh-workbench-personal`。

### ② 原有本地插件 `dsh-workbench-home`

`data/profiles/web/plugins/dsh-workbench-home` 注册的插槽 id/key 与本包**完全同名**
（`workbench-home` / `workbench-memory` / `workbench-audition`），两个都装会撞；
装本包就把补丁层里那两行 `insert: workbench-home` 删掉。

### ③ 与本套 `dsh-skill-library` 的分工

本包的能力库是**工作台页里的一个板块**（还带顶部大标题、统计条、下一阶段）；
`dsh-skill-library` 是**只有技能卡片墙的一页**。两块功能重叠，别同时装。

## 许可

MIT（见 `LICENSE`）。
