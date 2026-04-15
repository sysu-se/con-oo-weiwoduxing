<script>
  import { createEventDispatcher } from 'svelte';

  // 父组件传入的 props
  export let currentGrid;   // 9x9 数独数组
  export let isFixed;       // (row, col) => boolean 判断是否是题目固定格

  const dispatch = createEventDispatcher();

  // 当前选中的格子坐标
  let selectedRow = null;
  let selectedCol = null;

  // 点击格子：选中/切换
  function handleCellClick(row, col) {
    if (isFixed(row, col)) return;
    selectedRow = row;
    selectedCol = col;
  }

  // 键盘输入处理
  function handleKeyDown(e) {
    if (selectedRow === null || selectedCol === null) return;

    const key = e.key;
    let value = null;

    // 数字 1-9
    if (/^[1-9]$/.test(key)) {
      value = Number(key);
    }
    // 清空：0 / 回删 / 删除
    else if (key === '0' || key === 'Backspace' || key === 'Delete') {
      value = 0;
    }
    // 非有效按键直接忽略
    else {
      return;
    }

    // 向父组件发送填数事件
    dispatch('guess', {
      row: selectedRow,
      col: selectedCol,
      value
    });
  }

  // 判断是否选中
  function isSelected(row, col) {
    return selectedRow === row && selectedCol === col;
  }
</script>

<!-- 全局键盘监听 -->
<svelte:window on:keydown={handleKeyDown} />

<!-- 数独棋盘容器 -->
<div class="sudoku-board">
  {#each currentGrid as row, rowIdx}
    <div class="sudoku-row">
      {#each row as cell, colIdx}
        <!-- 单元格：区分固定/选中/普通状态 -->
        <div 
          class="sudoku-cell {isFixed(rowIdx, colIdx) ? 'fixed' : ''} {isSelected(rowIdx, colIdx) ? 'selected' : ''}"
          on:click={() => handleCellClick(rowIdx, colIdx)}
          on:keydown={(e) => {
            if (e.key === 'Enter' || e.key === ' ') {
              handleCellClick(rowIdx, colIdx);
            }
          }}
          role="button"
          tabindex="0"
        >
          {cell === 0 ? '' : cell}
        </div>
      {/each}
    </div>
  {/each}
</div>

<style>
  .sudoku-board {
    display: inline-block;
    border: 3px solid #333;
    padding: 5px;
    background: #333;
    outline: none;
  }

  .sudoku-row {
    display: flex;
  }

  .sudoku-cell {
    width: 40px;
    height: 40px;
    line-height: 40px;
    text-align: center;
    background: #fff;
    margin: 1px;
    font-size: 18px;
    cursor: pointer;
    position: relative;
    /* 3×3 宫格右侧分隔 */
    &:nth-child(3n) {
      margin-right: 3px;
    }
  }

  /* 3×3 宫格底部分隔 */
  .sudoku-row:nth-child(3n) .sudoku-cell {
    margin-bottom: 3px;
  }

  /* 固定格子（题目自带数字） */
  .fixed {
    background: #f0f0f0;
    font-weight: bold;
    cursor: not-allowed;
  }

  /* 选中格子（优化：不抖动、不偏移） */
  .selected {
    background: #e3f2fd;
    box-shadow: inset 0 0 0 2px #2196f3;
  }
</style>