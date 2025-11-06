<script>
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let voters = [];
  let loading = true;
  let error = '';
  let success = '';
  
  let searchQuery = '';
  let filteredVoters = [];
  let currentPage = 1;
  let itemsPerPage = 20;

  let showEditModal = false;
  let editingVoter = null;
  let editForm = { nama: '', email: '', negeri: '', jabatan: '' };

  let showAddModal = false;
  let addForm = { nama: '', email: '', negeri: '', jabatan: '' };

  const states = [
    'Johor', 'Kedah', 'Kelantan', 'Melaka', 'Negeri Sembilan',
    'Pahang', 'Perak', 'Perlis', 'Pulau Pinang', 'Sabah',
    'Sarawak', 'Selangor', 'Terengganu', 'WP Kuala Lumpur',
    'WP Labuan', 'WP Putrajaya'
  ];

  onMount(() => {
    loadVoters();
  });

  async function loadVoters() {
    loading = true;
    try {
      const { data, error: dbError } = await supabase
        .from('senarai_nama')
        .select('*')
        .order('nama');

      if (dbError) throw dbError;
      voters = data || [];
      filterVoters();
    } catch (err) {
      error = 'Gagal memuatkan data pengundi: ' + err.message;
    } finally {
      loading = false;
    }
  }

  function filterVoters() {
    if (!searchQuery.trim()) {
      filteredVoters = voters;
    } else {
      const query = searchQuery.toLowerCase();
      filteredVoters = voters.filter(v => 
        v.nama?.toLowerCase().includes(query) ||
        v.email?.toLowerCase().includes(query) ||
        v.negeri?.toLowerCase().includes(query) ||
        v.jabatan?.toLowerCase().includes(query)
      );
    }
    currentPage = 1;
  }

  function openEditModal(voter) {
    editingVoter = voter;
    editForm = { ...voter };
    showEditModal = true;
  }

  function closeEditModal() {
    showEditModal = false;
    editingVoter = null;
    editForm = { nama: '', email: '', negeri: '', jabatan: '' };
  }

  function openAddModal() {
    addForm = { nama: '', email: '', negeri: '', jabatan: '' };
    showAddModal = true;
  }

  function closeAddModal() {
    showAddModal = false;
    addForm = { nama: '', email: '', negeri: '', jabatan: '' };
  }

  async function handleUpdate() {
    if (!editForm.nama) {
      error = 'Nama diperlukan';
      return;
    }

    try {
      const { error: dbError } = await supabase
        .from('senarai_nama')
        .update({
          nama: editForm.nama,
          email: editForm.email || null,
          negeri: editForm.negeri || null,
          jabatan: editForm.jabatan || null
        })
        .eq('id', editingVoter.id);

      if (dbError) throw dbError;

      success = 'Pengundi berjaya dikemaskini!';
      setTimeout(() => success = '', 3000);
      closeEditModal();
      loadVoters();
    } catch (err) {
      error = 'Gagal kemaskini: ' + err.message;
    }
  }

  async function handleAdd() {
    if (!addForm.nama) {
      error = 'Nama diperlukan';
      return;
    }

    try {
      const { error: dbError } = await supabase
        .from('senarai_nama')
        .insert([{
          nama: addForm.nama,
          email: addForm.email || null,
          negeri: addForm.negeri || null,
          jabatan: addForm.jabatan || null
        }]);

      if (dbError) throw dbError;

      success = 'Pengundi berjaya ditambah!';
      setTimeout(() => success = '', 3000);
      closeAddModal();
      loadVoters();
    } catch (err) {
      error = 'Gagal tambah: ' + err.message;
    }
  }

  async function handleDelete(voter) {
    if (!confirm(`Padam ${voter.nama}?`)) return;

    try {
      const { error: dbError } = await supabase
        .from('senarai_nama')
        .delete()
        .eq('id', voter.id);

      if (dbError) throw dbError;

      success = 'Pengundi berjaya dipadam!';
      setTimeout(() => success = '', 3000);
      loadVoters();
    } catch (err) {
      error = 'Gagal padam: ' + err.message;
    }
  }

  $: paginatedVoters = filteredVoters.slice(
    (currentPage - 1) * itemsPerPage,
    currentPage * itemsPerPage
  );
  $: totalPages = Math.ceil(filteredVoters.length / itemsPerPage);
</script>

<svelte:head>
  <style>
    body {
      overflow-y: auto !important;
    }
  </style>
</svelte:head>

<div style="padding-bottom: 3rem;">
  <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 2rem;">
    <h1 style="font-size: 2rem; font-weight: 700; color: #111827;">
      👥 Kemaskini Pengundi
    </h1>
    <button
      on:click={openAddModal}
      style="background: #10b981; color: white; padding: 0.75rem 1.5rem; border-radius: 12px; font-weight: 600; border: none; cursor: pointer; font-family: inherit;"
    >
      ➕ Tambah Pengundi
    </button>
  </div>

  {#if success}
    <div style="background: #d1fae5; border-left: 4px solid #10b981; color: #065f46; padding: 1rem; border-radius: 8px; margin-bottom: 1rem;">
      ✅ {success}
    </div>
  {/if}

  {#if error}
    <div style="background: #fee2e2; border-left: 4px solid #ef4444; color: #991b1b; padding: 1rem; border-radius: 8px; margin-bottom: 1rem;">
      ⚠️ {error}
    </div>
  {/if}

  <div style="margin-bottom: 1.5rem;">
    <input
      type="text"
      bind:value={searchQuery}
      on:input={filterVoters}
      placeholder="🔍 Cari pengundi (nama, email, negeri, jabatan)..."
      style="width: 100%; padding: 1rem; border: 2px solid #d1d5db; border-radius: 12px; font-size: 1rem; font-family: inherit;"
    />
  </div>

  <div style="background: white; padding: 1rem; border-radius: 12px; margin-bottom: 1.5rem; box-shadow: 0 1px 3px rgba(0,0,0,0.1);">
    <strong>Jumlah: {filteredVoters.length} pengundi</strong>
    {#if searchQuery}
      <span style="color: #6b7280;"> (ditapis dari {voters.length})</span>
    {/if}
  </div>

  {#if loading}
    <div style="text-align: center; padding: 3rem; color: #6b7280;">
      ⏳ Memuatkan data...
    </div>
  {:else if paginatedVoters.length === 0}
    <div style="text-align: center; padding: 3rem; color: #6b7280;">
      Tiada pengundi dijumpai
    </div>
  {:else}
    <div style="background: white; border-radius: 12px; overflow: hidden; box-shadow: 0 1px 3px rgba(0,0,0,0.1); margin-bottom: 2rem;">
      <table style="width: 100%; border-collapse: collapse;">
        <thead style="background: #f3f4f6;">
          <tr>
            <th style="padding: 1rem; text-align: left; font-weight: 600; color: #374151; border-bottom: 2px solid #e5e7eb;">Nama</th>
            <th style="padding: 1rem; text-align: left; font-weight: 600; color: #374151; border-bottom: 2px solid #e5e7eb;">Email</th>
            <th style="padding: 1rem; text-align: left; font-weight: 600; color: #374151; border-bottom: 2px solid #e5e7eb;">Negeri</th>
            <th style="padding: 1rem; text-align: left; font-weight: 600; color: #374151; border-bottom: 2px solid #e5e7eb;">Jabatan</th>
            <th style="padding: 1rem; text-align: center; font-weight: 600; color: #374151; border-bottom: 2px solid #e5e7eb;">Tindakan</th>
          </tr>
        </thead>
        <tbody>
          {#each paginatedVoters as voter}
            <tr style="border-bottom: 1px solid #e5e7eb;">
              <td style="padding: 1rem; font-weight: 600;">{voter.nama}</td>
              <td style="padding: 1rem; color: #6b7280;">{voter.email || '-'}</td>
              <td style="padding: 1rem; color: #6b7280;">{voter.negeri || '-'}</td>
              <td style="padding: 1rem; color: #6b7280;">{voter.jabatan || '-'}</td>
              <td style="padding: 1rem; text-align: center;">
                <button
                  on:click={() => openEditModal(voter)}
                  style="background: #3b82f6; color: white; padding: 0.5rem 1rem; border-radius: 8px; border: none; cursor: pointer; margin-right: 0.5rem; font-family: inherit;"
                >
                  ✏️ Edit
                </button>
                <button
                  on:click={() => handleDelete(voter)}
                  style="background: #ef4444; color: white; padding: 0.5rem 1rem; border-radius: 8px; border: none; cursor: pointer; font-family: inherit;"
                >
                  🗑️ Padam
                </button>
              </td>
            </tr>
          {/each}
        </tbody>
      </table>
    </div>

    {#if totalPages > 1}
      <div style="display: flex; justify-content: center; align-items: center; gap: 1rem; margin-top: 2rem; margin-bottom: 2rem;">
        <button
          on:click={() => currentPage = Math.max(1, currentPage - 1)}
          disabled={currentPage === 1}
          style="padding: 0.5rem 1rem; border: 2px solid #d1d5db; border-radius: 8px; background: white; cursor: pointer; font-family: inherit; {currentPage === 1 ? 'opacity: 0.5; cursor: not-allowed;' : ''}"
        >
          ← Sebelum
        </button>
        <span style="color: #6b7280;">
          Halaman {currentPage} / {totalPages}
        </span>
        <button
          on:click={() => currentPage = Math.min(totalPages, currentPage + 1)}
          disabled={currentPage === totalPages}
          style="padding: 0.5rem 1rem; border: 2px solid #d1d5db; border-radius: 8px; background: white; cursor: pointer; font-family: inherit; {currentPage === totalPages ? 'opacity: 0.5; cursor: not-allowed;' : ''}"
        >
          Seterusnya →
        </button>
      </div>
    {/if}
  {/if}
</div>

<!-- Modals remain the same -->
{#if showEditModal}
  <div style="position: fixed; inset: 0; background: rgba(0,0,0,0.5); display: flex; align-items: center; justify-content: center; z-index: 1000;" on:click={closeEditModal}>
    <div style="background: white; padding: 2rem; border-radius: 16px; max-width: 500px; width: 90%;" on:click|stopPropagation>
      <h2 style="font-size: 1.5rem; font-weight: 700; margin-bottom: 1.5rem;">Edit Pengundi</h2>
      
      <div style="display: flex; flex-direction: column; gap: 1rem;">
        <div>
          <label style="display: block; font-weight: 600; margin-bottom: 0.5rem;">Nama *</label>
          <input type="text" bind:value={editForm.nama} style="width: 100%; padding: 0.75rem; border: 2px solid #d1d5db; border-radius: 8px; font-family: inherit;" />
        </div>
        <div>
          <label style="display: block; font-weight: 600; margin-bottom: 0.5rem;">Email</label>
          <input type="email" bind:value={editForm.email} style="width: 100%; padding: 0.75rem; border: 2px solid #d1d5db; border-radius: 8px; font-family: inherit;" />
        </div>
        <div>
          <label style="display: block; font-weight: 600; margin-bottom: 0.5rem;">Negeri</label>
          <select bind:value={editForm.negeri} style="width: 100%; padding: 0.75rem; border: 2px solid #d1d5db; border-radius: 8px; font-family: inherit;">
            <option value="">-- Pilih Negeri --</option>
            {#each states as state}
              <option value={state}>{state}</option>
            {/each}
          </select>
        </div>
        <div>
          <label style="display: block; font-weight: 600; margin-bottom: 0.5rem;">Jabatan</label>
          <input type="text" bind:value={editForm.jabatan} style="width: 100%; padding: 0.75rem; border: 2px solid #d1d5db; border-radius: 8px; font-family: inherit;" />
        </div>
      </div>

      <div style="display: flex; gap: 1rem; margin-top: 2rem;">
        <button on:click={handleUpdate} style="flex: 1; background: #10b981; color: white; padding: 0.75rem; border-radius: 8px; font-weight: 600; border: none; cursor: pointer; font-family: inherit;">
          ✅ Simpan
        </button>
        <button on:click={closeEditModal} style="flex: 1; background: #6b7280; color: white; padding: 0.75rem; border-radius: 8px; font-weight: 600; border: none; cursor: pointer; font-family: inherit;">
          ❌ Batal
        </button>
      </div>
    </div>
  </div>
{/if}

{#if showAddModal}
  <div style="position: fixed; inset: 0; background: rgba(0,0,0,0.5); display: flex; align-items: center; justify-content: center; z-index: 1000;" on:click={closeAddModal}>
    <div style="background: white; padding: 2rem; border-radius: 16px; max-width: 500px; width: 90%;" on:click|stopPropagation>
      <h2 style="font-size: 1.5rem; font-weight: 700; margin-bottom: 1.5rem;">Tambah Pengundi</h2>
      
      <div style="display: flex; flex-direction: column; gap: 1rem;">
        <div>
          <label style="display: block; font-weight: 600; margin-bottom: 0.5rem;">Nama *</label>
          <input type="text" bind:value={addForm.nama} style="width: 100%; padding: 0.75rem; border: 2px solid #d1d5db; border-radius: 8px; font-family: inherit;" />
        </div>
        <div>
          <label style="display: block; font-weight: 600; margin-bottom: 0.5rem;">Email</label>
          <input type="email" bind:value={addForm.email} style="width: 100%; padding: 0.75rem; border: 2px solid #d1d5db; border-radius: 8px; font-family: inherit;" />
        </div>
        <div>
          <label style="display: block; font-weight: 600; margin-bottom: 0.5rem;">Negeri</label>
          <select bind:value={addForm.negeri} style="width: 100%; padding: 0.75rem; border: 2px solid #d1d5db; border-radius: 8px; font-family: inherit;">
            <option value="">-- Pilih Negeri --</option>
            {#each states as state}
              <option value={state}>{state}</option>
            {/each}
          </select>
        </div>
        <div>
          <label style="display: block; font-weight: 600; margin-bottom: 0.5rem;">Jabatan</label>
          <input type="text" bind:value={addForm.jabatan} style="width: 100%; padding: 0.75rem; border: 2px solid #d1d5db; border-radius: 8px; font-family: inherit;" />
        </div>
      </div>

      <div style="display: flex; gap: 1rem; margin-top: 2rem;">
        <button on:click={handleAdd} style="flex: 1; background: #10b981; color: white; padding: 0.75rem; border-radius: 8px; font-weight: 600; border: none; cursor: pointer; font-family: inherit;">
          ✅ Tambah
        </button>
        <button on:click={closeAddModal} style="flex: 1; background: #6b7280; color: white; padding: 0.75rem; border-radius: 8px; font-weight: 600; border: none; cursor: pointer; font-family: inherit;">
          ❌ Batal
        </button>
      </div>
    </div>
  </div>
{/if}