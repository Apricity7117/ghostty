# 技术设计：Shift 点击区间选择

对应需求：`prd.md`。基线：上游 `v1.3.1`（commit `332b2ae`）。本文行号均指 v1.3.1 的 `src/Surface.zig`。

## 1. 设计总纲

不新写选区计算，全部复用 v1.3.1 既有链路：

```
Shift + 左键按下
  └─ mouseButtonCallback 的 extend_selection 块（3898）
       ├─ ① 多击防冲突判定（时间 + 距离）        ← FR9 新增距离维度
       ├─ ② 锚点决策 + 重定向 left_click_pin      ← FR1 / FR10 新增
       └─ ③ cursorPosCallback(pos, null)（4616）  ← 原样复用
             └─ dragLeftClickSingle（4883）
                   └─ mouseSelection（4897）      ← 完全不改
```

核心思路：**把「用哪个点当锚点」与「怎么算选区」解耦**。前者是本次新增的决策逻辑，后者沿用既有纯函数。这样矩形选择、60% 阈值、反向选区、跨屏滚动等既有能力自动继承，回归面最小。

## 2. 改动清单

| 文件 | 改动 | 规模 |
| --- | --- | --- |
| `src/config/Config.zig` | 新增 `@"mouse-shift-click-extend": bool = true` 及文档注释，紧邻 `mouse-shift-capture`（952） | ~15 行（含注释） |
| `src/Surface.zig` | `DerivedConfig` 新增字段（314 区）+ init 赋值（392 区） | 2 行 |
| `src/Surface.zig` | 改写 `extend_selection` 块（3898–3935） | ~20 行 |
| `src/Surface.zig` | 新增 `shiftClickIsMultiClick`、`prepareShiftClickAnchor`、`setLeftClickAnchor`、`shiftExtendAnchor` | ~110 行 |
| `src/Surface.zig` | 新增单元测试 | ~120 行 |

macOS / GTK apprt 均无需改动（C6）。

## 3. 数据与状态

**不新增任何状态字段**。锚点直接复用 `Mouse.left_click_pin`（`*terminal.Pin`，tracked pin，随内容滚动自动跟随，对应 D2），配套字段：

- `left_click_screen`：屏幕一致性判定（C4）
- `left_click_xpos`：`mouseSelection` 的 60% 阈值输入，锚点重定向时必须同步改写
- `left_click_ypos`：仅参与下一次 press 的多击距离判定，同步改写以免语义漂移
- `left_click_count`：决定扩展粒度（单字符 / 词 / 行，C3），不改写

## 4. 关键流程

### 4.1 `extend_selection` 块（替换 3898–3935）

```zig
if (button == .left and action == .press) {
    if (mods.shift and
        self.mouse.left_click_count > 0 and
        !shift_capture)
    extend_selection: {
        const pos = try self.rt_surface.getCursorPos();

        // ① 可能是多击（原地快速连点）则让位给多击逻辑
        if (self.shiftClickIsMultiClick(pos)) break :extend_selection;

        // ② 确定锚点；不满足扩展条件时回落到普通点击
        if (!try self.prepareShiftClickAnchor(pos)) break :extend_selection;

        // ③ 复用既有拖动链路完成选区计算
        try self.cursorPosCallback(pos, null);
        return true;
    }
}
```

注意 `click_state[left]` 已在 3884 被置为 `.press`，故 ③ 能进入 `cursorPosCallback` 的选择分支（4751）；且此处 `return true` 不更新 `left_click_time` / `left_click_count`，锚点得以长期保留（D2）。

### 4.2 ① 多击防冲突（FR9 / C2）

v1.3.1 只看时间，导致「单击后 500ms 内 Shift 点远处」失效。改为**时间与距离同时满足**才判为多击：

```zig
/// 判断这次 Shift 点击是否应让位给多击（双击选词 / 三击选行）逻辑。
/// 仅当「距上次点击不超过 mouse_interval」且「与上次点击位置的距离不超过一个
/// cell 宽度」时才成立，与 press 路径（4055）的多击判定标准保持一致。
fn shiftClickIsMultiClick(self: *const Surface, pos: apprt.CursorPos) bool {
    const now = std.time.Instant.now() catch |err| {
        // 取不到时间就保守地当作多击，维持 v1.3.1 的不扩展行为
        log.warn("failed to get time, not extending selection err={}", .{err});
        return true;
    };
    if (now.since(self.mouse.left_click_time) > self.config.mouse_interval) return false;

    const max_distance: f64 = @floatFromInt(self.size.cell.width);
    const distance = @sqrt(
        std.math.pow(f64, pos.x - self.mouse.left_click_xpos, 2) +
            std.math.pow(f64, pos.y - self.mouse.left_click_ypos, 2),
    );
    return distance <= max_distance;
}
```

顺序上此判定必须在锚点重定向**之前**，否则读到的是被改写过的 `left_click_xpos/ypos`。

### 4.3 ② 锚点决策（FR1 / FR10 / D3）

唯一需要持渲染锁的环节，一次加锁内完成读选区与改锚点：

```zig
/// 为 Shift 点击确定并设置锚点。返回 false 表示不应扩展，调用方回落到普通点击。
fn prepareShiftClickAnchor(self: *Surface, pos: apprt.CursorPos) !bool {
    self.renderer_state.mutex.lock();
    defer self.renderer_state.mutex.unlock();

    const t: *terminal.Terminal = self.renderer_state.terminal;
    const screen: *terminal.Screen = t.screens.active;

    // 开关关闭：完全回退 v1.3.1（需已有选区，且锚点固定不重定向）
    if (!self.config.mouse_shift_click_extend) return screen.selection != null;

    if (self.mouse.left_click_pin == null) return false;

    // 锚点与当前屏幕不一致（primary / alternate 切换）时吞掉本次点击，
    // 让 cursorPosCallback 跳过选区，同时保留锚点（C4）。
    if (self.mouse.left_click_screen != t.screens.active_key) return true;

    // 无选区：直接用上次点击落点作锚点（FR1，本需求的核心场景）
    const sel = screen.selection orelse return true;

    // 有选区：就近端点（FR10）
    const click_pin = screen.pages.pin(.{ .viewport = .{
        .x = pos_vp.x,
        .y = pos_vp.y,
    } }) orelse return false;

    const anchor = shiftExtendAnchor(screen, sel, click_pin);
    try self.setLeftClickAnchor(screen, anchor);
    return true;
}
```

`pos_vp` 由 `self.posToViewport(pos.x, pos.y)`（5067）在锁外算好后传入。

### 4.4 就近端点算法（纯函数，可单测）

```zig
const ShiftAnchor = struct {
    pin: terminal.Pin,
    /// 锚点是选区的哪一端，决定 left_click_xpos 落在 cell 的左边缘还是右边缘
    side: enum { left, right },
};

/// 选出 Shift 点击应固定的那一端：移动离点击点较近的端点，保留较远的端点。
/// 与 iTerm2 `beginExtendingSelectionAt:` 语义一致，保证选区只被拉伸不被丢弃。
fn shiftExtendAnchor(
    screen: *const terminal.Screen,
    sel: terminal.Selection,
    click_pin: terminal.Pin,
) ShiftAnchor {
    const tl = sel.topLeft(screen);
    const br = sel.bottomRight(screen);

    // 点击点在选区之前 → 前推首端，保留尾端
    if (click_pin.before(tl)) return .{ .pin = br, .side = .right };
    // 点击点在选区之后 → 延长尾端，保留首端
    if (br.before(click_pin)) return .{ .pin = tl, .side = .left };

    // 点击点落在选区内 → 比较到两端的距离，移动较近的一端
    const d_tl = cellDistance(screen, tl, click_pin);
    const d_br = cellDistance(screen, br, click_pin);
    return if (d_tl <= d_br)
        .{ .pin = br, .side = .right }
    else
        .{ .pin = tl, .side = .left };
}
```

`cellDistance` 用 `screen.pages.pointFromPin(.screen, pin)`（PageList:3980）取全局坐标，按 `y * cols + x` 转为线性索引后取绝对差；这与 iTerm2 的 `VT100GridAbsCoordDistance` 一致。

边界约定：

- 单 cell 选区（`tl == br`）：两条 before 分支自然覆盖点击点在其前 / 其后的情形；点击点恰在该 cell 上时 `d_tl == d_br`，取 `.right`（保留该 cell 并向后延伸），结果与 v1.3.1 的单击行为一致。
- 矩形选择（`isRectangleSelectState`）：仍按线性距离判定锚点，不做列向特判。矩形形态由 `mouseSelection` 依据 `mouse.mods` 自行处理（C5）。

### 4.5 ③ 锚点重定向

```zig
/// 把 left_click_pin 重定向到 anchor，并同步 mouseSelection 依赖的像素位置。
fn setLeftClickAnchor(self: *Surface, screen: *terminal.Screen, anchor: ShiftAnchor) !void {
    const pin = try screen.pages.trackPin(anchor.pin);
    errdefer screen.pages.untrackPin(pin);

    if (self.mouse.left_click_pin) |prev| {
        if (self.renderer_state.terminal.screens.get(self.mouse.left_click_screen)) |s| {
            s.pages.untrackPin(prev);
        }
    }
    self.mouse.left_click_pin = pin;
    self.mouse.left_click_screen = self.renderer_state.terminal.screens.active_key;

    // mouseSelection 的 60% 阈值规则：锚点 cell 要被包含在选区内，
    // 左端锚点必须落在阈值左侧，右端锚点必须落在阈值右侧。
    const cell_w = self.size.cell.width;
    const col_px = self.size.padding.left + anchor.pin.x * cell_w;
    self.mouse.left_click_xpos = @floatFromInt(switch (anchor.side) {
        .left => col_px,
        .right => col_px + cell_w - 1,
    });

    // ypos 仅用于下一次 press 的多击距离判定，取锚点所在行；锚点已滚出视口时
    // 保守地沿用当前点击位置，避免把远处误判成同点连击。
    ...
}
```

untrack / track 顺序遵循 press 路径（4065–4075）的既有写法：先 track 新 pin 再 untrack 旧 pin，`errdefer` 保证异常时不泄漏 tracked pin。

## 5. 时序与并发

- `prepareShiftClickAnchor` 内部加锁，返回后立即释放；`cursorPosCallback` 自己会再次加锁 —— 两次加锁之间选区理论上可被 IO 线程改写，但此时选区只会被终端输出清除，最坏结果是本次扩展基于稍旧的选区，不会崩溃。v1.3.1 原本的 `hasSelection()` + `cursorPosCallback` 也是同样的两段式，风险等价。
- 相比 v1.3.1，加锁次数不变（原本 `hasSelection()` 也要加一次锁）。

## 6. 配置项

```zig
/// 允许 Shift + 单击在当前没有选区时，从上一次点击的位置扩展出选区，
/// 并在已有选区时移动离点击点较近的那一端（类似 iTerm2）。
///
/// 关闭后恢复为：只有已经存在选区时 Shift + 单击才扩展，且始终以上一次
/// 点击的位置为固定锚点。
///
/// 该行为受 `mouse-shift-capture` 约束：当运行中的程序捕获了 Shift 时，
/// Shift + 单击仍会上报给程序而不产生选区。
@"mouse-shift-click-extend": bool = true,
```

`DerivedConfig` 同步新增 `mouse_shift_click_extend: bool`，init 赋值 `config.@"mouse-shift-click-extend"`。

## 7. 测试策略

纯函数优先，交互链路靠既有测试兜底。

| 测试 | 对象 | 要点 |
| --- | --- | --- |
| `shiftExtendAnchor` 点击在选区前 | 纯函数 | 返回 `br` / `.right` |
| `shiftExtendAnchor` 点击在选区后 | 纯函数 | 返回 `tl` / `.left` |
| `shiftExtendAnchor` 点击在选区内偏首 | 纯函数 | 返回 `br` / `.right` |
| `shiftExtendAnchor` 点击在选区内偏尾 | 纯函数 | 返回 `tl` / `.left` |
| `shiftExtendAnchor` 单 cell 选区 | 纯函数 | 前 / 后 / 同 cell 三种情形 |
| `shiftExtendAnchor` 反向选区 | 纯函数 | `start > end` 时经 `topLeft` / `bottomRight` 归一 |
| `mouseSelection` + 锚点 xpos | 纯函数组合 | 左端锚点用 `col_px`、右端锚点用 `col_px + w - 1` 时，锚点 cell 均被包含 |

`shiftClickIsMultiClick` 依赖 `Instant.now()` 与 `self.size`，不做单测，由 AC4 / AC5 手工验证覆盖。

运行方式：`zig build test -Dtest-filter=shiftExtendAnchor`、`-Dtest-filter=mouseSelection`。

## 8. 兼容性与回滚

- **回滚点 1**：`mouse-shift-click-extend = false` 即恢复 v1.3.1 全部行为，无需改代码（D3）。
- **回滚点 2**：改动集中在单文件的四个新函数 + 一处替换，`git revert` 单个提交即可。
- **上游冲突面**：`extend_selection` 块在 main 分支已被重构（`selection_gesture` 抽象、`Instant` 改 `untilNow`），未来若 rebase 到新版本需人工迁移这一块；`Config.zig` 新增项与上游冲突概率低。

## 9. 已知取舍

1. **不采用 iTerm2 的锚点重置**：iTerm2 扩展成功后清空 `findOnPageCursor`，我们保留锚点。保留后连续 Shift 点击行为更连贯，且与 ghostty「pin 长期跟踪」的设计一致。
2. **不迁移到 mouseUp 处理**：iTerm2 在 mouseUp 判定单击，我们保持 mouseDown，从而支持「Shift 按下后继续拖动微调」（FR4）。
3. **矩形选择不做列向就近判定**：iTerm2 也未对此特判，收益低于复杂度。
