<script>
  import { createEventDispatcher } from 'svelte';
  import { historyStore } from '../stores/historyStore';
  import { mibStore } from '../stores/mibStore';
  import { findMibNameByOid } from '../utils/mibTree';
  import { formatTimestamp } from '../utils/formatting';
  import Icon from '../Icon.svelte';
  import { _ } from 'svelte-i18n';

  const dispatch = createEventDispatcher();

  // Get display name for history entry (MIB name or OID)
  function getHistoryDisplayName(entry) {
    const mibName = findMibNameByOid(entry.oid, $mibStore.tree);
    return mibName || entry.oid;
  }

  // Extract value from history entry results
  function getHistoryValue(entry) {
    if (!entry.results || entry.results.length === 0) return null;

    // For GET/SET/GETNEXT operations, get the first result
    if (entry.operation === 'GET' || entry.operation === 'SET' || entry.operation === 'GETNEXT') {
      const firstResult = entry.results[0];
      if (firstResult?.result?.value !== undefined) {
        return firstResult.result.value;
      }
    }

    // For WALK/GETBULK operations, show count
    if ((entry.operation === 'WALK' || entry.operation === 'GETBULK') && entry.totalResults) {
      return $_('recentHistory.nResults', { values: { count: entry.totalResults } });
    }

    return null;
  }
</script>

<!-- Recent History -->
<div class="history-section">
  <div class="history-header">
    <h4><Icon name="history" size={15} /> {$_('recentHistory.title')}</h4>
    <button class="view-history-btn" on:click={() => dispatch('openHistory')} title={$_('recentHistory.viewAll')}>
      {$_('recentHistory.viewAll')} <Icon name="arrow-right-to-line" size={13} />
    </button>
  </div>
  {#if $historyStore.length > 0}
    <div class="history-list">
      {#each $historyStore.slice(0, 5) as entry}
        <div
          class="history-item op-{entry.operation.toLowerCase()}"
          class:error={!entry.success}
          role="button"
          tabindex="0"
          on:click={() => dispatch('viewInHistory', entry)}
          on:keydown={(e) => e.key === 'Enter' && dispatch('viewInHistory', entry)}
          title={$_('recentHistory.clickToView')}
        >
          <span class="history-date">{formatTimestamp(entry.timestamp)}</span>
          <span class="op-badge">{entry.operation}</span>
          <span class="history-status">{#if entry.success}<Icon name="circle-check" class="icon-success" size={14} />{:else}<Icon name="circle-x" class="icon-error" size={14} />{/if}</span>
          <span class="history-mib-name" title={entry.oid}>{getHistoryDisplayName(entry)}</span>
          {#if getHistoryValue(entry) !== null}
            <span class="history-value">→ {getHistoryValue(entry)}</span>
          {/if}
          <span class="history-meta">
            <span class="history-targets"><Icon name="target" size={12} /> {entry.targets.length}</span>
            {#if entry.duration}
              <span class="history-duration"><Icon name="timer" size={12} /> {entry.duration}ms</span>
            {/if}
          </span>
        </div>
      {/each}
    </div>
  {:else}
    <p class="no-history">{$_('recentHistory.empty')}</p>
  {/if}
</div>

<style>
  .history-section {
    background-color: var(--bg-lighter-color);
    border: 1px solid var(--border-color);
    border-radius: 6px;
    padding: 12px;
    margin-top: 15px;
  }

  .history-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 10px;
    padding-bottom: 8px;
    border-bottom: 1px solid var(--border-color);
  }

  .history-header h4 {
    margin: 0;
    font-size: 1em;
    color: var(--text-color);
  }

  .view-history-btn {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-size: 0.82em;
    color: var(--accent-color);
    background: none;
    border: none;
    cursor: pointer;
    padding: 3px 6px;
    border-radius: 4px;
  }

  .view-history-btn:hover {
    background-color: var(--hover-overlay);
    text-decoration: underline;
  }

  .history-list {
    display: flex;
    flex-direction: column;
    gap: 6px;
    max-height: 250px;
    overflow-y: auto;
  }

  /* Each entry is a single aligned row that wraps on narrow widths; the
     left border carries the operation-type color (var(--op-color)). */
  .history-item {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 4px 10px;
    padding: 7px 10px;
    border-radius: 4px;
    border: 1px solid var(--border-color);
    border-left: 3px solid var(--op-color, var(--border-color));
    background-color: var(--bg-color);
    cursor: pointer;
    transition: background-color 0.15s;
  }

  .history-item:hover {
    background-color: var(--hover-overlay);
  }

  .history-item.error {
    background-color: var(--error-subtle-medium);
  }

  .history-date {
    font-size: 0.8em;
    color: var(--text-muted);
    font-variant-numeric: tabular-nums;
    white-space: nowrap;
  }

  .history-status {
    display: inline-flex;
    align-items: center;
  }

  .history-mib-name {
    font-family: 'Courier New', monospace;
    color: var(--oid-color);
    font-weight: 500;
    font-size: 0.9em;
    background-color: var(--oid-subtle);
    padding: 2px 7px;
    border-radius: 3px;
    max-width: 100%;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .history-value {
    font-weight: 600;
    color: var(--accent-color);
    background-color: var(--accent-subtle);
    padding: 2px 8px;
    border-radius: 3px;
    border: 1px solid var(--accent-border);
    font-size: 0.88em;
    max-width: 320px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .history-meta {
    margin-left: auto;
    display: inline-flex;
    align-items: center;
    gap: 8px;
    font-size: 0.82em;
    flex-wrap: wrap;
  }

  .history-targets,
  .history-duration {
    display: inline-flex;
    align-items: center;
    gap: 3px;
    color: var(--text-dimmed);
    background-color: var(--hover-overlay);
    padding: 2px 6px;
    border-radius: 3px;
    white-space: nowrap;
  }

  .no-history {
    text-align: center;
    color: var(--text-muted);
    font-style: italic;
    padding: 20px;
  }

  /* Scrollbar for history list */
  .history-list::-webkit-scrollbar {
    width: 6px;
  }

  .history-list::-webkit-scrollbar-track {
    background: var(--bg-color);
    border-radius: 3px;
  }

  .history-list::-webkit-scrollbar-thumb {
    background: var(--border-color);
    border-radius: 3px;
  }

  .history-list::-webkit-scrollbar-thumb:hover {
    background: var(--bg-disabled-hover);
  }
</style>
