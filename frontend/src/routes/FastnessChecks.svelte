<script>
  import { onMount } from 'svelte';
  import { api, toLocalInput, fromLocalInput } from '../lib/api.js';

  let lots = [];
  let rows = [];
  let error = '';
  let form = {
    dyeLotId: '',
    checkedAt: toLocalInput(new Date().toISOString()),
    washFastness: 4,
    rubFastness: 3.5,
    tempC: 40,
    notes: '',
  };
  let editing = null;

  async function load() {
    error = '';
    try {
      [lots, rows] = await Promise.all([api('/dye-lots'), api('/fastness-checks')]);
      if (!form.dyeLotId) {
        // 默认选第一个未关闭染程；全部关闭时退回第一条
        const firstOpen = lots.find((l) => !l.closed);
        form.dyeLotId = String((firstOpen || lots[0] || {}).id ?? '');
      }
    } catch (e) {
      error = e.message;
    }
  }

  onMount(load);

  $: selectedLot = lots.find((x) => x.id === Number(form.dyeLotId)) || null;
  $: selectedClosed = !!selectedLot?.closed;
  // 已关闭染程禁止追加抽检；后端同样强制 409，这里只是先拦住误操作
  $: createBlocked = !editing && selectedClosed;

  function lotLabel(id) {
    const lot = lots.find((x) => x.id === id);
    return lot ? `${lot.recipeName} (#${lot.id})` : id;
  }

  async function save() {
    error = '';
    try {
      const body = {
        dyeLotId: Number(form.dyeLotId),
        checkedAt: fromLocalInput(form.checkedAt),
        washFastness: Number(form.washFastness),
        rubFastness: Number(form.rubFastness),
        tempC: Number(form.tempC),
        notes: form.notes.trim() || null,
      };
      if (editing) {
        await api(`/fastness-checks/${editing}`, { method: 'PUT', body: JSON.stringify(body) });
      } else {
        await api('/fastness-checks', { method: 'POST', body: JSON.stringify(body) });
      }
      editing = null;
      form = {
        ...form,
        checkedAt: toLocalInput(new Date().toISOString()),
        notes: '',
      };
      await load();
    } catch (e) {
      error = e.message;
    }
  }

  function startEdit(row) {
    editing = row.id;
    form = {
      dyeLotId: String(row.dyeLotId),
      checkedAt: toLocalInput(row.checkedAt),
      washFastness: row.washFastness,
      rubFastness: row.rubFastness,
      tempC: row.tempC,
      notes: row.notes || '',
    };
  }

  async function remove(id) {
    if (!confirm('确认删除该抽检？')) return;
    error = '';
    try {
      await api(`/fastness-checks/${id}`, { method: 'DELETE' });
      await load();
    } catch (e) {
      error = e.message;
    }
  }
</script>

<h1 class="page-title">色牢度抽检</h1>
<p class="page-sub">耐洗 1–5 级；摩擦牢度须大于 0；记录检测温度。染程关闭后任何人不可再追加抽检。</p>

<div class="panel" style="margin-bottom:1rem;">
  <div class="form-grid">
    <label
      >染程
      <select bind:value={form.dyeLotId} disabled={editing !== null}>
        {#each lots as lot}
          <option value={String(lot.id)} disabled={lot.closed && !editing}
            >{lot.recipeName} · {lot.fabricKg}kg{lot.closed ? '（已关闭）' : ''}</option
          >
        {/each}
      </select>
    </label>
    <label>检测时间 <input type="datetime-local" bind:value={form.checkedAt} /></label>
    <label>耐洗 (1–5) <input type="number" min="1" max="5" bind:value={form.washFastness} /></label>
    <label>摩擦 (&gt;0) <input type="number" step="0.1" min="0.1" bind:value={form.rubFastness} /></label>
    <label>温度 ℃ <input type="number" step="0.1" bind:value={form.tempC} /></label>
    <label>备注 <input bind:value={form.notes} /></label>
  </div>
  {#if createBlocked}
    <p class="warn">该染程已关闭，禁止再追加色牢度抽检（操作员与主管均不可）。</p>
  {/if}
  <div class="toolbar">
    <button class="btn" type="button" disabled={createBlocked} on:click={save}
      >{editing ? '保存修改' : '登记抽检'}</button
    >
    {#if editing}
      <button class="btn ghost" type="button" on:click={() => (editing = null)}>取消</button>
    {/if}
  </div>
  {#if error}<p class="err">{error}</p>{/if}
</div>

<div class="panel">
  <table>
    <thead>
      <tr>
        <th>ID</th>
        <th>染程</th>
        <th>检测时间</th>
        <th>耐洗</th>
        <th>摩擦</th>
        <th>温度</th>
        <th>备注</th>
        <th></th>
      </tr>
    </thead>
    <tbody>
      {#each rows as row}
        <tr>
          <td>{row.id}</td>
          <td>
            {lotLabel(row.dyeLotId)}
            {#if lots.find((l) => l.id === row.dyeLotId)?.closed}
              <span class="tag-closed">染程已关闭</span>
            {/if}
          </td>
          <td>{new Date(row.checkedAt).toLocaleString()}</td>
          <td>{row.washFastness}</td>
          <td>{row.rubFastness}</td>
          <td>{row.tempC}℃</td>
          <td>{row.notes || '—'}</td>
          <td class="row-actions">
            <button class="btn ghost small" type="button" on:click={() => startEdit(row)}>编辑</button>
            <button class="btn danger small" type="button" on:click={() => remove(row.id)}>删除</button>
          </td>
        </tr>
      {/each}
    </tbody>
  </table>
</div>

<style>
  .warn {
    margin: 0.5rem 0 0;
    padding: 0.5rem 0.75rem;
    font-size: 0.85rem;
    color: #f3d27a;
    border: 1px solid rgba(243, 210, 122, 0.45);
    background: rgba(243, 210, 122, 0.08);
    border-radius: 2px;
  }

  .tag-closed {
    margin-left: 0.4rem;
    padding: 0.05rem 0.4rem;
    border-radius: 2px;
    font-size: 0.7rem;
    color: var(--indigo-mist);
    border: 1px solid var(--line);
  }
</style>
