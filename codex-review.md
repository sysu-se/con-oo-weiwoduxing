# con-oo-weiwoduxing - Review

## Review 结论

代码已经有明确的 `Sudoku` / `Game` / `History` 分层雏形，也让一套新写的简化 UI 走到了领域接口上；但它没有把领域对象真正接入现有 Svelte 游戏流程，而是绕开原流程重新搭了一个小界面。与此同时，`Sudoku.guess`、`Game.getSudoku`、`History` 序列化等关键设计仍有明显契约问题，因此整体上未达到本次作业对 OOD 和 Svelte 接入的核心要求。

## 总体评价

| 维度 | 评价 |
| --- | --- |
| OOP | fair |
| JS Convention | fair |
| Sudoku Business | poor |
| OOD | fair |

## 缺点

### 1. 领域对象没有接入现有的 Svelte 游戏流程

- 严重程度：core
- 位置：src/App.svelte:21-84
- 原因：当前方案是在根组件里新建了一套简化版 `SudokuBoard/GameControls/StatusDisplay` 流程；而原有棋盘、键盘、菜单、开始新局等组件仍继续依赖 `@sudoku/stores/*` 与 `@sudoku/game`（如 `src/components/Board/index.svelte`、`src/components/Controls/Keyboard.svelte`、`src/components/Header/Dropdown.svelte`）。这不是让现有 View 真正消费 `Game/Sudoku`，而是绕开原流程重写了一套更小的 UI，未满足作业最核心的“真实接入”要求。

### 2. 把“校验能力”实现成“拒绝非法输入”，与现有数独交互语义冲突

- 严重程度：major
- 位置：src/domain/Sudoku.js:123-125
- 原因：`guess` 在有冲突时直接返回 `null`，导致用户无法通过正常输入把棋盘置为 `invalid`。这样 `StatusDisplay` 中的错误分支事实上几乎不可达，也与 starter 里冲突高亮/提示用户修正的业务语义不一致。领域对象更合理的职责应是表达当前局面是否合法，而不是直接阻止不合法局面出现。

### 3. 通过重写实例方法伪造不可变性，破坏对象契约

- 严重程度：major
- 位置：src/domain/Sudoku.js:131-133
- 原因：`guess` 在返回新 `Sudoku` 之前先把当前实例的 `getGrid` 动态改写为闭包返回 `newGrid`。这既与注释中的“原对象不变”相矛盾，也让同一个对象在一次方法调用后改变公开行为，属于明显的 OOP/JS 反模式。

### 4. Svelte 响应式依赖手工 `updateState()`，没有形成可订阅适配层

- 严重程度：major
- 位置：src/App.svelte:25-62
- 原因：UI 通过多份顶层 `let` 手动镜像 `game` 状态，每个命令都必须显式调用 `updateState()` 才会刷新。这个方案虽然能工作，但没有利用 Svelte 3 最自然的 store/custom store 机制，也使渲染更新依赖人为同步，扩展新状态时很容易漏掉。

### 5. 读取接口返回整盘快照，导致 UI 频繁深拷贝且接口粒度不适合视图层

- 严重程度：major
- 位置：src/domain/Game.js:20-22
- 原因：`getSudoku()` 每次都 `snapshot()`，而 `App.svelte` 初始化、`updateState()` 和 `isFixed` 回调都反复调用它；尤其棋盘渲染时每格都会经由 `game.getSudoku().isFixed(row, col)` 重新克隆整个数独。说明 `Game` 没有给 View 提供稳定的只读视图/adapter，接口设计不适合 Svelte 的细粒度渲染。

### 6. 历史对象的序列化契约是空实现，`Game` 的存档接口不完整

- 严重程度：major
- 位置：src/domain/History.js:175-186
- 原因：`History.toJSON()` 固定返回 `{}`，`History.fromJSON()` 也直接新建空历史；因此 `Game.toJSON()` / `createGameFromJSON()` 看似支持存档，实际上会丢失 undo/redo 分支，接口语义与实现不一致。

### 7. Undo/Redo 通过再次调用 `guess` 恢复状态，历史恢复依赖当前校验规则

- 严重程度：major
- 位置：src/domain/Game.js:57-87
- 原因：撤销/重做不是直接恢复历史中的状态，而是把旧值/新值再次送回 `Sudoku.guess`。这让历史回放与当前业务规则强耦合；一旦校验策略或固定格语义变化，历史恢复就可能失败或得到空值，不是稳健的逆操作设计。

### 8. 构造函数和工厂函数没有真正维护 9x9/数值范围不变量

- 严重程度：minor
- 位置：src/domain/Sudoku.js:16-24
- 原因：注释写的是“由工厂函数保证合法结构”，但 `createSudoku` 只是直接 `new Sudoku(input)`；构造函数也未校验尺寸和单元值范围。领域对象把核心不变量留给调用方自觉保证，边界偏弱。

## 优点

### 1. `Sudoku` 对棋盘和题面做了基础封装

- 位置：src/domain/Sudoku.js:6-31
- 原因：使用 `#grid` / `#question` 区分当前局面与题面固定格，并通过 `getGrid()` 返回副本，避免 UI 直接拿到内部二维数组后随手改写。

### 2. 历史结构考虑了分支 redo，而不只是线性栈

- 位置：src/domain/History.js:46-57
- 原因：`children + activeChild` 的建模让“撤销后再走新分支”的语义有落点，设计上比单纯的 undo/redo 栈更接近真实编辑历史。

### 3. 游戏写操作被集中到 `Game` 边界

- 位置：src/domain/Game.js:31-49
- 原因：`guess` 统一处理旧值读取、合法性委托和历史记录，避免把这些逻辑散落到 Svelte 事件处理函数里。

### 4. 新写的简化 UI 至少通过领域接口发起操作

- 位置：src/App.svelte:33-52
- 原因：在当前这套简化界面里，输入、撤销、重做、清空都经过 `game.guess()` / `undo()` / `redo()` / `clearAnswers()`，而不是直接在组件里修改 `currentGrid`。

## 补充说明

- 本次结论仅基于静态阅读，未运行测试，也未在浏览器中实际点击流程验证。
- 关于“没有真正接入现有 Svelte 游戏流程”的判断，基于代码结构静态分析：`App.svelte` 当前只挂载新建的简化组件，而旧棋盘/键盘/菜单/新游戏流程仍直接依赖 `@sudoku/stores/*` 与 `@sudoku/game`。
- 关于“数独业务是否允许用户输入冲突数字”的判断，基于当前代码中仍存在 `isValid`/冲突提示分支以及 starter 现有冲突高亮设计，而非运行时观察。
- 本次只审查了 `src/domain/*` 及其关联的 Svelte 接入文件，没有扩展评价无关目录。
