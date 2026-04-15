<script>
  import { createEventDispatcher } from 'svelte';

  // 1. 接收外部props
  export let canUndo;   // 是否可撤销（映射 History.canUndo）
  export let canRedo;   // 是否可重做（映射 History.canRedo）

  // 2. 事件分发
  const dispatch = createEventDispatcher();

  // 3. 按钮点击事件
  function undo() {
    dispatch('undo');
  }

  function redo() {
    dispatch('redo');
  }

  function clearAnswers() {
    dispatch('clearAnswers');
  }
</script>

<div class="game-controls">
  <button 
    class="control-btn" 
    on:click={undo} 
    disabled={!canUndo}
  >
    撤销 (Undo)
  </button>
  
  <button 
    class="control-btn" 
    on:click={redo} 
    disabled={!canRedo}
  >
    重做 (Redo)
  </button>
  
  <button 
    class="control-btn clear-btn" 
    on:click={clearAnswers}
  >
    清空答案
  </button>
</div>

<style>
  .game-controls {
    margin-top: 20px;
    display: flex;
    gap: 10px;
    justify-content: center;
  }

  .control-btn {
    padding: 8px 16px;
    border: none;
    border-radius: 4px;
    background: #2196f3;
    color: white;
    font-size: 14px;
    cursor: pointer;
    &:disabled {
      background: #cccccc;
      cursor: not-allowed;
    }
  }

  .clear-btn {
    background: #f44336;
  }
</style>