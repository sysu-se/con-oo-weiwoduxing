一、Sudoku / Game 的职责边界

    1.1 Sudoku 负责管理数独盘面本身，核心职责是封装数独领域规则（行 / 列 / 宫冲突校验、固定格判断、完成态 / 合法性校验等），
    保证内部状态不可变（通过深拷贝和返回新实例实现）。
    具体包括：棋盘数据的存储（真私有字段 #grid/#question）、填数操作（guess）、合法性校验（isSafe/isValid）、
    状态克隆（snapshot/clone）、序列化（toJSON）和外表化（toString），对外仅暴露安全的操作接口，禁止直接篡改内部状态。

    1.2 Game 负责游戏流程控制，核心职责是解耦 UI 操作与数独核心规则，管理基于多叉树的撤销 / 重做历史（History 类）。
    不直接操作棋盘内部数据，仅通过 Sudoku 提供的公开方法完成状态更新，对外提供统一的游戏操作入口（guess/undo/redo/clearAnswers 等），
    并封装历史记录的复杂逻辑（分支管理、剪枝、清空等）。

二、Move 是值对象还是实体对象？为什么？

    Move 是值对象。原因：Move 仅包含 row、col、value 三个数据属性，无唯一标识（如 ID），
    不具备独立的生命周期，仅用于传递 “填数操作” 的一次性数据。只要三个属性值完全相同，即可认为是同一个 Move 实例，符合值对象 “以值相等为核心” 的特征；
    同时 Move 仅作为方法入参传递，不被持久化或跟踪状态变化，进一步印证其值对象属性。
    
三、history 中存储的是什么？为什么？

    3.1 迭代后设计（多叉树 History）
        History 类（多叉树实现）中存储的是增量操作快照（包含 row、col、oldValue、newValue），而非完整 Sudoku 快照。原因：
        空间效率优化：相比存储完整 9x9 棋盘快照，增量快照仅记录变更的坐标和数值，大幅降低内存占用；
        多分支回溯支持：多叉树节点（HistoryNode）存储增量快照，可维护多条操作分支，满足数独 “尝试不同填数路径” 的核心场景，而完整快照会导致分支存储成本过高；
        状态恢复准确性：结合 Sudoku 的不可变设计，通过增量快照反向调用 Sudoku.guess 恢复盘面状态，既保证准确性，又避免深拷贝完整棋盘的性能开销。

    3.2 原设计（栈结构）
        原 Game.js 中 history（undoStack/redoStack）存储的是 Sudoku 的深拷贝完整快照。
        原因：早期栈结构仅支持线性撤销 / 重做，完整快照可直接替换当前 Sudoku 实例，实现简单，但空间和性能成本较高。

四、你的复制策略是什么？哪些地方需要深拷贝？
    采用深拷贝 + 不可变实例的核心策略，保证所有状态独立、无引用污染，同时在迭代后优化 “增量更新” 减少不必要的深拷贝。

    4.1 必须深拷贝的场景
        Sudoku 构造函数：对传入的 grid 和 question 进行深拷贝，避免外部数组引用篡改内部状态；
        Sudoku.getGrid ()：返回棋盘的深拷贝数组，防止外部修改返回值影响内部 #grid；
        Sudoku.snapshot/clone：返回深拷贝的新 Sudoku 实例，用于历史快照备份（原设计）、状态隔离；
        Game.undo/redo（原设计）：入栈时存储 Sudoku 深拷贝快照，保证历史状态不被当前操作污染。

    4.2 迭代优化（减少深拷贝）
        增量快照替代完整拷贝：History 类仅存储 row/col/oldValue/newValue 增量数据，仅在恢复状态时通过 Sudoku.guess 生成新实例，避免每次操作深拷贝完整棋盘；
        Sudoku.guess：返回新的 Sudoku 实例（仅对修改行做深拷贝，其余行复用引用），平衡不可变性与性能。

五、序列化 / 反序列化设计

    5.1 序列化（toJSON）
        Sudoku.toJSON：返回包含 grid（当前盘面）、question（题目初始盘面）的纯 JSON 对象，保证内部私有字段（#grid/#question）可序列化；
        Game.toJSON：返回包含 current（当前 Sudoku 的 JSON）、history（History 的 JSON）的对象，覆盖多叉树历史的序列化；
        原 Game.toJSON：返回包含 current、undoStack、redoStack（均为 Sudoku JSON 数组）的对象，适配栈结构历史；
        History.toJSON：空实现（兼容层），聚焦运行时分支管理，暂不实现多叉树持久化。

    5.2 反序列化（createXXXFromJSON）
        createSudokuFromJSON：校验 JSON 结构（9x9 数组），从 grid/question 恢复 Sudoku 实例，保证输入合法性；
        createGameFromJSON：解析 current、history（或 undoStack/redoStack），分别恢复 Sudoku 和 History（或栈结构），重构 Game 实例；
        核心原则：反序列化时做严格结构校验（如棋盘是否为 9x9），抛出明确错误，避免非法数据导致运行异常。

六、外表化接口是什么？为什么这样设计？

    6.1 核心外表化接口
        Sudoku.toString ()：将数独盘面转为带分隔线的可读字符串（使用╔/║/╟等分隔符，・代表空白格），优化原简单字符串格式，更贴合数独视觉结构；
        Game.toString ()：代理 History 的 toString，返回当前历史节点 ID 等关键信息；
        History.toString ()：返回当前节点 ID 的摘要信息。

    6.2 设计原因
        调试友好：toString 输出人类可读的盘面 / 历史状态，便于开发阶段验证数独规则、历史操作是否正确；
        封装性：不暴露内部私有字段（如 #grid、HistoryNode 的结构），仅对外展示可读状态，符合 “最小知识原则”；
        兼容性：保留原 toString 的核心功能（展示盘面），迭代后增强格式可读性，不破坏原有调试习惯。

七、领域对象如何被消费

    7.1 View 层直接消费的是什么？
        View 层直接消费的是根组件 App.svelte 从 Game 领域对象中提取并传递的响应式
        
    7.2 View 层拿到的数据是什么？
        View 层拿到的数据是根组件从 Game 领域对象中提取的纯数据和状态：9×9 棋盘数组 currentGrid、
        能否撤销 / 重做的布尔值 canUndo/canRedo、游戏是否完成 / 合法的布尔值 isCompleted/isValid

    7.3 用户操作如何进入领域对象？
        用户操作 → 子组件派发事件 → 根组件接收并调用 Game 实例的对应方法（guess/undo/redo）→ 操作直接进入领域对象

    7.4 领域对象变化后，Svelte 为什么会更新？
        领域对象变化后，根组件会调用updateState()给响应式变量重新赋值，Svelte 检测到赋值，就自动更新 UI
        
八、响应式机制说明
    8.1 你依赖的是 store、$:、重新赋值，还是其他机制？
        依赖 Svelte 变量重新赋值触发更新，无 store、无 $:
        
    8.2 你的方案中，哪些数据是响应式暴露给 UI 的？
        currentGrid、canUndo、canRedo、isCompleted、isValid
        
    8.3 哪些状态留在领域对象内部？
        原始棋盘、题目、历史记录、固定格、内部校验状态
        
    8.4 如果不用你的方案，而是直接 mutate 内部对象，会出现什么问题？
        UI 不更新、状态混乱、撤销重做失效、破坏封装
        
九、改进说明

    9.1 相比 HW1，你改进了什么？
        （1）历史记录从 “栈结构” 升级为 “多叉树结构”
            HW1：基于 undoStack/redoStack 的线性历史，仅支持 “撤销 - 重做” 的线性操作，无法处理 “尝试不同填数路径” 的场景；
            迭代后：引入 History 类（多叉树），每个节点可包含多个子分支，支持 “分支跳转 / 剪枝”，贴合数独 “试错 - 回溯” 的核心场景。
        （2）Sudoku 的不可变设计强化
            HW1：Sudoku.guess 直接修改内部 grid（mutate），通过 clone 方法生成快照；
            迭代后：Sudoku.guess 返回新的 Sudoku 实例（不可变更新），内部 #grid 改为真私有字段，完全禁止外部篡改。
        （3）状态传递优化：增量快照替代完整快照
            HW1：undo/redo 存储完整 Sudoku 快照，内存占用高；
            迭代后：History 存储增量快照，大幅降低内存开销，同时保证状态恢复准确性。
        （4）职责边界更清晰
            HW1：Game 与 Sudoku 的职责有重叠（如 Game 直接操作 Sudoku 的 grid）；
            迭代后：Game 仅负责流程控制和历史管理，Sudoku 专注领域规则，History 专注多叉树操作，完全解耦。
            
    9.2 为什么 HW1 中的做法不足以支撑真实接入？
        （1）可变状态导致数据不一致：HW1 中 Sudoku 允许直接修改内部 grid，真实场景中多用户 / 多组件操作易导致状态污染，调试和维护成本高；
        （2）完整快照的性能问题：9x9 数独盘面每次操作存储完整快照，内存占用随历史长度线性增长，移动端 / 低性能设备易卡顿；
        （3）封装性不足：HW1 的 Sudoku 未使用真私有字段，外部可直接修改_grid，违反 “封装隐藏实现” 原则，真实接入时易引发未知 bug；
        （4）UI 与领域对象紧耦合：HW1 中 UI 直接调用 Game/Sudoku 的方法，真实项目中若需替换数独规则 / 历史实现，需修改大量 UI 代码，可维护性差。
        
    9.3 你的新设计有哪些 trade-off？
        （1）多叉树 History 的 Trade-off
            优势：支持多分支回溯，贴合数独试错场景，增量快照内存效率高；
            劣势：多叉树节点管理逻辑复杂，开发 / 调试成本高于栈结构；序列化多叉树状态难度大（当前仅支持运行时）。
        （2）不可变 Sudoku 的 Trade-off
            优势：状态安全，无引用污染，调试可追溯；
            劣势：每次 guess 生成新实例，存在一定性能开销（虽经增量优化，但仍高于直接 mutate）；需开发者理解 “不可变” 思维，学习成本略高。
        （3）响应式桥接层的 Trade-off
            优势：UI 与领域对象完全解耦，便于替换 / 扩展领域逻辑；
            劣势：增加 updateState 桥接函数，遗漏调用会导致 UI 与状态不一致；相比 Svelte store 缺少自动订阅机制。
        （4）真私有字段的 Trade-off
            优势：完全禁止外部篡改内部状态，封装性强；
            劣势：旧版 JS 环境兼容性差，序列化 / 反序列化需额外处理（通过 toJSON 暴露数据）。
