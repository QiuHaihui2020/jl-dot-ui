# jl-dot-ui

杰理（JL）**单色点阵屏** UI 开发的 Claude Code Skill。

硬约束是单色点阵：图层的 `color_format` 只能是 `OSD1`（设备端 `DC_DATA_FORMAT_MONO`），
软件逐点写 1bpp framebuffer。资源产物是 `JL.sty` / `JL.res` / `JL.str`，ID 头是 `style_jl02.h`。

彩屏（`OSD16`，IMB 硬件合成）是**另一条完全不同的路径**，看
[jl-lcd-ui](https://github.com/QiuHaihui2020/jl-lcd-ui) —— 两套有好几条结论是相反的，
照着点阵那套写彩屏会踩坑，反之亦然。

> **分辨率不限死。** 屏幕尺寸是每个工程自己的（ui-config 的 `Size=宽*高`、
> json 里页节点的 `rect`）。文档里的坐标例子按 128×64 写，换个尺寸照样用，只是数字不同。

---

## 这个 skill 解决什么

这套工具链有**可脚本化的命令行**（`--gen` / `--shot` / `--json-roundtrip`），
所以 **AI 做界面不需要开编辑器** —— 直接编辑 `.uiproj`（就是 JSON），
用命令行验格式、出图看效果，每一步都有退出码或图可判。

skill 补的是命令行没覆盖的部分：

1. **语义体检**。`--json-roundtrip` 只证明"读得进、写得回"，
   ename 重复、slider 零件被改名、子数组键名写错，它全都放行 ——
   这些错要到写应用代码或上板才发现。
2. **坐标从哪来**。复刻既有设备界面时最耗时的不是摆控件，是坐标对不准，
   差 1px 就看得出来。有底图和运行时截图的话这一步可以完全自动算。
3. **属性表不过时**。控件属性从工具实际读的那份控件库导出，不手写 ——
   手写的表一旦过时，AI 会写出"json 语法正确、工具不认"的工程，比没有文档更糟。
4. **把踩过的坑固化下来**，尤其那些不报任何错的。

---

## 安装

作为**项目级** skill（只对某个工程生效）：

```bash
cd <你的 SDK 工程>
mkdir -p .claude/skills
git clone https://github.com/QiuHaihui2020/jl-dot-ui.git .claude/skills/jl-dot-ui
```

作为**个人级** skill（对所有工程生效）：

```bash
git clone https://github.com/QiuHaihui2020/jl-dot-ui.git ~/.claude/skills/jl-dot-ui
```

装好后 Claude Code 会在做点阵屏 UI 相关的活时自动加载，也可以直接 `/jl-dot-ui` 调用。

---

## 目录

```
SKILL.md              主文档：工作循环、八条铁律、控件表
reference/
  widgets.md          选控件：每个控件什么时候用、怎么摆、代码怎么调
  authoring.md        建工程 / 搭结构：从零建一个 UI 工程、弹层怎么做
  layout.md           写 json：树结构、坐标系、类型码规则、不变量清单
  widgets_props.md    查控件属性和取值范围（**自动生成，勿手改**）
  app.md              写代码：事件表注册、控件生命周期、各控件 API
  export.md           导出资源、出图验收、和现有产物比对
tools/
  locate_frames.py    坐标怎么来：底图 vs 运行时截图，算出帧集名 + 精确 rect
  check_project.py    语义体检，有错返回 1
  gen_widgets_ref.py  从控件库重生成 widgets_props.md
```

排一个新界面的顺序：`widgets.md` 选控件 → `authoring.md` 搭结构 →
`layout.md` 落成 json → `app.md` 写回调 → `export.md` 导出。

---

## 工作循环

```
改 .uiproj
   ↓
--json-roundtrip --out x        规范化（见铁律 7），再验就该 rc=0
   ↓
tools/check_project.py          语义体检
   ↓
--shot --no-chrome --zoom 400   出图（不放大只有指甲盖大）
   ↓
读图看排版 ──不对──► 回去改
   ↓ 对了
--gen                           看有没有「认不出控件类型」的警告
   ↓
--gen --run-resbuilder          出资源（要更新固件时才跑，见铁律 6）
```

**每一步都有退出码或图可判，不要凭想象说"应该没问题"。**

> ⚠ **工具的输出是 GBK。** 在 Bash / UTF-8 终端里直接看 `--gen`、`--json-roundtrip`
> 的信息全是乱码，要 `2>&1 | iconv -f GBK -t UTF-8`。退出码是准的，
> 但**警告内容看不懂就等于没看**。

---

## 脚本用法

```bash
# 语义体检：有错误退出码 1
python tools/check_project.py <工程.uiproj>

# 复刻既有界面时先跑这个：底图 vs 运行时截图，算出帧集名 + 精确 rect
python tools/locate_frames.py --help

# 控件库变了就重生成属性表
python tools/gen_widgets_ref.py > reference/widgets_props.md
```

`check_project.py` 拿三个已知正确的工程标定过（都是 0 错 0 疑），
所以**它一旦报错就是真错**，别当噪音跳过。标定原则是宁可漏报也不误报 ——
一个在正确工程上刷屏的检查器等于没有，会被直接忽略。

`locate_frames.py` 的四条局限写在脚本抬头，**用之前看一眼**
（尤其"矢量绘制的东西匹配不上是正常结果，别硬塞"）。

---

## 几条最值钱的结论

摘自 `SKILL.md` 的八条铁律，完整版连同依据看文档本身。

- **`caption` 决定控件类型，不是 `-type`**。slider 的四个零件 `-type` 全是
  `ImageList`，靠 caption 才被认出来。**顺手"整理" caption 会让它静默退化成
  普通控件，不报错。**

- **图层 `color_format` 必须是 `OSD1`**。选了 `OSD16` 不只是跑错路径 ——
  **编辑器预览按点阵的"叠加"画，屏上却是"覆盖"，预览和实机对不上**，极难查。

- **背景填充是三态，不是两态**：空串 = 透明叠加；某个颜色 = 点亮；
  **其它任何颜色 = 擦灭覆盖**。所以控件库的彩色默认值不是"没效果"，等于"擦除"；
  反过来要盖住底图就**必须主动设颜色**，光靠图片盖不住（MONO 画图只写亮点）。
  这一条和彩屏那套是**相反**的。

- **绘制期回调里不能调 UI 更新 API**，要用 `ui_set_call(cb, 0)` 推迟。
  症状是一整页只剩一两个控件，看着完全像资源坏了。
  （真实案例查了很久 `.sty`，逐字段全是对的。）

- **应用层不要用 `REGISTER_UI_EVENT_HANDLER`**。原生 SDK 那套链接段收集在本框架里
  `sec()` 是空宏，照抄会编译过、链接过、**就是不响应**。本框架用显式表
  + `config/ui_port_registry.c` 登记。这一条和彩屏那套也是**相反**的。

- **`--gen --run-resbuilder` 会直接改固件树**，编辑器工具栏的「资源导出」按钮
  走同一条链且点了直接跑。排查"谁改了 `style_jl02.h`"时先想到这个。

---

## 和 jl-lcd-ui 的对照

| | 本 skill（点阵） | jl-lcd-ui（彩屏） |
|---|---|---|
| 图层 `color_format` | 必须 `OSD1` | 必须 `OSD16` |
| 设备端绘制 | 软件逐点写 1bpp framebuffer | IMB 硬件合成任务，每个控件一个 task |
| 事件注册 | 显式表 + `ui_port_registry.c`，**禁用** `REGISTER_UI_EVENT_HANDLER` | **必须**用 `REGISTER_UI_EVENT_HANDLER` |
| 背景填充语义 | 三态：透明 / 点亮 / **擦灭** | 两态：不填充 / 填一块不透明色 |
| 工具 | 有定制 CLI（`--gen`/`--shot`/`--json-roundtrip`） | **只有 GUI**，没有可脚本化的命令行 |

其余（json 树形状、`caption` 决定类型码、坐标相对父节点、控件生命周期、
`ui_set_call` 那个坑）两边是一样的，可以互相参考。

---

## 作者

本 skill 的文档和脚本由 **Claude Opus 5**（Anthropic）编写，
在真实的杰理点阵屏工程上边做界面边整理、逐条验证而成。
