<script>
  import { createEventDispatcher } from 'svelte';
  import Icon from '../Icon.svelte';
  import ContextMenu from '../ContextMenu.svelte';
  import ResultsComparison from './ResultsComparison.svelte';
  import { copyToClipboard, copyRich } from '../utils/clipboard';
  import { escapeCSV, downloadFile } from '../utils/csv';
  import { formatValueWithEnum as _formatValueWithEnum, findTableParentNode } from '../utils/mibTree';
  import { notificationStore } from '../stores/notifications';
  import { _ } from 'svelte-i18n';
  import { get } from 'svelte/store';
  import { anonMode, anonymizeIp } from '../utils/anonymize';

  const dispatch = createEventDispatcher();

  /** @type {Array} */
  export let bulkResults = [];

  /** @type {string} */
  export let activeOperation = 'GET';

  /** @type {object|null} */
  export let selectedNode = null;

  /** @type {object} */
  export let oidInfoCache = {};

  /** @type {Array} */
  export let mibTree = [];

  // Internal state
  let tableViewEnabled = false;
  let sortColumn = null;
  let sortAscending = true;

  // The default view (raw/table) is decided exactly once per result set. This
  // guard is what stops the view from flipping — and the filter input from
  // losing focus — while the user is interacting with the results.
  let autoViewApplied = false;
  let lastResultsForView = null;

  // Raw WALK/GETBULK list sorting (clickable OID/Type/Value headers).
  let rawSortKey = null; // 'oid' | 'type' | 'value'
  let rawSortAsc = true;

  // Table cell context menu
  let cellMenu = { visible: false, x: 0, y: 0, items: [] };
  let cellMenuCtx = null;
  let comparisonViewEnabled = false;
  let compareEnabled = false;
  let walkFilter = '';


  /**
   * Filter WALK items by text or regex against OID, name, type, and value.
   */
  function filterWalkItems(items, filterText) {
    const query = (filterText || '').trim();
    if (!query) return items;
    let test;
    try {
      const re = new RegExp(query, 'i');
      test = (str) => re.test(str);
    } catch {
      const lower = query.toLowerCase();
      test = (str) => str.toLowerCase().includes(lower);
    }
    return items.filter(item => {
      const name = oidInfoCache[item.oid]?.name || '';
      return test(item.oid) || test(name) || test(item.type) || test(String(item.value));
    });
  }

  // Compare two OIDs numerically, segment by segment (so 1.2 < 1.10).
  function compareOids(a, b) {
    const pa = String(a).replace(/^\./, '').split('.').map(Number);
    const pb = String(b).replace(/^\./, '').split('.').map(Number);
    const n = Math.max(pa.length, pb.length);
    for (let i = 0; i < n; i++) {
      const x = pa[i] ?? -1;
      const y = pb[i] ?? -1;
      if (x !== y) return x - y;
    }
    return 0;
  }

  // Sort raw WALK/GETBULK items by a column (kept out of buildTableData so it
  // doesn't affect the reconstructed MIB-table view).
  function sortWalkItems(items, key, asc) {
    if (!key) return items;
    const arr = [...items];
    arr.sort((a, b) => {
      let cmp;
      if (key === 'oid') {
        cmp = compareOids(a.oid, b.oid);
      } else {
        const av = key === 'type' ? a.type : a.value;
        const bv = key === 'type' ? b.type : b.value;
        const an = Number(av);
        const bn = Number(bv);
        cmp = (!isNaN(an) && !isNaN(bn)) ? an - bn : String(av ?? '').localeCompare(String(bv ?? ''));
      }
      return asc ? cmp : -cmp;
    });
    return arr;
  }

  function sortRaw(key) {
    if (rawSortKey === key) {
      rawSortAsc = !rawSortAsc;
    } else {
      rawSortKey = key;
      rawSortAsc = true;
    }
  }

  // Filter reconstructed table ROWS: keep a row if the query matches its index or
  // any cell's value / column name / OID (same query as the raw-view filter).
  function filterTableRows(rows, columns, query) {
    const q = (query || '').trim();
    if (!q) return rows;
    let test;
    try {
      const re = new RegExp(q, 'i');
      test = (s) => re.test(s);
    } catch {
      const lower = q.toLowerCase();
      test = (s) => String(s).toLowerCase().includes(lower);
    }
    return rows.filter(row => {
      if (test(String(row.index))) return true;
      for (const col of columns) {
        const cell = row.cells[col.oid];
        if (!cell) continue;
        if (test(String(cell.value)) || test(col.name) || test(cell.fullOid || '')) return true;
      }
      return false;
    });
  }

  // Reactive: reset table view when operation changes away from WALK/GETBULK
  $: if (activeOperation !== 'WALK' && activeOperation !== 'GETBULK') {
    tableViewEnabled = false;
  }

  // Can show comparison view: multi-target + WALK/GETBULK
  $: canShowComparison = (activeOperation === 'WALK' || activeOperation === 'GETBULK')
    && bulkResults.filter(r => !r.error && Array.isArray(r.result?.value)).length > 1;

  $: uniqueTargets = [...new Set(bulkResults.filter(r => !r.error).map(r => r.target))];
  $: canCompare = uniqueTargets.length >= 2;

  // Wrapper: resolves oidInfoCache entry then delegates to shared util
  function formatValueWithEnum(value, oid, snmpType) {
    return _formatValueWithEnum(value, oidInfoCache[oid], snmpType);
  }


  // Export results as CSV
  function exportAsCSV() {
    if (bulkResults.length === 0) return;
    const lines = [];
    const isMulti = activeOperation === 'WALK' || activeOperation === 'GETBULK';

    if (isMulti) {
      lines.push('Target,OID,Type,Value');
      for (const res of bulkResults) {
        if (res.error) {
          lines.push(`${escapeCSV(res.target)},,,"Error: ${escapeCSV(res.error)}"`);
          continue;
        }
        if (Array.isArray(res.result?.value)) {
          for (const item of res.result.value) {
            lines.push(`${escapeCSV(res.target)},${escapeCSV(item.oid)},${escapeCSV(item.type)},${escapeCSV(typeof item.value === 'string' ? item.value : JSON.stringify(item.value))}`);
          }
        }
      }
    } else {
      lines.push('Target,OID,Type,Value,Error');
      for (const res of bulkResults) {
        if (res.error) {
          lines.push(`${escapeCSV(res.target)},,,,${escapeCSV(res.error)}`);
        } else {
          lines.push(`${escapeCSV(res.target)},${escapeCSV(res.result.oid)},${escapeCSV(res.result.type)},${escapeCSV(typeof res.result.value === 'string' ? res.result.value : JSON.stringify(res.result.value))},`);
        }
      }
    }

    const timestamp = new Date().toISOString().replace(/[:.]/g, '-').slice(0, 19);
    downloadFile(lines.join('\n'), `snmp-${activeOperation.toLowerCase()}-${timestamp}.csv`, 'text/csv');
    notificationStore.add(get(_)('results.exportedCsv'), 'success');
  }

  // Export results as text
  function exportAsText() {
    if (bulkResults.length === 0) return;
    const lines = [];
    const isMulti = activeOperation === 'WALK' || activeOperation === 'GETBULK';

    for (const res of bulkResults) {
      lines.push(`--- Target: ${res.target} ---`);
      if (res.error) {
        lines.push(`  Error: ${res.error}`);
      } else if (isMulti && Array.isArray(res.result?.value)) {
        for (const item of res.result.value) {
          const val = typeof item.value === 'string' ? item.value : JSON.stringify(item.value);
          lines.push(`  ${item.oid} = ${item.type}: ${val}`);
        }
        lines.push(`  (${res.result.value.length} results)`);
      } else {
        const val = typeof res.result.value === 'string' ? res.result.value : JSON.stringify(res.result.value);
        lines.push(`  ${res.result.oid} = ${res.result.type}: ${val}`);
      }
      lines.push('');
    }

    const timestamp = new Date().toISOString().replace(/[:.]/g, '-').slice(0, 19);
    downloadFile(lines.join('\n'), `snmp-${activeOperation.toLowerCase()}-${timestamp}.txt`, 'text/plain');
    notificationStore.add(get(_)('results.exportedTxt'), 'success');
  }

  // Export table view as CSV
  function exportTableAsCSV() {
    if (bulkResults.length === 0 || !effectiveTableNode) return;
    const colDefs = getTableColumnDefs(effectiveTableNode);
    if (colDefs.length === 0) return;

    // Use the first result's walk data
    const firstRes = bulkResults.find(r => !r.error && Array.isArray(r.result?.value));
    if (!firstRes) return;

    const tableData = buildTableData(firstRes.result.value, colDefs, sortColumn, sortAscending);
    const lines = [];

    // Header
    lines.push(['Index', ...tableData.columns.map(c => c.name)].map(escapeCSV).join(','));

    // Rows
    for (const row of tableData.rows) {
      const cells = [row.index, ...tableData.columns.map(col => {
        const cell = row.cells[col.oid];
        if (!cell) return '';
        return typeof cell.value === 'string' ? cell.value : JSON.stringify(cell.value);
      })];
      lines.push(cells.map(escapeCSV).join(','));
    }

    const timestamp = new Date().toISOString().replace(/[:.]/g, '-').slice(0, 19);
    downloadFile(lines.join('\n'), `snmp-table-${timestamp}.csv`, 'text/csv');
    notificationStore.add(get(_)('results.exportedTable'), 'success');
  }

  // Display value of a table cell (enum-decoded / formatted, like the rendered cell).
  function cellText(cell) {
    return cell && cell.value !== undefined ? formatValueWithEnum(cell.value, cell.fullOid || '', cell.type) : '';
  }

  const htmlEscape = (s) => String(s).replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');

  // Copy the whole table as an HTML table (pastes into Word/Docs/Outlook as a
  // real table) with a TSV plain-text fallback for editors and spreadsheets.
  function copyTableForWord() {
    if (bulkResults.length === 0 || !effectiveTableNode) return;
    const colDefs = getTableColumnDefs(effectiveTableNode);
    if (colDefs.length === 0) return;
    const firstRes = bulkResults.find(r => !r.error && Array.isArray(r.result?.value));
    if (!firstRes) return;
    const td = buildTableData(firstRes.result.value, colDefs, sortColumn, sortAscending);

    const headers = [get(_)('results.index'), ...td.columns.map(c => c.name)];
    let html = '<table border="1" cellspacing="0" cellpadding="4" style="border-collapse:collapse">';
    html += '<thead><tr>' + headers.map(h => `<th>${htmlEscape(h)}</th>`).join('') + '</tr></thead><tbody>';
    const tsv = [headers.join('\t')];
    for (const row of td.rows) {
      const cells = [row.index, ...td.columns.map(col => cellText(row.cells[col.oid]))];
      html += '<tr>' + cells.map(c => `<td>${htmlEscape(c)}</td>`).join('') + '</tr>';
      tsv.push(cells.join('\t'));
    }
    html += '</tbody></table>';
    copyRich(html, tsv.join('\n'), get(_)('results.tableView'));
  }

  // Right-click a table cell → context menu with copy actions.
  function openCellMenu(event, row, col, columns) {
    event.preventDefault();
    event.stopPropagation();
    const cell = row.cells[col.oid];
    cellMenuCtx = { row, col, columns, cell };
    const t = get(_);
    cellMenu = {
      visible: true,
      x: event.clientX,
      y: event.clientY,
      items: [
        { label: t('common.copyValue'), icon: 'copy', action: 'value', disabled: !cell },
        { label: t('common.copyOid'), icon: 'route', action: 'oid', disabled: !cell },
        { label: t('results.copyOidValue'), icon: 'copy', action: 'oidValue', disabled: !cell },
        { label: '---', action: 'sep' },
        { label: t('results.copyRow'), icon: 'table', action: 'row' },
        { label: t('results.copyIndex'), icon: 'copy', action: 'index' },
        { label: t('results.copyColumn'), icon: 'columns-3', action: 'column' },
      ],
    };
  }

  function handleCellMenuAction(e) {
    const action = e.detail.action;
    const ctx = cellMenuCtx;
    cellMenu = { ...cellMenu, visible: false };
    if (!ctx) return;
    const { row, col, columns, cell } = ctx;
    const t = get(_);
    if (action === 'value') copyToClipboard(cellText(cell), t('common.value'));
    else if (action === 'oid') copyToClipboard(cell?.fullOid || '', t('common.oid'));
    else if (action === 'oidValue') copyToClipboard(`${cell?.fullOid || ''} = ${cellText(cell)}`, t('common.value'));
    else if (action === 'index') copyToClipboard(String(row.index), t('results.index'));
    else if (action === 'column') copyToClipboard(col.name, col.name);
    else if (action === 'row') {
      const cells = [row.index, ...columns.map(c => cellText(row.cells[c.oid]))];
      copyToClipboard(cells.join('\t'), t('results.tableView'));
    }
  }

  // ============ TABLE VIEW FUNCTIONS ============

  // Get column definitions from the MIB tree for a Table or Row node
  function getTableColumnDefs(node) {
    if (!node) return [];
    let rowNode = node;
    if (node.mibType === 'Table') {
      rowNode = (node.children || []).find(c => c.mibType === 'Row');
      if (!rowNode) return [];
    }
    if (rowNode.mibType !== 'Row') return [];
    return (rowNode.children || [])
      .filter(c => c.mibType === 'Column')
      .sort((a, b) => {
        const aLast = parseInt(a.oid.split('.').pop());
        const bLast = parseInt(b.oid.split('.').pop());
        return aLast - bLast;
      });
  }

  // Check if table view is applicable
  function canShowTableView(node, results) {
    if (!node) return false;
    if (results.length === 0) return false;
    const nodeType = node.mibType;
    if (nodeType !== 'Table' && nodeType !== 'Row') return false;
    return getTableColumnDefs(node).length > 0;
  }

  // Reconstruct WALK results into a structured table
  function buildTableData(walkResults, columnDefs, sortCol = null, sortAsc = true) {
    const columns = columnDefs.map(col => ({ name: col.name, oid: col.oid, syntax: col.syntax || '' }));
    const rowMap = {};

    // gosnmp returns walk OIDs with a leading dot (".1.3.6...") while MIB column
    // OIDs do not, so normalize both sides before matching — otherwise no cell
    // matches its column and the table renders headers with no rows.
    const stripDot = (o) => (o && o.charAt(0) === '.' ? o.slice(1) : o);
    for (const item of walkResults) {
      const itemOid = stripDot(item.oid);
      let matchedCol = null;
      let instanceIdx = '';
      for (const col of columnDefs) {
        const colOid = stripDot(col.oid);
        if (itemOid.startsWith(colOid + '.')) {
          matchedCol = col;
          instanceIdx = itemOid.substring(colOid.length + 1);
          break;
        }
      }
      if (!matchedCol) continue;

      if (!rowMap[instanceIdx]) {
        rowMap[instanceIdx] = {};
      }
      rowMap[instanceIdx][matchedCol.oid] = {
        value: item.value,
        type: item.type,
        fullOid: item.oid
      };
    }

    let rows = Object.entries(rowMap).map(([index, cells]) => ({ index, cells }));

    // Apply sorting
    if (sortCol) {
      rows.sort((a, b) => {
        let aVal, bVal;
        if (sortCol === '__index') {
          aVal = a.index;
          bVal = b.index;
        } else {
          aVal = a.cells[sortCol]?.value ?? '';
          bVal = b.cells[sortCol]?.value ?? '';
        }
        const aNum = Number(aVal);
        const bNum = Number(bVal);
        let cmp;
        if (!isNaN(aNum) && !isNaN(bNum)) {
          cmp = aNum - bNum;
        } else {
          cmp = String(aVal).localeCompare(String(bVal));
        }
        return sortAsc ? cmp : -cmp;
      });
    }

    return { columns, rows };
  }

  $: autoDetectedTableNode = (() => {
    // Only try auto-detection for WALK/GETBULK when selectedNode doesn't provide table structure
    if (activeOperation !== 'WALK' && activeOperation !== 'GETBULK') return null;
    if (selectedNode && canShowTableView(selectedNode, bulkResults)) return null;
    if (!bulkResults.length || !mibTree.length) return null;

    // Find first successful walk result with data
    const firstRes = bulkResults.find(r => !r.error && r.result?.type === 'WalkResponse' && Array.isArray(r.result?.value) && r.result.value.length > 0);
    if (!firstRes) return null;

    // Try to detect table from first few OIDs
    return findTableParentNode(firstRes.result.value[0].oid, mibTree);
  })();

  // Use detected table node as fallback for table view
  $: effectiveTableNode = (selectedNode && canShowTableView(selectedNode, bulkResults)) ? selectedNode : autoDetectedTableNode;

  // New result set → reset the filter and re-arm the one-time view decision.
  $: if (bulkResults !== lastResultsForView) {
    lastResultsForView = bulkResults;
    walkFilter = '';
    rawSortKey = null;
    autoViewApplied = false;
  }

  // Decide the default view exactly once (table when a Table/Row is detected,
  // raw otherwise). Only once, so it never flips while the user interacts.
  $: if (!autoViewApplied && bulkResults.length > 0 && (activeOperation === 'WALK' || activeOperation === 'GETBULK')) {
    tableViewEnabled = !!(effectiveTableNode && (effectiveTableNode.mibType === 'Table' || effectiveTableNode.mibType === 'Row'));
    autoViewApplied = true;
  }

  function handleColumnSort(colId) {
    if (sortColumn === colId) {
      sortAscending = !sortAscending;
    } else {
      sortColumn = colId;
      sortAscending = true;
    }
  }
</script>

{#if bulkResults.length > 0}
  <div class="results-container">
    {#if cellMenu.visible}
      <ContextMenu
        x={cellMenu.x}
        y={cellMenu.y}
        items={cellMenu.items}
        on:action={handleCellMenuAction}
        on:close={() => (cellMenu = { ...cellMenu, visible: false })}
      />
    {/if}
    <div class="results-header">
      <h4>{$_('results.title')}</h4>
      <div class="export-buttons">
        {#if canShowComparison}
          <button
            class="btn-view"
            class:active={comparisonViewEnabled}
            on:click={() => comparisonViewEnabled = !comparisonViewEnabled}
          >
            {$_('results.comparison')}
          </button>
        {/if}
        {#if canCompare}
          <button class="btn-view" class:active={compareEnabled} on:click={() => { compareEnabled = !compareEnabled; }}>
            {$_('results.compare')}
          </button>
        {/if}
        <button class="btn-export" on:click={exportAsCSV} title={$_('results.csv')}>{$_('results.csv')}</button>
        <button class="btn-export" on:click={exportAsText} title={$_('results.txt')}>{$_('results.txt')}</button>
        {#if tableViewEnabled && canShowTableView(effectiveTableNode, bulkResults)}
          <button class="btn-export" on:click={exportTableAsCSV} title={$_('results.tableCsv')}>{$_('results.tableCsv')}</button>
          <button class="btn-export" on:click={copyTableForWord} title={$_('results.copyForWordHint')}>
            <Icon name="copy" size={13} /> {$_('results.copyForWord')}
          </button>
        {/if}
      </div>
    </div>

    {#if compareEnabled}
      <ResultsComparison mode="enhanced" {bulkResults} {oidInfoCache} />
    {/if}

    {#if comparisonViewEnabled && canShowComparison}
      <ResultsComparison mode="legacy" {bulkResults} {oidInfoCache} />
    {:else}
      {#each bulkResults as res}
        <div class="result" class:success={!res.error} class:error={res.error}>
          <p class="result-target">
            {$anonMode ? anonymizeIp(res.target) : res.target}
            {#if res.responseTimeMs}
              <span class="response-time-badge">{res.responseTimeMs}ms</span>
            {/if}
          </p>
          {#if res.error}
            <p><strong>{$_('common.error')}:</strong> {res.error}</p>
          {:else if (res.result.type === 'WalkResponse' || res.result.type === 'GetBulkResponse') && Array.isArray(res.result.value)}
          <!-- WALK/GETBULK results display -->
          <div class="result-fields walk-summary">
            <span class="rfield">
              <span class="rlabel">{$_('results.baseOid')}</span>
              <span class="rval mono">{res.result.oid}</span>
            </span>
            <span class="rfield rcount">{$_('results.resultsFound', { values: { count: res.result.value.length } })}</span>
          </div>

          {#if canShowTableView(effectiveTableNode, bulkResults)}
            <div class="view-toggle">
              <button
                class="btn-view"
                class:active={!tableViewEnabled}
                on:click={() => { tableViewEnabled = false; }}
              >
                {$_('results.rawView')}
              </button>
              <button
                class="btn-view"
                class:active={tableViewEnabled}
                on:click={() => { tableViewEnabled = true; sortColumn = null; }}
              >
                {$_('results.tableView')}
              </button>
            </div>
          {/if}

          {#if tableViewEnabled && canShowTableView(effectiveTableNode, bulkResults)}
            {@const colDefs = getTableColumnDefs(effectiveTableNode)}
            {@const tableData = buildTableData(res.result.value, colDefs, sortColumn, sortAscending)}
            {@const tableRows = filterTableRows(tableData.rows, tableData.columns, walkFilter)}
            <div class="walk-filter-bar">
              <input
                type="text"
                class="walk-filter-input"
                bind:value={walkFilter}
                placeholder={$_('results.filterPlaceholder')}
              />
              {#if walkFilter.trim()}
                <span class="walk-filter-count">{tableRows.length} / {tableData.rows.length}</span>
                <button class="btn-copy-small" on:click={() => walkFilter = ''} title={$_('common.clear')}>&times;</button>
              {/if}
            </div>
            <div class="table-view-results">
              <table>
                <thead>
                  <tr>
                    <th
                      class="sortable"
                      on:click={() => handleColumnSort('__index')}
                    >
                      {$_('results.index')} {sortColumn === '__index' ? (sortAscending ? '▲' : '▼') : ''}
                    </th>
                    {#each tableData.columns as col}
                      <th
                        class="sortable"
                        on:click={() => handleColumnSort(col.oid)}
                        title="{col.oid} ({col.syntax})"
                      >
                        {col.name}
                        {#if sortColumn === col.oid}
                          {sortAscending ? '▲' : '▼'}
                        {/if}
                      </th>
                    {/each}
                  </tr>
                </thead>
                <tbody>
                  {#each tableRows as row}
                    <tr>
                      <td class="index-cell">{row.index}</td>
                      {#each tableData.columns as col}
                        <td
                          class="table-value-cell clickable"
                          title={row.cells[col.oid]?.fullOid || ''}
                          on:click={() => row.cells[col.oid] && dispatch('walkResultClick', {oid: row.cells[col.oid].fullOid, value: row.cells[col.oid].value, type: row.cells[col.oid].type})}
                          on:keydown={(e) => e.key === 'Enter' && row.cells[col.oid] && dispatch('walkResultClick', {oid: row.cells[col.oid].fullOid, value: row.cells[col.oid].value, type: row.cells[col.oid].type})}
                          on:contextmenu={(e) => openCellMenu(e, row, col, tableData.columns)}
                        >
                          {row.cells[col.oid]?.value !== undefined ? formatValueWithEnum(row.cells[col.oid].value, row.cells[col.oid].fullOid || '', row.cells[col.oid].type) : '-'}
                        </td>
                      {/each}
                    </tr>
                  {/each}
                </tbody>
              </table>
            </div>
            <p class="table-info">{$_('results.tableInfo', { values: { rows: tableRows.length, cols: tableData.columns.length } })}</p>
          {:else}
            <!-- Raw WALK results table -->
            {@const filtered = sortWalkItems(filterWalkItems(res.result.value, walkFilter), rawSortKey, rawSortAsc)}
            <div class="walk-filter-bar">
              <input
                type="text"
                class="walk-filter-input"
                bind:value={walkFilter}
                placeholder={$_('results.filterPlaceholder')}
              />
              {#if walkFilter.trim()}
                <span class="walk-filter-count">{filtered.length} / {res.result.value.length}</span>
                <button class="btn-copy-small" on:click={() => walkFilter = ''} title={$_('common.clear')}>&times;</button>
              {/if}
            </div>
            <div class="walk-results">
              <table>
                <thead>
                  <tr>
                    <th class="sortable" on:click={() => sortRaw('oid')}>{$_('common.oid')} {rawSortKey === 'oid' ? (rawSortAsc ? '▲' : '▼') : ''}</th>
                    <th class="sortable" on:click={() => sortRaw('type')}>{$_('common.type')} {rawSortKey === 'type' ? (rawSortAsc ? '▲' : '▼') : ''}</th>
                    <th class="sortable" on:click={() => sortRaw('value')}>{$_('common.value')} {rawSortKey === 'value' ? (rawSortAsc ? '▲' : '▼') : ''}</th>
                    <th class="copy-col"></th>
                  </tr>
                </thead>
                <tbody>
                  {#each filtered as walkItem}
                    <tr
                      class="walk-result-row clickable"
                      on:click={() => dispatch('walkResultClick', walkItem)}
                      on:keydown={(e) => e.key === 'Enter' && dispatch('walkResultClick', walkItem)}
                      role="button"
                      tabindex="0"
                      title={$_('results.clickToUseOid')}
                    >
                      <td class="oid-cell" title={walkItem.oid}>
                        {#if oidInfoCache[walkItem.oid]?.name}
                          <span class="oid-name">{oidInfoCache[walkItem.oid].name}</span>
                        {/if}
                        <span class="oid-raw">{walkItem.oid}</span>
                      </td>
                      <td>{walkItem.type}</td>
                      <td class="value-cell" title={JSON.stringify(walkItem.value)}>{formatValueWithEnum(walkItem.value, walkItem.oid, walkItem.type)}</td>
                      <td class="copy-cell">
                        <button
                          class="btn-copy-small"
                          on:click|stopPropagation={() => copyToClipboard(String(walkItem.value), $_('common.value'))}
                          title={$_('common.copyValue')}
                        ><Icon name="copy" size={13} /></button>
                      </td>
                    </tr>
                  {/each}
                </tbody>
              </table>
            </div>
          {/if}
        {:else}
          <!-- GET/SET results display -->
          <div class="result-fields">
            <span class="rfield">
              <span class="rlabel">{$_('common.oid')}</span>
              <span class="rval mono">{res.result.oid}</span>
              <button class="btn-copy-small" on:click={() => copyToClipboard(res.result.oid, $_('common.oid'))} title={$_('common.copyOid')}><Icon name="copy" size={13} /></button>
            </span>
            <span class="rfield">
              <span class="rlabel">{$_('common.type')}</span>
              <span class="rval">{res.result.type}{#if oidInfoCache[res.result.oid]?.name} <span class="resolved-name">({oidInfoCache[res.result.oid].name})</span>{/if}</span>
            </span>
            <span class="rfield rfield-grow">
              <span class="rlabel">{$_('common.value')}</span>
              <span class="rval">{formatValueWithEnum(res.result.value, res.result.oid, res.result.type)}</span>
              <button class="btn-copy-small" on:click={() => copyToClipboard(String(res.result.value), $_('common.value'))} title={$_('common.copyValue')}><Icon name="copy" size={13} /></button>
            </span>
          </div>
        {/if}
        </div>
      {/each}
    {/if}
  </div>
{/if}

<style>
  .results-container {
    margin-top: 20px;
  }

  .result {
    margin-top: 10px;
    padding: 12px;
    border-radius: 5px;
    border: 1px solid;
  }

  .result-target {
    font-weight: bold;
    margin-bottom: 8px;
  }

  /* Aligned, responsive result fields (GET/SET display + walk summary) */
  .result-fields {
    display: flex;
    flex-wrap: wrap;
    align-items: baseline;
    row-gap: 6px;
    margin-bottom: 4px;
  }

  .rfield {
    display: inline-flex;
    align-items: baseline;
    gap: 6px;
    min-width: 0;
    padding: 0 16px;
    border-left: 1px solid var(--border-color);
  }

  .rfield:first-child {
    padding-left: 0;
    border-left: none;
  }

  .rfield-grow {
    flex: 1 1 220px;
  }

  .rlabel {
    color: var(--text-muted);
    font-size: 0.78em;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.04em;
    flex-shrink: 0;
  }

  .rval {
    min-width: 0;
    overflow-wrap: anywhere;
  }

  .walk-summary .rcount {
    color: var(--text-light);
    font-weight: 600;
  }

  /* OID cell: MIB name and raw OID separated by a light divider */
  .oid-name {
    color: var(--oid-color);
    font-weight: 600;
    margin-right: 8px;
    padding-right: 8px;
    border-right: 1px solid var(--border-color);
  }

  .oid-raw {
    color: var(--text-muted);
  }

  /* Light vertical separators between result-table columns */
  .walk-results td,
  .walk-results th,
  .table-view-results td,
  .table-view-results th {
    border-right: 1px solid var(--border-color);
  }

  .walk-results td:last-child,
  .walk-results th:last-child,
  .table-view-results td:last-child,
  .table-view-results th:last-child {
    border-right: none;
  }

  .success {
    background-color: var(--success-subtle-medium);
    border-color: var(--success-color);
  }

  .error {
    background-color: var(--error-subtle-medium);
    border-color: var(--error-color);
    color: var(--error-color);
  }

  .walk-filter-bar {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-top: 10px;
    margin-bottom: 6px;
  }

  .walk-filter-input {
    flex: 1;
    max-width: 350px;
    padding: 5px 10px;
    font-size: 0.85em;
    background-color: var(--bg-lighter-color);
    border: 1px solid var(--border-color);
    border-radius: 4px;
    color: var(--text-color);
  }

  .walk-filter-input:focus {
    outline: none;
    border-color: var(--accent-color);
  }

  .walk-filter-count {
    font-size: 0.8em;
    color: var(--text-muted);
    white-space: nowrap;
  }

  .walk-results {
    margin-top: 0;
    max-height: 400px;
    overflow-y: auto;
    border: 1px solid var(--border-color);
    border-radius: 4px;
  }

  .walk-results table {
    width: 100%;
    border-collapse: collapse;
    font-size: 0.9em;
  }

  .walk-results thead {
    position: sticky;
    top: 0;
    background-color: var(--bg-lighter-color);
    z-index: 1;
  }

  .walk-results th {
    text-align: left;
    padding: 8px;
    border-bottom: 2px solid var(--border-color);
    font-weight: 600;
  }

  .walk-results th.sortable {
    cursor: pointer;
    user-select: none;
    white-space: nowrap;
  }

  .walk-results th.sortable:hover {
    background-color: var(--hover-overlay);
  }

  .walk-results td {
    padding: 6px 8px;
    border-bottom: 1px solid var(--border-color);
  }

  .walk-results tr:hover {
    background-color: var(--hover-overlay);
  }

  .response-time-badge {
    font-size: 0.8em;
    padding: 2px 8px;
    border-radius: 10px;
    margin-left: 8px;
    font-weight: 600;
    background-color: var(--accent-subtle-strong);
    color: var(--oid-color);
  }

  .oid-name {
    color: var(--name-color);
    font-size: 0.85em;
    margin-right: 6px;
    font-family: inherit;
  }

  .resolved-name {
    color: var(--name-color);
    font-size: 0.9em;
    margin-left: 6px;
  }

  .walk-results .oid-cell {
    font-family: 'Courier New', monospace;
    font-size: 0.85em;
    color: var(--oid-color);
    max-width: 300px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .walk-results .value-cell {
    max-width: 200px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  /* Clickable WALK result rows */
  .walk-result-row.clickable {
    cursor: pointer;
    transition: background-color 0.15s ease;
  }

  .walk-result-row.clickable:hover {
    background-color: var(--accent-subtle-intense) !important;
  }

  .walk-result-row.clickable:focus {
    outline: 2px solid var(--accent-color);
    outline-offset: -2px;
  }

  .walk-result-row.clickable:hover .oid-cell {
    color: var(--accent-color);
    text-decoration: underline;
  }

  /* Copy buttons */
  .btn-copy-small {
    background: transparent;
    border: none;
    cursor: pointer;
    padding: 2px 4px;
    font-size: 0.85em;
    opacity: 0.5;
    transition: opacity 0.2s;
  }

  .btn-copy-small:hover {
    opacity: 1;
  }

  .copy-col {
    width: 40px;
  }

  .copy-cell {
    text-align: center;
  }

  /* Table View styles */
  .view-toggle {
    display: flex;
    gap: 4px;
    margin: 10px 0;
  }

  .btn-view {
    padding: 6px 14px;
    font-size: 0.85em;
    background-color: transparent;
    border: 1px solid var(--border-color);
    color: var(--text-muted);
    border-radius: 4px;
    cursor: pointer;
    transition: all 0.2s;
    font-weight: 500;
  }

  .btn-view:hover {
    border-color: var(--accent-color);
    color: var(--text-color);
  }

  .btn-view.active {
    background-color: var(--accent-color);
    border-color: var(--accent-color);
    color: white;
  }

  .table-view-results {
    margin-top: 10px;
    max-height: 500px;
    overflow: auto;
    border: 1px solid var(--border-color);
    border-radius: 4px;
  }

  .table-view-results table {
    width: 100%;
    border-collapse: collapse;
    font-size: 0.9em;
  }

  .table-view-results thead {
    position: sticky;
    top: 0;
    background-color: var(--bg-lighter-color);
    z-index: 1;
  }

  .table-view-results th {
    text-align: left;
    padding: 8px;
    border-bottom: 2px solid var(--border-color);
    font-weight: 600;
    white-space: nowrap;
  }

  .table-view-results th.sortable {
    cursor: pointer;
    user-select: none;
  }

  .table-view-results th.sortable:hover {
    background-color: var(--accent-subtle-strong);
    color: var(--accent-color);
  }

  .table-view-results td {
    padding: 6px 8px;
    border-bottom: 1px solid var(--border-color);
    max-width: 200px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .table-view-results tr:hover {
    background-color: var(--hover-overlay);
  }

  .table-value-cell {
    cursor: pointer;
  }

  .table-value-cell:hover {
    background-color: var(--accent-subtle-strong);
    color: var(--accent-color);
  }

  .index-cell {
    font-family: 'Courier New', monospace;
    color: var(--oid-color);
    font-size: 0.85em;
    font-weight: 600;
  }

  .table-info {
    font-size: 0.85em;
    color: var(--text-muted);
    margin-top: 8px;
    font-style: italic;
    text-align: center;
  }

  /* Export buttons */
  .results-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 5px;
  }

  .results-header h4 {
    margin: 0;
  }

  .export-buttons {
    display: flex;
    gap: 6px;
  }

  .btn-export {
    padding: 4px 10px;
    font-size: 0.8em;
    background-color: transparent;
    border: 1px solid var(--border-color);
    color: var(--text-dimmed);
    border-radius: 3px;
    cursor: pointer;
    font-weight: 500;
    transition: all 0.2s;
  }

  .btn-export:hover {
    border-color: var(--accent-color);
    color: var(--accent-color);
    background-color: var(--accent-subtle-medium);
  }

</style>
