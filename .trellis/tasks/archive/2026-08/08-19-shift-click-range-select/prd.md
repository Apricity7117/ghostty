# Shift 点击区间选择（iTerm2 风格）

## 背景

基线：上游 tag `v1.3.1`（commit `332b2ae`），在其上新开分支开发。

用户诉求：终端中长段落文本，希望「先单击起点 → 移动鼠标 → 按住 Shift 点击终点」即可选中整段，无需按住左键拖动。该交互参考 iTerm2。

## 现状调研（基于 v1.3.1 源码）

`src/Surface.zig` 中已存在 Shift 扩展选择的骨架（`mouseButtonCallback` 的 `extend_selection` 块，v1.3.1 约 3898–3935 行）：

- Shift + 左键按下时，若满足条件则直接复用 `cursorPosCallback`，以 `mouse.left_click_pin`（上一次点击落点）为锚点、当前鼠标位置为终点重算选择，并 `return true` 吞掉本次点击（因此锚点不会被刷新，可反复 Shift 点击微调终点）。
- 生效前置条件有三个：`mods.shift`、`mouse.left_click_count > 0`、`!shift_capture`（终端未捕获 Shift），另加块内两条：
  1. `self.hasSelection()` 必须为真；
  2. 距上次点击时间必须大于 `mouse-interval`（避免与双击/三击计数冲突）。

**缺口**：条件 1。v1.3.1 中单击（`left_click_count == 1`）会主动清空选择（约 4103 行 `switch` 的 `1 =>` 分支），此时 `hasSelection()` 为 false，扩展分支被 `break`。结果是 Shift + 点击退化为一次普通新点击：清空选择、把锚点移到新位置；若两次点击间隔小于 `mouse-interval`，还会被计入双击而变成「选词」。

即：**只有先拖出过一段选区，Shift + 点击才能扩展**；用户要的「单击设锚点 → Shift 点击成区间」在 v1.3.1 下不成立。

## iTerm2 实现参考（源码：`/Users/a/Projects/swift/iTerm2`）

核实结论：iTerm2 **并非**靠「已有选区」实现该交互，而是维护了一个独立锚点——查找页游标 `findOnPageCursor`。三条路径分工明确：

| 场景 | 位置 | 行为 |
| --- | --- | --- |
| 普通单击（未拖动、无 Shift） | `PTYMouseHandler.m:619` mouseUp | `mouseHandlerSetFindOnPageCursorCoord:clickCoord` —— 把锚点设到点击处，这一步是整个交互的前提 |
| Shift 单击且**无选区** | `PTYMouseHandler.m:649` mouseUp | 以 `findOnPageCursor` 为起点 `beginSelectionAtAbsCoord` → `moveSelectionEndpointTo:` 点击处 → `endLiveSelection` → **重置锚点** |
| Shift 单击且**有选区** | `PTYMouseHandler.m:355` mouseDown | `beginExtendingSelectionAt:`（`iTermSelection.m:226`），走「就近端点」算法 |

对本任务的关键印证与差异：

1. **需求成立**：iTerm2 确实支持「单击 → Shift 点击成区间」，靠的是单独锚点而非选区。ghostty 的 `left_click_pin` 恰好就是等价物（且是 tracked pin，比 iTerm2 的坐标 + overflow 更健壮），**无需新增状态字段**。
2. **端点语义有分歧**：已有选区时，iTerm2 的 `beginExtendingSelectionAt:` 比较点击点到选区两端的距离，移动**较近的那一端**（点击点落在选区内时同理，落在选区外则按方向决定移动哪端）；ghostty 是**锚点固定**，永远移动非锚点那一端。用户描述的核心场景两者结果一致，差异仅在「已有选区后反复微调」。
3. **触发时机有分歧**：iTerm2 在 mouseUp 处理（因此需要 `!mouseDragged` 判定）；ghostty 在 mouseDown 处理并转成一次 `cursorPosCallback`，天然支持「Shift 按下后继续拖动微调」（对应 FR4）。ghostty 的做法更优，保留。
4. **锚点生命周期有分歧**：iTerm2 扩展成功后重置锚点（下一次 Shift 点击改走「就近端点」路径）；ghostty 扩展时 `return true` 不刷新 `left_click_pin`，锚点长期保留。保留 ghostty 语义（对应 FR3、D2）。
5. **新发现的缺陷（必须修）**：iTerm2 用 macOS 原生 `clickCount` 区分多击，而 macOS 只在**点击位置相近**时才递增 `clickCount`，因此远距离的快速 Shift 点击天然不会被误判为双击。ghostty 的 `extend_selection` 块只判断了时间间隔（`mouse_interval`，默认 500ms），**没有判断距离**。后果：用户单击后在 500ms 内 Shift 点击远处，扩展被跳过，反而落到普通 press 路径并清除选区——即「手快就失效」。ghostty 在 press 路径里本就有距离判断（`distance > max_distance` 则重置多击计数，约 4055 行），扩展路径应复用同一距离标准。


## 目标

让 Shift + 左键点击在「已有单击锚点但当前无选区」的情况下也能从锚点扩展出选区，对齐 iTerm2 行为，且不破坏已有的拖动选择、多击选择与鼠标上报行为。

## 需求

### 功能需求

- FR1：普通左键单击后（未拖动、当前无选区），移动鼠标到另一位置按住 Shift 点击左键，应选中「首次点击落点」到「Shift 点击落点」之间的连续区间。
- FR2：反向选择成立，即 Shift 点击落点在锚点之前时，选区方向自动反转。
- FR3：连续多次 Shift + 点击可反复调整选区，每次按 FR10 的端点规则决定移动哪一端。
- FR4：Shift + 点击后不松开继续拖动，应继续以本次确定的锚点扩展选区（与现有拖动语义一致）。
- FR5：选区端点的字符级归属沿用现有 `mouseSelection` 的 60% 阈值规则，保证与拖动选择结果一致。
- FR6：`copy-on-select` 生效时，Shift + 点击产生的选区在鼠标释放时同样写入选择剪贴板。
- FR9：多击防冲突判定需同时考虑时间与距离——只有当 Shift 点击落点与锚点的像素距离在多击阈值（`max_distance`，即一个 cell 宽度）以内**且**距上次点击不超过 `mouse-interval` 时，才视为多击而放弃扩展；距离超阈值时无论间隔多短都应扩展。
- FR10（就近端点，对齐 iTerm2）：Shift + 点击时的锚点按以下规则确定：
  - 当前**无选区**：锚点为 `mouse.left_click_pin`（上一次点击落点）。
  - 当前**有选区**：比较点击落点到选区 start / end 的距离，取**较远**的一端作为锚点、移动较近的一端。即点击点在选区之后则延长尾端，在选区之前则前推首端，均为选区扩大而非丢弃已选内容。
  - 该规则替代 v1.3.1「锚点恒为 `left_click_pin`」的行为，属于对既有扩展行为的有意变更。

### 兼容性 / 约束

- C1：终端开启鼠标上报且配置允许其捕获 Shift（`mouse-shift-capture`）时，行为不变——Shift + 点击仍上报给应用，不做选择。
- C2：多击（双击选词 / 三击选行）不得被破坏：在原位置快速连续 Shift 点击仍应递增多击计数而非扩展选区（与 FR9 互为边界）。
- C3：上一次点击是双击 / 三击时，Shift + 点击按既有的词 / 行粒度扩展（`dragLeftClickDouble` / `dragLeftClickTriple`），本任务不改变该粒度语义。
- C4：锚点所在屏幕（primary / alternate）与当前活动屏幕不一致时不扩展，且不破坏已有锚点。
- C5：矩形选择修饰键组合（`isRectangleSelectState`）与 Shift 扩展并用时行为需明确，不得崩溃。
- C6：改动集中在共享核心 `src/Surface.zig`，macOS 与 GTK 两端共用，不新增 apprt 层代码。

### 非目标

- 不改动键盘选择模式、搜索选择、URL 点击等其他选择入口。
- 不重构 `mouse` 状态结构或多击计数机制。
- 不向上游提 PR / issue（遵循项目 CLAUDE.md）。

## 已定决策

- D1（已定）：新增配置项 `mouse-shift-click-extend`，布尔值，默认 `true`。关闭时完全回退到 v1.3.1 行为（即仍要求已有选区才扩展）。
  - FR7：配置为 `false` 时，无选区状态下的 Shift + 点击表现与 v1.3.1 完全一致。
  - FR8：配置项需带中文/英文说明文档注释，纳入 `ghostty +show-config` 等既有配置展示链路（沿用 `Config.zig` 既有机制，无需额外适配）。
- D2（已定）：锚点只要其 tracked pin 仍有效即可用，不设时间窗口、不要求锚点位于当前视口内。仅在锚点所在屏幕与当前活动屏幕不一致时不扩展（对应 C4）。
- D3（已定）：采用 iTerm2 的「就近端点」语义（FR10），并与 FR1/FR7 共用同一个 `mouse-shift-click-extend` 开关——关闭时整体回退到 v1.3.1 的「需有选区 + 固定锚点」行为，不再单独增设第二个配置项。

## 验收标准

- [ ] AC1：在 primary 屏幕单击某处，间隔超过 `mouse-interval` 后 Shift + 点击另一处，两点间文本被选中并高亮。
- [ ] AC2：Shift + 点击位置在锚点之前时，选区正确反向覆盖。
- [ ] AC3：已有选区 `[A, B]` 时 Shift + 点击 B 之后的 C，选区变为 `[A, C]`；Shift + 点击 A 之前的 C，选区变为 `[C, B]`（FR10 就近端点）。
- [ ] AC4：单击后**立即**（`mouse-interval` 内）Shift + 点击远处，仍能正确扩展选区（FR9，iTerm2 对照下发现的缺陷）。
- [ ] AC5：在同一位置快速双击仍正常选词、三击仍正常选行（C2 未被 FR9 破坏）。
- [ ] AC6：连续多次 Shift + 点击可持续收放选区，无跳变、无选区丢失。
- [ ] AC7：开启鼠标上报且 `mouse-shift-capture` 允许捕获时，Shift + 点击不产生选区、正常上报。
- [ ] AC8：`mouse-shift-click-extend = false` 时，Shift + 点击的行为（含无选区场景与就近端点）与 v1.3.1 完全一致（回归开关有效）。
- [ ] AC9：滚动使锚点移出当前视口后，Shift + 点击仍能从该锚点扩展出跨屏选区。
- [ ] AC10：`zig build test -Dtest-filter=<相关过滤>` 全部通过，并为新行为补充针对 `mouseSelection` / 端点选择 / 扩展条件的单元测试。
- [ ] AC11：`zig fmt .` 无改动残留；`zig build -Demit-macos-app=false` 编译通过。
- [ ] AC12：手动在 macOS 应用中验证 AC1–AC6。

## 备注

- 关键代码位置（v1.3.1 行号）：`src/Surface.zig:3898` 扩展块、`src/Surface.zig:4055` 多击距离判定、`src/Surface.zig:4103` 多击 switch、`src/Surface.zig:4751` `cursorPosCallback` 选择分支、`src/Surface.zig:4883` `dragLeftClickSingle`、`src/Surface.zig:4897` `mouseSelection`。
- `click_state[left]` 在扩展块之前（约 3884 行）已被置为 `.press`，因此 `cursorPosCallback` 的选择分支可被正常触发——这是现有扩展机制可复用的前提。
- FR10 的实现思路：在进入扩展前把 `mouse.left_click_pin` 重定向到选区较远的一端，随后仍复用既有的 `cursorPosCallback` → `dragLeftClickSingle` 链路，避免另写一套选区计算。需同步处理 tracked pin 的 track / untrack 以及 `left_click_xpos`（`mouseSelection` 的 60% 阈值依赖它），具体方案在 `design.md` 中确定。
- iTerm2 参考源码：`sources/TerminalView/PTYMouseHandler.m`（mouseDown/mouseUp 分派）、`sources/Selection/iTermSelection.m:226`（`beginExtendingSelectionAt:` 就近端点算法）。
