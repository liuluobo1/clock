# Clock32 源码说明

> 源码位置：`H:\deep seek\Clock32\`
> 单文件 C 程序，`clock.c` 约 2540 行、90 KB；产物 `Clock32.exe` 约 170 KB。
> 定位：想改功能、改参数、或者接手维护时看这份。

---

## 一、文件清单

| 文件 | 作用 |
| --- | --- |
| `clock.c` | **全部程序逻辑**（约 2540 行），下面详述 |
| `msvcrt_shim.c` | 78 行。抹平新版 MinGW 对 UCRT 的依赖，提供 `__acrt_iob_func` / `_crt_atexit` 等，**没有它 exe 在 Win7 上起不来** |
| `clock.rc` | 图标 + 版本信息（注意：**必须存成 GBK**，windres 不支持 `\uXXXX`） |
| `clock.ico` | 多尺寸图标（16/20/24/32/40/48/64/128/256） |
| `clock.manifest` | 兼容性/DPI 声明。**注意：没有嵌进 rc**（会与 MinGW 默认 manifest 冲突），DPI 靠代码里 `SetProcessDPIAware()` |
| `make_icon.py` | 生成 ICO（纯标准库） |
| `make_msvcrt_def.py` | 生成 msvcrt 导入库 |
| `check_imports.py` | 校验 exe 的每个导入函数在系统 DLL 中真实存在 |
| `inspect_window.py` | 打印窗口类名/尺寸/exstyle |
| `test_race.py` | 造置顶对手，实测抢 Z 序胜率 |
| `test_tiers.py` | 逐档验证优先级实效 |
| `test_ac.py` | 验证自动配色（造黑/白背景） |
| `regress.py` | 全流程回归（托盘/选档/置顶/缩放/退出） |

---

## 二、编译

工具链在 `C:\dsh_tc32`（i686-w64-mingw32，32 位）。**路径不能带空格**，否则 `ld` 会按空格切分报错。

```powershell
cd "H:\deep seek\Clock32"
$bin="C:\dsh_tc32\bin"

& "$bin\gcc.exe" -c -O2 -Wall -Wextra -municode clock.c -o clock.o
& "$bin\gcc.exe" -c -O2 -Wno-attributes msvcrt_shim.c -o msvcrt_shim.o
& "$bin\windres.exe" -I . clock.rc -o clock.res.o

$la=@('-O2','-municode','-mwindows','-static','-static-libgcc',
      '-specs','C:/dsh_tc32/msvcrt-xp.specs',
      'clock.o','msvcrt_shim.o','clock.res.o','-o','Clock32.exe',
      '-L','C:/dsh_tc32/lib','-lcomdlg32')
& "$bin\gcc.exe" @la
& "$bin\strip.exe" Clock32.exe
```

> PowerShell 里 `--` 开头的参数会被吃掉，所以用 `$la=@(...)` + `& $exe @la` 的数组写法。

**验收**：`python check_imports.py` 必须输出 `RESULT: all imported functions exist in system DLLs`；
`Machine=0x14C`、`Subsystem=2`、导入里**不能出现 `api-ms-win-crt-*`**。

---

## 三、代码地图（按功能分组）

`clock.c` 从上到下依次是：

| 区段（约） | 内容 | 关键函数 |
| --- | --- | --- |
| 40–120 | 常量：定时器 ID、菜单命令 ID、字体/外观/托盘 | `TIMID_*`、`CMD_*`、`FONT_*`、`PAD`、`BORDER_PX` |
| 120–200 | 动态 API 加载（Win7 没有的一律 `GetProcAddress`） | `GetProcOf` |
| 200–300 | 全局状态 + 前向声明 | `g_hwnd`、`g_textColor`、`g_autoColor`、`g_topmost`… |
| 300–360 | 路径与 ini 读写 | `ExeDirPath`、`IniGetInt`、`IniSetInt` |
| 348 / 364 | **配置存取**（新增字段改这里） | `SaveSettings`、`LoadSettings` |
| 420–450 | 档位查表 / 位置钳制 / **时间文本生成** | `PrioInfo`、`ClampWindowPos`、`UpdateClockText` |
| 450–630 | **字体与自适应布局** | `ResolveFace`、`MakeFont`、`CreateFontsAt`、`MeasureLines`、`NaturalSize`、`RefitFontsTo` |
| 636 | 32bpp 预乘 DIB 画布 | `EnsureCanvas` |
| 699 | **渲染核心** | `RenderClock` |
| 797 | 上屏（分层窗口 ULW） | `PaintClock` |
| 860–1030 | **优先级引擎** | `TryEnableRealtimePrivilege`、`DetectElevated`、`OptOutEcoQoS`、`RaiseTimerResolution`、`ApplyUiThreadBoost`、`ApplyPriorityLevel`、`BoostPriorityTemporarily`、`RestoreBoost` |
| 1030–1090 | 前台锁 | `AcquireForegroundLock`、`RevertForegroundLock` |
| 1100–1200 | 自绘菜单：构建与度量 | `AddMenuItem`、`AddMenuSeparator`、`BuildMenu`、`CalculateMenuSize` |
| 1219 | 菜单绘制 | `PaintPopup` |
| 1290 | 弹出菜单（含定位/让光标落在菜单内） | `ShowPopup` |
| 1359 | 菜单翻页（主菜单 ↔ 优先级页） | `SwitchPopupPage` |
| 1418 | 托盘 | `TrayAdd`、`TrayRemove` |
| 1464 | 右下角三角热区判定 | `HitGrip` |
| 1472 | 应用新尺寸（重排字体） | `ApplySize`、`ApplyAutoSize` |
| 1519 | 以管理员身份重启 | `RelaunchElevated` |
| 1546 | **自动配色：采样背景亮度** | `SampleBackgroundLuminance` |
| 1621 | **自动配色：判定并切换** | `ApplyAutoColor` |
| 1648 | 调色盘 | `PickTextColor` |
| 1669 | 菜单命令分发 | `RunMenuCommand` |
| 1750+ | 菜单窗口过程（点击/悬停/自动收起） | `PopupWndProc` |
| 1937 | 判断“别人家的置顶窗口” | `IsForeignTopmost` |
| 1970 | **抢 Z 序线程** | `TopFightThread`、`StartTopFight`、`StopTopFight`、`RestartTopFight` |
| 1955 | 应用置顶 | `ApplyTopmost` |
| 2100+ | 时钟窗口过程（拖动/缩放/滚轮/托盘消息） | `ClockWndProc` |
| 2250 | 定时器回调 | `MainTimerProc`、`BoostTimerProc` |
| 2400 | 入口 | `wWinMain` |

---

## 四、核心数据结构

### 档位表

```c
typedef struct {
    int level;          /* 档位编号（ini 里存的值） */
    DWORD cls;          /* 进程优先级类 */
    int thread;         /* 线程相对优先级 */
    int mmcss;          /* 是否让线程进 MMCSS（能到 26） */
    int harden;         /* A 方案：退 EcoQoS + 0.5ms 计时 */
    int busy;           /* C 方案：抢 Z 序常驻忙等 */
    int admin;          /* 需要管理员才能真正生效 */
    const wchar_t *name;
} PRIO_INFO;
```

`g_prios[]` 里共 11 项。**新增档位只要往数组里加一行**，菜单会自动出现（`BuildMenu` 遍历它）。

### 主要全局变量

| 变量 | 含义 |
| --- | --- |
| `g_hwnd` / `g_hwndPopup` | 时钟窗口 / 菜单窗口 |
| `g_textColor` | 时间颜色（COLORREF） |
| `g_autoColor` / `g_bgLum` | 自动配色开关 / 最近一次采样亮度 |
| `g_showBorder` | 是否画 1px 黑框 |
| `g_w` / `g_h` | 窗口尺寸 |
| `g_clockPx` / `g_pxTime` / `g_pxDate` | 字号与实测字高 |
| `g_topmost` / `g_prioLevel` / `g_autoTier` | 置顶 / 档位 / 是否跟随置顶自动切档 |
| `g_mask[]` | 圆角覆盖率 mask（本轮未用，保留） |
| `g_fightThread` / `g_topFight` | 抢 Z 序线程与其开关 |

---

## 五、五条核心流程

### 1. 启动（`wWinMain`）

```
单实例互斥量 → 动态加载 API → 检测管理员/尝试提权 → 读 ini
→ 创字体 → 算自然尺寸 → 建窗口 → 显示 → 首帧绘制
→ SetTimer(200ms 重绘) / SetTimer(1s 首次自动配色) → 托盘 → ApplyTopmost/ApplyPriority → 消息循环
```

### 2. 每秒重绘

`SetTimer(g_hwnd, TIMID_TICK, 200, MainTimerProc)` ——
**注意定时器传了回调，`WM_TIMER` 不会进窗口过程**，重绘必须写在 `MainTimerProc` 里。

```
UpdateClockText()  取本地时间，文本变了才置脏
→ RenderClock()    在 32bpp 预乘 DIB 上画：覆盖率当 alpha、预乘颜色、1px 黑框、右下角三角
→ PaintClock()     UpdateLayeredWindow 上屏
```

### 3. 右键菜单

```
WM_RBUTTONUP → BoostPriorityTemporarily() → ShowPopup(x,y)
ShowPopup: BuildMenu → CalculateMenuSize → 定位（保证光标落在菜单内）→ 显示 → 60ms 轮询收起
点击 → PopupWndProc 命中项 → RunMenuCommand(cmd)
```

菜单有**两页**：主菜单（`g_menuPage=0`）和优先级页（`g_menuPage=1`），
`SwitchPopupPage()` 原地换页、保持左上角不动。

### 4. 优先级切换（`ApplyPriorityLevel`）

```
设进程类 → （强化档）退 EcoQoS + 0.5ms 计时
→ 决定是否走 MMCSS：pi->mmcss || (实时类 && 没拿到特权)
→ 走 MMCSS 就 ApplyUiThreadBoost()（revert 后立刻重新申请，中间不碰 SetThreadPriority）
→ 否则 SetThreadPriority
→ 写档位参数到 g_tier* → RestartTopFight()（换新线程重新申请 MMCSS）
```

**为什么必须换线程**：`SetPriorityClass` 会按相对优先级重算所有线程基准，把 MMCSS 的 26 打回 15；
而同一线程 `AvRevert` 之后若先调了 `SetThreadPriority`，再 `AvSet` 会返回 0。

### 5. 自动配色（`SampleBackgroundLuminance` + `ApplyAutoColor`）

```
每 30 秒（首次 1 秒）→ 框内 7×5 取点（四周留 2px 避开黑框）
→ 亮度 = (299R + 587G + 114B)/1000
→ 剔掉与当前文字色距离 <70 的点（那是自己的字）；黑框开着时再剔掉近纯黑点
→ 背景点 ≥3 用均值，否则退回全部点中位数
→ 亮度 ≥140 用黑字；≤110 用白字；110~140 保持不动（滞回防抖）
→ 只有颜色真的变了才 SaveSettings + 重绘
```

**原理**：分层窗口 `alpha=0` 的像素在屏幕上显示的就是背后内容，
所以 `GetPixel` 直接拿到背景色，不需要截屏或隐藏窗口。

### 6. 抢 Z 序（`TopFightThread`）

```
1ms 计时精度 → 线程进 MMCSS(26) → 循环：
   SetWindowPos(hwnd, HWND_TOP, ...)      ← 只有这一个动作
   看紧邻上方窗口是不是别人的置顶窗口 → 是就 Sleep(0) 忙等，否则 Sleep(1)
```

**不要**把对手按到 `HWND_BOTTOM`：跨进程改 Z 序一次几百微秒，会把循环频率拖垮（实测 0% 胜率）。

---

## 六、想改东西看这里（常量速查）

| 想改 | 改哪里 |
| --- | --- |
| 字体 | `FONT_FACE` / `FONT_FACE_ALT` / `FONT_FACE_ALT2` / `FONT_FACE_FALLBACK` |
| 字号 | `FONT_PX_TIME`（默认 36）/ `FONT_PX_DATE`（20） |
| 内边距 | `PAD`（8） |
| 黑框粗细 | `BORDER_PX`（1） |
| 右下角三角大小 | `GRIP_TRI`（12）；热区 `GRIP`（16） |
| 最小/最大尺寸 | `MIN_W=100` / `MIN_H=56` / `MAX_W=4000` / `MAX_H=2000` |
| 重绘频率 | `SetTimer(..., TIMID_TICK, 200, ...)` |
| 自动配色间隔 | `SetTimer(hWnd, TIMID_AUTOCOLOR, 30000, NULL)` |
| 自动配色阈值 | `LUM_DARK=110` / `LUM_LIGHT=140` |
| 采样点密度 | `SAMPLE_COLS=7` / `SAMPLE_ROWS=5` |
| 抢 Z 序频率 | `TopFightThread` 里的 `Sleep(0)` / `Sleep(1)` |
| 菜单行高 | `MENU_ROW`（30）/ `MENU_SEP`（9） |
| 版本号 | `clock.rc` 的 `FILEVERSION` + About 文本 + `SaveSettings` 的 `Version` |

---

## 七、踩坑清单（精选）

1. **GDI 画纯黑字 + 透明背景 = 一个字都不写**。必须先白底 `OPAQUE` 画黑字，
   取 `255 - 红通道` 当覆盖率。
2. **覆盖率要直接当 alpha**，把所有有墨像素抬到 255 会让字发糊。
3. **带颜色的字必须预乘**：`RGB = 颜色 × alpha / 255`，否则浅色边缘发白。
4. **`SetTimer` 传了回调就不发 `WM_TIMER`**，重绘写在回调里。
5. **MMCSS 不能在同一条线程上反复申请**（revert 后再 AvSet 返回 0），
   且中间不能碰 `SetThreadPriority`。
6. **`SetPriorityClass` 会重算所有线程基准**，把 26 打回 15 → 必须换新线程。
7. **`NtSetInformationThread(ThreadPriority, 31)` 会被内核拒**（`0xC0000001`）。
8. **抢 Z 序只做 `HWND_TOP`**；按到 `HWND_BOTTOM` 反而更慢更差。
9. **`WS_EX_TOOLWINDOW` 不能一律跳过**（置顶小工具基本都是工具窗口），只按类名排除系统浮层。
10. **托盘菜单不能“光标在菜单外就关”**（光标本来就在任务栏上），要等它先进过菜单。
11. **菜单不要 `SetCapture`**（和任务栏打架会闪）。
12. **退出不能在菜单 wndproc 里 `DestroyWindow` 主窗口**，用 `PostMessage(WM_CLOSE)`。
13. **windres 不支持 `\uXXXX`**，rc 存 GBK。
14. **MinGW 默认 manifest 会冲突**（`.rsrc merge failure`），别往 rc 里塞 manifest。
15. **工具链路径不能带空格**，否则 `ld` 报 `cannot find H:/deep`。

---

## 八、二次开发示例

### 加一个菜单项

```c
/* 1) 定义命令 ID */
#define CMD_MYFEATURE 0x29B

/* 2) BuildMenu() 里加 */
AddMenuItem(L"我的功能", L"说明文字", CMD_MYFEATURE, FALSE, FALSE);

/* 3) RunMenuCommand() 里加分支 */
else if (cmd == CMD_MYFEATURE) { /* ... */ PaintClock(); }
```

### 加一个配置项

```c
/* SaveSettings() */
IniSetInt(L"MyKey", g_myValue);
/* LoadSettings() */
g_myValue = IniGetInt(L"MyKey", 默认值);
```

### 改自动配色为 10 秒

```c
SetTimer(hWnd, TIMID_AUTOCOLOR, 10000, NULL);   /* WM_TIMER 分支里那一处 */
```

### 让自动配色支持自定义阈值

把 `LUM_DARK` / `LUM_LIGHT` 换成全局变量，在 `LoadSettings` 里从 ini 读
`LumDark` / `LumLight` 即可。

---

## 九、已知边界

- 自动配色对**纯色背景**最准；一半黑一半白的复杂背景会落进中间带，**保持上一颜色**不跳动。
- 亮度用的是 Rec.601 加权（未做 sRGB 线性化），极暗颜色会略微偏亮。
- **独占全屏游戏**无法覆盖（任何用户态程序都不行，需要驱动级叠加层）。
- 磨砂/圆角那套是 `FloatingClock32` 实验版的东西，**本程序不涉及**（本程序是纯透明 + 可选黑框）。
