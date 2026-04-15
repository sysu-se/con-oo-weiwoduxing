<script>
  import { createSudoku } from './domain/Sudoku.js';
  import { createGame } from './domain/Game.js';
  import SudokuBoard from './components/SudokuBoard.svelte';
  import GameControls from './components/GameControls.svelte';
  import StatusDisplay from './components/StatusDisplay.svelte';

  // 1. 初始化数独题目
  const initialGrid = [
    [7, 0, 0, 0, 0, 9, 0, 0, 0],
    [0, 3, 0, 7, 0, 0, 4, 2, 0],
    [0, 0, 0, 8, 6, 0, 1, 9, 0],
    [0, 0, 0, 0, 0, 0, 8, 0, 9],
    [9, 7, 0, 0, 2, 0, 0, 6, 3],
    [1, 0, 3, 0, 0, 0, 0, 0, 0],
    [0, 4, 5, 0, 1, 2, 0, 0, 0],
    [0, 9, 0, 0, 0, 4, 0, 5, 0],
    [0, 0, 0, 3, 0, 0, 0, 0, 4]
  ];

  // 2. 创建核心实例（映射 Game.js 逻辑）
  const sudoku = createSudoku(initialGrid);
  const game = createGame({ sudoku });

  // 3. 响应式状态（同步游戏实例的当前状态）
  let currentGrid = game.getSudoku().getGrid();
  let canUndo = game.canUndo();
  let canRedo = game.canRedo();
  let isCompleted = game.getSudoku().isCompleted();
  let isValid = game.getSudoku().isValid();

  // 4. 核心方法：转发游戏操作（映射 Game.guess/undo/redo）
  function handleGuess(move) {
    const success = game.guess(move);
    if (success) {
      updateState(); // 操作成功后更新UI状态
    }
  }

  function handleUndo() {
    game.undo();
    updateState();
  }

  function handleRedo() {
    game.redo();
    updateState();
  }

  function handleClearAnswers() {
    game.clearAnswers();
    updateState();
  }

  // 5. 状态同步工具函数（确保UI与游戏实例一致）
  function updateState() {
    currentGrid = game.getSudoku().getGrid();
    canUndo = game.canUndo();
    canRedo = game.canRedo();
    isCompleted = game.getSudoku().isCompleted();
    isValid = game.getSudoku().isValid();
  }
</script>

<div class="app-container">
  <h1>数独游戏</h1>
  <!-- 状态展示组件 -->
  <StatusDisplay {isCompleted} {isValid} />
  
  <!-- 棋盘渲染+交互组件 -->
  <SudokuBoard 
    {currentGrid} 
    on:guess={({ detail }) => handleGuess(detail)} 
    isFixed={(row, col) => game.getSudoku().isFixed(row, col)}
  />
  
  <!-- 游戏控制组件 -->
  <GameControls 
    {canUndo} 
    {canRedo} 
    on:undo={handleUndo} 
    on:redo={handleRedo} 
    on:clearAnswers={handleClearAnswers}
  />
</div>

<style>
  .app-container {
    max-width: 600px;
    margin: 0 auto;
    padding: 20px;
    font-family: Arial, sans-serif;
  }
  h1 {
    text-align: center;
    color: #333;
  }
</style>

<style global>
	@import "./styles/global.css";
</style>
