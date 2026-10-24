# 运行与构建（T0Level）

读者设定：从没碰过这个工程的人。三条硬规矩：**每条命令必须本机跑过并把真实输出原样贴进「实测输出」**；跑不了或没跑的显式写「未验证」+ 原因；不许出现想象中的命令与想象中的输出。环境数字（耗时/体积/版本）标清口径（哪一档、是否 minify、是否首次）。

前八节按静态站点做了映射（映射理由在节标题括注）：桌面工程的「打包与启动」→ 静态产物与打开方式；「修改标题/图标」→ 站点标题、首页文案与配图；「发布·分发」→ 部署 `public/`。第九节补上四套视觉预设的配置与切换验证。三个不变量不变：最短可跑路径不断链、判据可机器检、交付面收窄。

本文件所有命令与输出均于 2026-10-24 在本机执行；命令在 Git Bash 中运行，涉及环境变量的 PowerShell 写法在对应位置单独注明。

## 一、前置要求

| 工具 | 版本 | 本机路径 | 为什么必须是它 |
|------|------|---------|----------------|
| Hugo | v0.168.0+extended windows/amd64（BuildDate 2026-10-13） | `/d/Apps/hugo_extended_0.168.0_windows-amd64/hugo`；Git Bash 中 `hugo` 可直接用 | 主题用 SCSS，标准版编不了 |
| Go | go1.27.2 windows/amd64 | `C:\Program Files\Go\bin`；Git Bash 中 `go` 可直接用 | 与 `go.mod` 的 `go` 指令（1.27.0）对齐，否则触发工具链下载 |
| Python | 3.13.15 | `python` 可直接用 | 判据脚本（verify_all 等）用 Python 运行 |

实测版本输出：

```
$ hugo version
hugo v0.168.0-e27a6760c5547237f87d353aebe0580feadb20f4+extended windows/amd64 BuildDate=2026-10-13T16:42:50Z VendorInfo=gohugoio

$ go version
go version go1.27.2 windows/amd64

$ python --version
Python 3.13.15
```

环境变量硬要求：`GOTOOLCHAIN=local`、`GOPROXY=off`。不设的后果：Go 会尝试按 `go.mod` 的工具链指令联网下载匹配版本，离线/内网环境直接失败。

```bash
# Git Bash
export GOTOOLCHAIN=local
export GOPROXY=off
```

```powershell
# PowerShell 等价写法
$env:GOTOOLCHAIN = "local"; $env:GOPROXY = "off"
```

## 二、项目启动（首次构建）

在站点根目录 `dsh-docs/` 执行：

```bash
GOTOOLCHAIN=local GOPROXY=off hugo --cleanDestinationDir
```

实测输出（2026-10-24，本机，非 minify）：

```
Start building sites …
hugo v0.168.0-e27a6760c5547237f87d353aebe0580feadb20f4+extended windows/amd64 BuildDate=2026-10-13T16:42:50Z VendorInfo=gohugoio

WARN  deprecated: css.Sass: libsass was deprecated in Hugo v0.153.0 and will be removed in a future release. Use dartsass instead. See https://gohugo.io/functions/css/sass/#dart-sass

                  │ ZH
──────────────────┼─────
 Pages            │ 111
 Paginator pages  │   0
 Non-page files   │   0
 Static files     │ 35
 Processed images │   0
 Aliases          │   0
 Cleaned          │   0

Total in 1809 ms
```

判据：0 个 `ERROR`；`WARN` 只允许主题侧 `css.Sass: libsass was deprecated` 一条（实测仅有这一条）。

> 注意：在 PowerShell 里 `hugo ... 2>&1` 会把这条 WARN 包装成 `NativeCommandError`（exit code 1 的红字），那是 PowerShell 的 stderr 重定向语义，**不是 hugo 构建失败**；判断成败看 `Pages=` 统计表与有无 `ERROR` 行。

严格发布闸门实测（`--panicOnWarning`）：

```
$ GOTOOLCHAIN=local GOPROXY=off hugo --panicOnWarning --cleanDestinationDir --destination ../.tmp/strict-build
...
ERROR error building site: render: ... css.Sass: libsass was deprecated ...
RC=1
```

结论：在 Hugo 0.168.0 下严格闸门会因主题固有的 libsass 弃用 WARN 失败，这是主题契约，不是内容错误。需要严格闸门时按主题版本口径改用 Hugo 0.166.0；普通构建用 0.168.0 即可。

## 三、本地调试

```bash
GOTOOLCHAIN=local GOPROXY=off hugo server -D --port 1314 --disableFastRender
```

启动后默认监听 `http://localhost:1314/`，`-D` 连草稿页一起渲染，内容变更触发增量重建，`Ctrl+C` 停止。

路由探针（`curl -o /dev/null -w "%{http_code}"` 逐个真实状态码）：

| 请求路径 | 实测状态码 |
|----------|-----------|
| `/` | 200 |
| `/docs/` | 200 |
| `/docs/01-deepseek/` | 200 |
| `/docs/01-deepseek/模型全景/模型参数对比/`（百分号编码后） | 200 |
| `/docs/03-dsh/生态实践/providers-hub实战/`（百分号编码后） | 200 |
| `/docs/02-agent-tools/协议标准/MCP与ACP协议/`（百分号编码后） | 200 |
| `/docs/04-hands-on/` | 200 |
| `/docs/04-hands-on/安装启动/安装与首次启动/`（百分号编码后） | 200 |
| `/docs/04-hands-on/配置与日常/配置插件与日常运维/`（百分号编码后） | 200 |
| `/docs/04-hands-on/排障与验收/排障验收与附录/`（百分号编码后） | 200 |
| `/docs/04-hands-on/JetHub与升级/JetHub安装与升级/`（百分号编码后） | 200 |
| `/offline-search-index.zh.cbf97d7470fe574b97affaf546e92133.json` | 200 |
| `/nope-404-test/` | 404 |

> 坑：中文路径直接交给 `curl` 会得到 404，因为 curl 不替你编码；用浏览器或先做 URL 编码。上面的 404 只用于确认不存在的路径会返回 404。

浏览器实测（Chromium 内核，2026-10-24）：

| 页面 | H1 | 表格 | 代码块 | Mermaid 图块 / 已生成 SVG | 横向溢出 | 控制台错误 |
|------|----|------|--------|---------------------------|----------|------------|
| 首页 | 从 DeepSeek 模型底座，到终端 Agent 工具，再到 DSH。 | 0 | 0 | 0 / 0 | false | 0 |
| 模型发布时间线 | 模型发布时间线：从 DeepSeek LLM 到 V4.1-Flash | 13 | 3 | 2 / 2 | false | 0 |
| dsh CLI 参考 | dsh CLI 参考：启动、转储与插件管理 | 11 | 11 | 1 / 1 | false | 0 |
| MCP 与 ACP | MCP 与 ACP：agent 工具的互操作协议 | 9 | 2 | 1 / 1 | false | 0 |
| providers-hub 实战 | providers-hub 实战：多渠道接入与 Jet Hub | 14 | 7 | 1 / 1 | false | 0 |
| 安装与首次启动 | 安装与首次启动：从 Node 到可用的 DSH Web UI | 5 | 13 | 1 / 1 | false | 0 |
| 配置插件与日常运维 | 配置、插件与日常运维：把 DSH 用成日常工具 | 3 | 14 | 1 / 1 | false | 0 |
| 排障验收与附录 | 排障、验收与附录：症状对照、检查单与速查 | 6 | 4 | 1 / 1 | false | 0 |
| JetHub 安装与升级 | Jet Hub 安装与升级：从 Gitee 源到可用面板 | 7 | 22 | 1 / 1 | false | 0 |

教程篇的图片实测：`static/picture/jet-hub-settings.png` 在页内请求返回 200 / `image/png` / 551,538 字节；强制加载后 `naturalWidth=1416`、`naturalHeight=779`（默认 `loading="lazy"`，隐藏窗口不会自动解码）。

搜索面板实测：

- 在 HTTP 站点搜索 `providers-hub`，返回「providers-hub 实战」「DeepSeek Harness 指南」「Web GUI」等结果。
- 在 HTTP 站点搜索 `allowBuilds`，返回「配置、插件与日常运维」「Jet Hub 安装与升级」等教程篇结果。
- 在 `file://` 打开的文章页搜索 `R1`，返回「公司历程与里程碑」「模型参数对比」「模型发布时间线」等结果——离线搜索索引可用。

## 四、静态产物与打开方式（对应「打包与启动」）

两档构建的实测口径：

| 档位 | 命令 | Pages | Static files | 耗时 | 产物 |
|------|------|-------|--------------|------|------|
| 普通 | `hugo --cleanDestinationDir` | 111 | 35 | 1809 ms | `public/` |
| 发布 | `hugo --cleanDestinationDir --gc --minify` | 111 | 35 | 1864 ms | `public/`（minify） |

发布档产物体积实测（`os.walk` 计数，含所有文件）：

```
files=162 bytes=11610955
index_exists True
search_index ['offline-search-index.zh.cbf97d7470fe574b97affaf546e92133.json']
llms True printdir True
```

`relativeURLs: true` 的实证：

- `public/index.html` 的站内链接是 `./`、`./docs/`、`./docs/01-deepseek/` 这类相对路径；深层文章页的搜索索引是 `../../../../offline-search-index.zh.<hash>.json`。
- 直接用 `file://` 打开 `public/index.html` 与深层文章 `public/docs/01-deepseek/模型全景/模型参数对比/index.html`，页面正常渲染：11 张表、1 个 Mermaid SVG，正文 14,559 字符。
- 页面里的 canonical/alternate 链接仍指向 `baseURL`（`https://dsh-docs.local/`），这是 SEO 缺省值，不影响离线打开。
- 发布产物含 `llms.txt`、`_print/` 与离线搜索索引；整目录 `public/` 即为可分发形态。

## 五、改站点标题与首页文案（对应「修改标题」）

| 要改什么 | 改哪里 | 生效位置 |
|----------|--------|----------|
| 站点名 | `hugo.yaml` 的 `title: &siteTitle` | 标签页、顶栏、页脚 |
| 首页大标题/导语/按钮 | `data/home/zh.yaml` 的 `hero` | 首页 |
| 首页卡片文案 / CTA | 同文件 `cards.items` / `cta` | 首页 |
| 文档总览卡片 | `content/docs/_index.md` 的链接列表 + `{.cards}` | `/docs/` |
| 侧边栏分类图标与描述 | 各一级分类 `_index.md` 的 `icon` / `description` | 侧边栏与分类页 |

常见半改：只改 `hugo.yaml` 不改 `data/home/zh.yaml`；或只改一级分类 `_index.md` 不改 `content/docs/_index.md` 的卡片。两处都要改。

实测：首页卡片「认识 DeepSeek」的描述已从 V3.2 更新为 V4.1；`hugo` 重建后首页文本命中「从 Coder 到 V4.1 的模型发布时间线与参数对比」。

## 六、改图标与配图（对应「修改图标」）

| 对象 | 位置 | 说明 |
|------|------|------|
| 站点 logo | `hugo.yaml` 的 `params.logo` → `assets/icons/logo.svg` | 顶栏/侧栏品牌区 |
| favicon | `static/favicon.svg` | 浏览器标签 |
| 分类图标 | 各一级分类 `_index.md` 的 `icon` 字段 | Font Awesome 类名 |
| 正文配图 | `static/picture/` | 正文用 `/picture/...` 引用；缺失会被审计报 `BROKEN_IMAGE` |

实测坑：主题内置的 Font Awesome 副本没有 `fa-whale`，首页「认识 DeepSeek」卡片因此显示为空方块。已改用 `fa-solid fa-water`（矩阵、分类 `_index.md`、`data/home/zh.yaml` 三处同步）。浏览器读回确认四张卡片图标依次为 `fa-water`、`fa-terminal`、`fa-rocket`、`fa-screwdriver-wrench`。

教程篇带一张实拍图：`static/picture/jet-hub-settings.png`（551,538 字节），`picture_map` 实测 `articles=16 refs=1 mapped=1 missing_in_static=0`。其余正文图表仍用 Mermaid；要再加图片时，先落到 `static/picture/`，再在正文用 `/picture/<文件名>` 引用，最后跑一次 `audit_structure.py` 与 `picture_map.py`。

## 七、发布与分发

发布档命令：

```bash
GOTOOLCHAIN=local GOPROXY=off hugo --cleanDestinationDir --gc --minify
```

必须随产物分发的文件：整目录 `public/`（含离线搜索索引 `offline-search-index.zh.<hash>.json`、`llms.txt` 与 `_print/`）。子路径部署不需要改 `baseURL`：`relativeURLs: true` 让产物自带相对路径。未做真实部署（没有目标环境），标「未验证」。

严格闸门提醒：若 CI 使用 `--panicOnWarning`，在 Hugo 0.168.0 下会因主题 libsass WARN 失败；按主题契约改 Hugo 0.166.0，或不要把该 WARN 当内容错误处理。

## 八、注意事项与判据

| 判据 | 实测结果（2026-10-24） |
|------|------------------------|
| `verify_all.py --selftest` | 末行 `SELFTEST_PASS`；20 个红样本逐条报出、5 个合规样本零误报、22 类判据可达 |
| `audit_structure.py`（rewrite） | `articles=16 expected=16 dirs=17 problems=0` |
| `matrix_check.py` | `categories=4 sections=12 articles=16 source_refs=17 on_disk=17 problems=0`；三类查重对均为 0 |
| `check_links.py` | `links_checked=101 broken=0 unmapped=0 home_urls=8` |
| `check_coverage.py` | `rows=17 error_rows=0 triage_below_80pct=8`；教程四篇均 100%，低覆盖项是去重后的预期结果，人工判定见下 |
| `picture_map.py` | `articles=16 refs=1 mapped=1 missing_in_static=0` |
| `validate_mermaid.py` | 解析台 `TOTAL=21 REAL_FAIL=0`、`HEADLINE=GREEN`；浏览器 DOM 读回教程四篇与 03-dsh 代表页均为「图块数 = 已生成 SVG 数」 |
| `hugo build` | `Pages=111 Static files=35 ERROR=0`，普通档 1809 ms / 发布档 1864 ms |
| 视觉预设（4 套） | `preset: paper` + `preset_menu` 实值 4 项；产物 `data-td-preset=paper`、12 个单选项、首帧白名单 4 项；浏览器逐项切换与刷新持久化通过（详见第九节） |
| 严格闸门 `--panicOnWarning` | `RC=1`，原因是主题 libsass 弃用 WARN（主题契约，非内容错误） |

`check_coverage` 的 8 条低覆盖是**线索不是结论**，逐条判定如下：

| 低覆盖项 | 人工判定 |
|----------|----------|
| `dsh:05-webui.md` → WebGUI（49%） | 启动命令、公告 URL、DSH_HOME 基础表与 7 条 patch 已按要求迁到教程篇与《Cordis 插件体系》，本篇只留会话变量、深层落盘、凭据与排障；不是事实缺口 |
| `dsh:03-arch.md` → DSH 是什么（55%） | 漏项为 `Architecture`/`DumpConfigInvocation`/`Flag` 等英文标题与类型名；DSH_HOME 与插件安装已迁到教程篇，其余细节按主题拆在《CLI 参考》《Cordis 插件体系》 |
| `dsh:04-ecosystem.md` → providers-hub（60%） | 安装、`allowBuilds`、旧包名排障已迁到《Jet Hub 安装与升级》；本篇保留 17 个渠道、账号池、积分、聚合路由与网关等独有内容 |
| `dsh:01-overview.md` → DSH 是什么（71%） | 漏项为 `copy`/`custom`/`each` 等英文说明词与拆到教程/CLI 篇的入口细节 |
| `dsh:02-cli.md` → CLI 参考（73%） | 插件安装示例已迁到教程篇；帮助原文、七个 flag、dump 三兄弟与 invocation 语义完整保留 |
| `web:openclaw.md` → OpenClaw（77%） | 漏项为来源 URL 与「事实由…」类核查说明，不是事实缺口 |
| `web:protocols.md` → MCP 与 ACP（79%） | 漏项为来源 URL 与核查说明 |
| `web:deepseek-opensource.md` → 开源策略（80%） | 漏项为来源 URL 与「供正文论证用」等说明词 |
| `handbook:*` → 教程四篇（100%） | 源行逐条命中；README 的 1,052 行按四篇 1:1 落位，仅改写跨页链接、`FAQ` 标题与一处树形字符 |

内容口径：本站为原创撰写，素材源为 17 份本地一手材料（不随站点交付），目标为 16 篇重写文档，合并倍率 1.06 篇源文档 / 篇正文。其中 `04-hands-on` 四篇由根目录 README 按主题 1:1 切分，README 源行逐条核对无缺失；教程四篇与 `03-dsh` 五篇的精确段落交集为 0，重复的操作步骤只在教程篇保留，概念篇以链接指回。逐篇归属、字节数与查重结论见 `doc/复刻覆盖矩阵.md`。本站没有需要照抄的上游缺陷清单。

## 九、视觉预设与外观切换（4 套，默认 paper）

站点默认使用 `paper`（暖纸面），读者可在页面 Appearance 面板切换四套视觉预设。配置只有两行：

```yaml
    preset: paper                              # 站点默认观感
    preset_menu: [paper, slate, ink, terminal] # 四套全开；列表必须含默认 preset
```

`preset_menu` 必须是实值，不能是注释：只有 `dark_mode: true` 时面板仅带明暗两挡，不带预设。预设体系是 OINK 1.2+ 能力，配在 1.0/1.1 主题副本上构建不报错但不生效，所以判据必须落在产物层。

主题副本核对（2026-10-24）：

```
find dsh-docs/oink-main/oink-main -type f | wc -l   → 1273
diff -rq dsh-docs/oink-main/oink-main <技能>/assets/theme/oink-main/oink-main → 无差异
theme.toml: min_version = "0.166.0"
关键文件：assets/scss/td/_presets.scss、_experimental-presets.scss、layouts/_partials/preset-config.html、assets/js/appearance.js 均存在
```

构建产物核对：

```
<html … data-td-preset=paper data-td-site-preset=paper …>
data-td-preset-value 出现 12 次（导航栏 / 页脚 / 移动端抽屉各 4 项）
首帧白名单：s=["paper","slate","ink","terminal"]
CSS 选择器：[data-td-preset='paper'|'slate'|'ink'|'terminal'] 四套都在
```

浏览器实测（`http://127.0.0.1:8767/`，Chromium 内核，先设 `td-color-theme=light`，逐项点击后读计算样式）：

| 预设 | `html[data-td-preset]` | 页面背景 | 正文色 |
|------|------------------------|----------|--------|
| paper（默认） | `paper` | `rgb(247, 246, 243)` / `#f7f6f3` | `rgb(33, 32, 28)` |
| slate | `slate` | `rgb(241, 244, 248)` / `#f1f4f8` | `rgb(22, 34, 46)` |
| ink | `ink` | `rgb(255, 255, 255)` / `#ffffff` | `rgb(20, 20, 20)` |
| terminal | `terminal` | `rgb(244, 245, 242)` / `#f4f5f2` | `rgb(29, 33, 31)` |

暗色模式同一轮测得页面背景：paper `#161513`、slate `#0b1119`、ink `#0b0b0b`、terminal `#0c0f0e`，与主题画布令牌一致。

选择持久化实测：

- 点击 `terminal` → `localStorage['td-preset']="terminal"`；刷新后 `<html data-td-preset="terminal">`，同时 `data-td-site-preset="paper"` 保持不变（站点默认未被覆盖）。
- 点击回 `paper` → 存储键被删除、`data-td-preset` 回到 `paper`；清空 localStorage 后刷新，默认仍是 `paper`，三个入口的单选均选中 paper。
- 明暗模式独立于预设：`td-color-theme` 只影响 `data-bs-theme`，不改变预设名。

未做逐页人眼核对：`ink` / `terminal` 是主题标注的实验预设，本轮验证的是配置、产物、四套切换、持久化与画布色，没有逐篇检查长表格与代码块在两套实验配色下的可读性。

## 未验证项汇总

| 项 | 状态 | 原因 |
|----|------|------|
| 真实部署（独立域名 / 子路径） | 未验证 | 没有目标环境；产物层已用 `file://` 与本地 HTTP 服务验证 |
| 逐页截图与多浏览器回归 | 部分验证 | 已做首页截图 + 4 个代表页 DOM 断言（Chromium 内核）；未在 Firefox/Safari 逐页跑 |
| PowerShell 下逐条命令复跑 | 未逐条复跑 | 命令以 Git Bash 实测；PowerShell 仅给出环境变量等价写法与 stderr 重定向注意项 |
| CI 严格闸门 | 已验证会失败 | Hugo 0.168.0 + 主题 1.3.2 的 libsass WARN 会致败，属主题契约；未切 0.166.0 复跑绿路 |
| ink / terminal 逐页可读性 | 未验证 | 四套切换、持久化与画布色已实测；未逐篇人眼核对长表格/代码块在两套实验配色下的观感 |
