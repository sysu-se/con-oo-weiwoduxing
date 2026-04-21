## HW 问题收集

列举在HW 1、HW1.1过程里，你所遇到的2\~3个通过自己学习已经解决的问题，和2\~3个尚未解决的问题与挑战

### 已解决

1. 如何将 Sudoku/Game 领域对象状态同步到 Svelte 界面？
   1. **上下文**：本次 Homework 1.1 核心要求是让 Svelte 界面真正消费 Sudoku/Game 领域对象，UI 渲染的棋盘、合法性、完成度、撤销 / 重做状态必须全部来源于领域对象，而非旧状态；用户操作后领域对象变更，界面必须自动同步刷新。 
   2. **解决手段**：我是新写了一套简化 UI，没有接入现有的 Svelte 游戏流程。
         针对新建UI的同步问题，我在 App.svelte 中创建 Sudoku 与 Game 实例后，直接调用领域对象的公开接口，将初始状态赋值给 Svelte 顶层 let 响应式变量。
        用户的所有操作都仅调用 Game 领域接口，操作执行后统一调用 updateState() 同步最新数据。
        最后利用 Svelte 顶层变量响应式机制，updateState()对 Svelte 顶层 let 变量执行重新赋值，会自动触发 Svelte 的响应式更新，完成界面与领域对象的同步。

2. 数独领域对象不可变更新与快照的实现方式
   1. **上下文**：Sudoku 领域对象需要保证不可变性，guess 操作不能修改原对象，需通过快照生成新实例，保障状态安全。
   2. **解决手段**：Sudoku 类使用 #grid、#question 私有字段封装核心状态，外部无法直接修改内部数据，奠定不可变基础
        实现 snapshot() 方法，对当前棋盘网格进行深拷贝，创建并返回一个全新的 Sudoku 实例，完成状态快照
        guess() 操作不修改原 Sudoku 实例，优先调用 snapshot() 生成快照副本，仅对新副本的网格数据进行修改
        修改完成后返回新的 Sudoku 实例，原实例的所有状态保持不变，严格遵循不可变更新原则
        Game 层基于 Sudoku 的不可变快照实现历史记录管理，为 Undo/Redo 提供稳定的状态支持

### 未解决

1. 如何将 Game/Sudoku 封装为 Svelte custom store 实现自动响应式？

   1. **上下文**：App.svelte 目前依赖手动 updateState 同步状态，无法利用 Svelte store 机制实现领域对象变化后 UI 自动刷新
   2. **尝试解决手段**：后续计划按照作业要求封装 Svelte 自定义 Game Store，将领域对象托管到 store 中实现自动订阅通知，替代手工 updateState ()，让 UI 通过 $ 符号直接响应领域对象的状态变化。

2. 历史对象的序列化契约是空实现，Game 的存档接口不完整

   1. **上下文**：History.toJSON() 固定返回 {}，History.fromJSON() 也直接新建空历史；因此 Game.toJSON() / createGameFromJSON() 看似支持存档，实际上会丢失 undo/redo 分支，接口语义与实现不一致。
   2. **尝试解决手段**：后续计划补全 History 与 Sudoku 的序列化逻辑，实现 toJSON () 方法完整保存历史分支、快照数据；实现 fromJSON () 方法反序列化重建历史结构，同时联动 Sudoku 快照的序列化，保证游戏存档和历史分支可完整恢复。