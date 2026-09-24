<script>
  import { onMount } from 'svelte';
  import { api, VAT_STATUS, toLocalInput, fromLocalInput } from '../lib/api.js';
  import { user } from '../lib/auth.js';

  let vats = [];
  let rows = [];
  let error = '';
  // all=全部 open=仅未关闭 closed=仅已关闭
  let filter = 'all';
  let form = {
    vatId: '',
    recipeName: '',
    fabricKg: 20,
    startedAt: toLocalInput(new Date().toISOString()),
  };
  let editing = null;

  $: isSupervisor = $user?.role === 'admin';
  $: displayName = $user?.displayName || '';

  async function load() {
    error = '';
    try {
      const closedParam = filter === 'all' ? '' : `?closed=${filter === 'closed'}`;
      const [vatData, lotData] = await Promise.all([
        api('/vats'),
        api(`/dye-lots${closedParam}`),
      ]);
      vats = vatData;
      rows = lotData;
      const usable = vats.filter((v) => v.status === 'ready' || v.status === 'dyeing');
      if (!form.vatId && usable.length) form.vatId = String(usable[0].id);
      else if (!form.vatId && vats.length) form.vatId = String(vats[0].id);
    } catch (e) {
      error = e.message;
    }
  }

  onMount(load);

  function switchFilter(value) {
    filter = value;
    load();
  }

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
        // 更新接口不接收操作人：operatorName 创建后不可更改
        await api(`/dye-lots/${editing}`, { method: 'PUT', body: JSON.stringify(body) });
      } else {
        // 操作人恒为登录显示名，输入框只读；后端会再次强校验
        body.operatorName = displayName;
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
  仅 ready / dyeing 染缸可开缸；操作员可开立染程但不可关闭，关闭收口仅主管可执行；关闭后禁止再追加色牢度。
</p>

<div class="panel" style="margin-bottom:1rem;">
  <div class="form-grid">
    <label
      >染缸
      <select bind:value={form.vatId} disabled={editing !== null}>
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
    <label>操作员 <input value={displayName} readonly title="操作人恒为登录显示名" /></label>
  </div>
  <div class="toolbar">
    <button class="btn" type="button" on:click={save}>{editing ? '保存修改' : '新建染程'}</button>
    {#if editing}
      <button class="btn ghost" type="button" on:click={() => (editing = null)}>取消</button>
    {/if}
    <span class="role-hint"
      >当前角色：{isSupervisor ? '染坊主管（可关闭染程）' : '染程操作员（可开立，不可关闭）'}</span
    >
  </div>
  {#if error}<p class="err">{error}</p>{/if}
</div>

<div class="panel">
  <div class="toolbar" style="margin-bottom:0.75rem;">
    <div class="seg">
      <button
        type="button"
        class="btn ghost small"
        class:on={filter === 'all'}
        on:click={() => switchFilter('all')}>全部</button
      >
      <button
        type="button"
        class="btn ghost small"
        class:on={filter === 'open'}
        on:click={() => switchFilter('open')}>未关闭</button
      >
      <button
        type="button"
        class="btn ghost small"
        class:on={filter === 'closed'}
        on:click={() => switchFilter('closed')}>已关闭</button
      >
    </div>
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
        <tr class:closed-row={row.closed}>
          <td>{row.id}</td>
          <td>{vatLabel(row.vatId)}</td>
          <td>{row.recipeName}</td>
          <td>{row.fabricKg}</td>
          <td>{new Date(row.startedAt).toLocaleString()}</td>
          <td>{row.operatorName}</td>
          <td>
            {#if row.closed}
              <span class="tag-closed">已关闭</span>
              {#if row.closedAt}<span class="closed-at">{new Date(row.closedAt).toLocaleString()}</span>{/if}
            {:else}
              <span class="tag-open">未关闭</span>
            {/if}
          </td>
          <td class="row-actions">
            <button class="btn ghost small" type="button" on:click={() => startEdit(row)}>编辑</button>
            <!-- 关闭按钮仅前端按角色隐藏；后端对操作员强制 403 -->
            {#if isSupervisor && !row.closed}
              <button class="btn small" type="button" on:click={() => closeLot(row)}>关闭</button>
            {/if}
            <button class="btn danger small" type="button" on:click={() => remove(row.id)}>删除</button>
          </td>
        </tr>
      {/each}
    </tbody>
  </table>
</div>

<style>
  .role-hint {
    color: var(--indigo-mist);
    font-size: 0.8rem;
  }

  .seg {
    display: inline-flex;
    gap: 0.25rem;
  }

  .seg .btn.on {
    background: rgba(107, 92, 231, 0.35);
    border-color: rgba(107, 92, 231, 0.55);
    color: white;
  }

  .tag-open {
    display: inline-block;
    padding: 0.1rem 0.5rem;
    border-radius: 2px;
    font-size: 0.75rem;
    color: var(--ok);
    border: 1px solid rgba(76, 175, 130, 0.5);
    background: rgba(76, 175, 130, 0.12);
  }

  .tag-closed {
    display: inline-block;
    padding: 0.1rem 0.5rem;
    border-radius: 2px;
    font-size: 0.75rem;
    color: var(--indigo-mist);
    border: 1px solid var(--line);
    background: rgba(255, 255, 255, 0.04);
  }

  .closed-at {
    display: block;
    font-size: 0.7rem;
    color: var(--indigo-mist);
    margin-top: 0.15rem;
  }

  .closed-row {
    opacity: 0.65;
  }
</style>
