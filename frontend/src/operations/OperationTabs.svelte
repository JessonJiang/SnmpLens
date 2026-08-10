<script>
  import { _ } from 'svelte-i18n';
  import Icon from '../Icon.svelte';

  export let activeOperation = 'GET'; // bound
  export let snmpVersion = 'v2c';     // GETBULK is disabled on v1

  const TABS = [
    { op: 'GET', icon: 'download', labelKey: 'operations.get' },
    { op: 'SET', icon: 'upload', labelKey: 'operations.set' },
    { op: 'GETNEXT', icon: 'arrow-right-to-line', labelKey: 'operations.getNext' },
    { op: 'GETBULK', icon: 'layers', labelKey: 'operations.getBulk' },
    { op: 'WALK', icon: 'footprints', labelKey: 'operations.walk' },
  ];
</script>

<div class="operation-tabs">
  {#each TABS as tab}
    <button
      class="tab-btn"
      class:active={activeOperation === tab.op}
      on:click={() => activeOperation = tab.op}
      disabled={tab.op === 'GETBULK' && snmpVersion === 'v1'}
      title={tab.op === 'GETBULK' && snmpVersion === 'v1' ? $_('operations.getBulkV1Warning') : ''}
    >
      <Icon name={tab.icon} /> {$_(tab.labelKey)}
    </button>
  {/each}
</div>

<style>
  .operation-tabs {
    display: flex;
    gap: 8px;
    margin-bottom: 15px;
    border-bottom: 2px solid var(--border-color);
  }

  .tab-btn {
    flex: 1;
    padding: 10px 16px;
    background: transparent;
    border: none;
    border-bottom: 3px solid transparent;
    color: var(--text-muted);
    cursor: pointer;
    font-weight: 500;
    font-size: 0.95em;
    transition: all 0.2s;
  }

  .tab-btn:hover {
    background-color: var(--hover-overlay);
    color: var(--text-color);
  }

  .tab-btn.active {
    color: var(--accent-color);
    border-bottom-color: var(--accent-color);
    background-color: var(--accent-subtle-medium);
  }

  .tab-btn:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
</style>
