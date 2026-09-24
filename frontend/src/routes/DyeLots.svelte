<script>
  import { onMount } from 'svelte';
  import { api, VAT_STATUS, toLocalInput, fromLocalInput } from '../lib/api.js';
  import { user } from '../lib/auth.js';

  let vats = [];
  let rows = [];
  let error = '';
  let closedFilter = ''; // '' 全部 · 'false' 未关闭 · 'true' 已关闭
  let form = {
    vatId: '',
    recipeName: '',
    fabricKg: 20,
    startedAt: toLocalInput(new Date().toISOString()),
    operatorName: '',
  };
  let editing = null;

  $: isAdmin = $user?.role === 'admin';
  // 新建时操作人恒为登录显示名（只读）；编辑时保留原操作人
  $: if (!editing) form.operatorName = $user?.displayName || '';

  async function load() {
    error = '';
    try {
      const lotQuery = closedFilter === '' ? '' : `?closed=${closedFilter}`;
      [vats, rows] = await Promise.all([api('/vats'), api(`/dye-lots${lotQuery}`)]);
      const usable = vats.filter((v) => v.status === 'ready' || v.status === 'dyeing');
      if (!form.vatId && usable.length) form.vatId = String(usable[0].id);
      else if (!form.vatId && vats.length) form.vatId = String(vats[0].id);
    } catch (e) {
      error = e.message;
    }
  }

  onMount(load);

  function vatLabel(id) {
    const v = vats.find((x) => x.id === id);
    if (!v) return id;
    return `${v.vatCode}（${VAT_STATUS[v.status] || v.status}）`;
  }

  async function save() {
    error = '';
    try {
      const body = {
        vatId: Number(form.vatId),
        recipeName: form.recipeName.trim(),
        fabricKg: Number(form.fabricKg),
        startedAt: fromLocalInput(form.startedAt),
      };
      if (editing) {
        // 操作人不可修改，更新时不提交该字段
        await api(`/dye-lots/${editing}`, { method: 'PUT', body: JSON.stringify(body) });
      } else {
        body.operatorName = ($user?.displayName || '').trim();
        await api('/dye-lots', { method: 'POST', body: JSON.stringify(body) });
      }
      editing = null;
      form = {
        ...form,
        recipeName: '',
        fabricKg: 20,
        startedAt: toLocalInput(new Date().toISOString()),
      };
      await load();
    } catch (e) {
      error = e.message;
    }
  }

  function startEdit(row) {
    editing = row.id;
    form = {
      vatId: String(row.vatId),
      recipeName: row.recipeName,
      fabricKg: row.fabricKg,
      startedAt: toLocalInput(row.startedAt),
      operatorName: row.operatorName,
    };
  }

  async function closeLot(row) {
    if (!confirm(`确认关闭染程 #${row.id}（${row.recipeName}）？关闭后不可再追加色牢度抽检。`)) return;
    error = '';
    try {
      await api(`/dye-lots/${row.id}/close`, { method: 'POST' });
      await load();
    } catch (e) {
      error = e.message;
    }
  }

  async function remove(id) {
    if (!confirm('确认删除该染程？')) return;
    error = '';
    try {
      await api(`/dye-lots/${id}`, { method: 'DELETE' });
      await load();
    } catch (e) {
      error = e.message;
    }
  }
</script>

<h1 class="page-title">染程</h1>
<p class="page-sub">
  仅 ready / dyeing 染缸可开缸；操作人恒为登录显示名（只读）；仅染坊主管可关闭染程，关闭后禁止追加色牢度抽检。
</p>

<div class="panel" style="margin-bottom:1rem;">
  <div class="form-grid">
    <label
      >染缸
      <select bind:value={form.vatId}>
        {#each vats as v}
          <option value={String(v.id)}
            >{v.vatCode} · {VAT_STATUS[v.status] || v.status} · {v.fiberType}</option
          >
        {/each}
      </select>
    </label>
    <label>配方名 <input bind:value={form.recipeName} /></label>
    <label>布料 kg <input type="number" step="0.1" bind:value={form.fabricKg} /></label>
    <label>开始时间 <input type="datetime-local" bind:value={form.startedAt} /></label>
    <label
      >操作员（登录显示名） <input value={form.operatorName} readonly title="操作人固定为登录显示名，不可修改" /></label
    >
  </div>
  <div class="toolbar">
    <button class="btn" type="button" on:click={save}>{editing ? '保存修改' : '新建染程'}</button>
    {#if editing}
      <button class="btn ghost" type="button" on:click={() => (editing = null)}>取消</button>
    {/if}
  </div>
  {#if error}<p class="err">{error}</p>{/if}
</div>

<div class="panel">
  <div class="toolbar" style="margin-bottom:0.75rem;">
    <label style="display:flex;align-items:center;gap:0.5rem;font-size:0.85rem;color:var(--indigo-mist);">
      状态筛选
      <select bind:value={closedFilter} on:change={load} style="min-width:7rem;">
        <option value="">全部</option>
        <option value="false">未关闭</option>
        <option value="true">已关闭</option>
      </select>
    </label>
  </div>
  <table>
    <thead>
      <tr>
        <th>ID</th>
        <th>染缸</th>
        <th>配方</th>
        <th>布料 kg</th>
        <th>开始</th>
        <th>操作员</th>
        <th>状态</th>
        <th></th>
      </tr>
    </thead>
    <tbody>
      {#each rows as row}
        <tr>
          <td>{row.id}</td>
          <td>{vatLabel(row.vatId)}</td>
          <td>{row.recipeName}</td>
          <td>{row.fabricKg}</td>
          <td>{new Date(row.startedAt).toLocaleString()}</td>
          <td>{row.operatorName}</td>
          <td>
            {#if row.closed}
              <span class="tag closed">已关闭</span>
              <div class="closed-meta">{row.closedBy || ''} · {new Date(row.closedAt).toLocaleString()}</div>
            {:else}
              <span class="tag open">未关闭</span>
            {/if}
          </td>
          <td class="row-actions">
            {#if isAdmin && !row.closed}
              <button class="btn small" type="button" on:click={() => closeLot(row)}>关闭</button>
            {/if}
            <button class="btn ghost small" type="button" on:click={() => startEdit(row)}>编辑</button>
            <button class="btn danger small" type="button" on:click={() => remove(row.id)}>删除</button>
          </td>
        </tr>
      {/each}
    </tbody>
  </table>
</div>

<style>
  .tag {
    display: inline-block;
    padding: 0.1rem 0.5rem;
    border-radius: 999px;
    font-size: 0.75rem;
    border: 1px solid var(--line);
  }

  .tag.open {
    color: var(--ok);
    border-color: rgba(76, 175, 130, 0.5);
    background: rgba(76, 175, 130, 0.12);
  }

  .tag.closed {
    color: var(--indigo-mist);
    background: rgba(107, 92, 231, 0.12);
  }

  .closed-meta {
    font-size: 0.7rem;
    color: var(--indigo-mist);
    margin-top: 0.2rem;
  }
</style>
