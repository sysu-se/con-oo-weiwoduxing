<script>
  // 接收外部props
  export let isCompleted; // 是否完成（所有格子填满且合法）
  export let isValid;     // 当前棋盘是否合法（无冲突）
</script>

<div class="status-display">
  {#if isCompleted}
    <div class="status success">✅ 数独完成！所有数字合法</div>
  {:else}
    {#if isValid}
      <div class="status info">ℹ️ 当前棋盘无冲突，请继续填写</div>
    {:else}
      <div class="status error">❌ 当前棋盘存在数字冲突</div>
    {/if}
  {/if}
</div>

<style>
  .status-display {
    margin-bottom: 15px;
    text-align: center;
  }

  .status {
    padding: 8px;
    border-radius: 4px;
    font-weight: bold;
  }

  .success {
    background: #e8f5e9;
    color: #2e7d32;
  }

  .info {
    background: #e3f2fd;
    color: #1976d2;
  }

  .error {
    background: #ffebee;
    color: #c62828;
  }
</style>