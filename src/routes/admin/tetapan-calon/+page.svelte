<script>
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let positions = [];
  let candidates = {};
  let loading = true;
  let error = '';
  let success = '';
  let saving = false;

  // Modal states
  let showAddModal = false;
  let selectedPosition = '';
  let newCandidateName = '';

  const positionList = [
    { name: 'Presiden', max: 1, icon: '👑' },
    { name: 'Timbalan Presiden', max: 1, icon: '🥈' },
    { name: 'Naib Presiden', max: 1, icon: '🥉' },
    { name: 'Exco', max: 7, icon: '💼' },
    { name: 'Pemeriksa Kira-kira', max: 2, icon: '📊' }
  ];

  onMount(() => {
    loadData();
  });

  async function loadData() {
    loading = true;
    try {
      // Load position settings
      const { data: posData, error: posError } = await supabase
        .from('position_settings')
        .select('*')
        .order('position');

      if (posError) throw posError;
      positions = posData || [];

      // Load candidates
      const { data: candData, error: candError } = await supabase
        .from('voting_candidates')
        .select('*')
        .order('position')
        .order('display_order');

      if (candError) throw candError;
      
      // Group candidates by position
      candidates = {};
      (candData || []).forEach(cand => {
        if (!candidates[cand.position]) {
          candidates[cand.position] = [];
        }
        candidates[cand.position].push(cand);
      });

    } catch (err) {
      error = 'Gagal memuatkan data: ' + err.message;
      console.error(err);
    } finally {
      loading = false;
    }
  }

  async function togglePosition(position) {
    try {
      const currentSetting = positions.find(p => p.position === position);
      const newEnabled = !currentSetting.is_enabled;

      const { error: dbError } = await supabase
        .from('position_settings')
        .update({ is_enabled: newEnabled })
        .eq('position', position);

      if (dbError) throw dbError;

      success = `Jawatan ${position} ${newEnabled ? 'diaktifkan' : 'dinyahaktifkan'}!`;
      setTimeout(() => success = '', 3000);
      loadData();
    } catch (err) {
      error = 'Gagal kemaskini: ' + err.message;
    }
  }

  function openAddModal(position) {
    selectedPosition = position;
    newCandidateName = '';
    showAddModal = true;
  }

  function closeAddModal() {
    showAddModal = false;
    selectedPosition = '';
    newCandidateName = '';
  }

  async function addCandidate() {
    if (!newCandidateName.trim()) {
      error = 'Sila masukkan nama calon';
      return;
    }

    try {
      const existingCandidates = candidates[selectedPosition] || [];
      const displayOrder = existingCandidates.length;

      const { error: dbError } = await supabase
        .from('voting_candidates')
        .insert([{
          position: selectedPosition,
          candidate_name: newCandidateName.trim(),
          display_order: displayOrder,
          is_enabled: true
        }]);

      if (dbError) throw dbError;

      success = 'Calon berjaya ditambah!';
      setTimeout(() => success = '', 3000);
      closeAddModal();
      loadData();
    } catch (err) {
      error = 'Gagal tambah calon: ' + err.message;
    }
  }

  async function removeCandidate(candidateId) {
    if (!confirm('Padam calon ini?')) return;

    try {
      const { error: dbError } = await supabase
        .from('voting_candidates')
        .delete()
        .eq('id', candidateId);

      if (dbError) throw dbError;

      success = 'Calon berjaya dipadam!';
      setTimeout(() => success = '', 3000);
      loadData();
    } catch (err) {
      error = 'Gagal padam calon: ' + err.message;
    }
  }

  function getPositionSetting(position) {
    return positions.find(p => p.position === position) || { is_enabled: false };
  }

  function getPositionIcon(position) {
    const pos = positionList.find(p => p.name === position);
    return pos ? pos.icon : '🎯';
  }
</script>

<svelte:head>
  <style>
    body {
      overflow-y: auto !important;
    }
  </style>
</svelte:head>

<div style="padding-bottom: 3rem;">
  
  <div style="margin-bottom: 2rem;">
    <h1 style="font-size: 2rem; font-weight: 700; color: #111827; margin-bottom: 0.5rem;">
      🎯 Tetapan Calon Untuk Undian
    </h1>
    <p style="color: #6b7280;">Tetapkan calon yang akan dipertandingkan untuk setiap jawatan dalam pengundian</p>
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

  {#if loading}
    <div style="text-align: center; padding: 3rem; color: #6b7280;">
      ⏳ Memuatkan data...
    </div>
  {:else}
    <!-- Positions -->
    {#each positionList as pos}
      {@const setting = getPositionSetting(pos.name)}
      {@const positionCandidates = candidates[pos.name] || []}
      {@const isEnabled = setting.is_enabled}
      
      <div style="background: white; border-radius: 16px; padding: 1.5rem; margin-bottom: 1.5rem; box-shadow: 0 2px 4px rgba(0,0,0,0.1); border: 2px solid {isEnabled ? '#10b981' : '#e5e7eb'};">
        
        <!-- Position Header with Toggle -->
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 1.5rem; padding-bottom: 1rem; border-bottom: 2px solid #e5e7eb;">
          <div style="display: flex; align-items: center; gap: 1rem;">
            <span style="font-size: 2rem;">{pos.icon}</span>
            <div>
              <h2 style="font-size: 1.5rem; font-weight: 700; color: #111827; margin-bottom: 0.25rem;">
                {pos.name}
              </h2>
              <p style="font-size: 0.875rem; color: #6b7280;">
                Maksimum: {pos.max} {pos.max > 1 ? 'calon' : 'calon'}
              </p>
            </div>
          </div>

          <!-- Enable/Disable Toggle -->
          <label style="display: flex; align-items: center; gap: 0.75rem; cursor: pointer; padding: 0.75rem 1.25rem; background: {isEnabled ? '#d1fae5' : '#f3f4f6'}; border-radius: 12px; border: 2px solid {isEnabled ? '#10b981' : '#d1d5db'};">
            <input
              type="checkbox"
              checked={isEnabled}
              on:change={() => togglePosition(pos.name)}
              style="width: 1.25rem; height: 1.25rem; cursor: pointer;"
            />
            <span style="font-weight: 600; color: {isEnabled ? '#065f46' : '#6b7280'};">
              {isEnabled ? '✅ Diaktifkan' : 'Tidak Aktif'}
            </span>
          </label>
        </div>

        <!-- Candidates List -->
        <div style="margin-bottom: 1rem;">
          <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 1rem;">
            <h3 style="font-size: 1.125rem; font-weight: 600; color: #374151;">
              Senarai Calon ({positionCandidates.length})
            </h3>
            <button
              on:click={() => openAddModal(pos.name)}
              style="background: #3b82f6; color: white; padding: 0.5rem 1rem; border-radius: 8px; font-weight: 600; border: none; cursor: pointer; font-family: inherit; font-size: 0.875rem;"
            >
              ➕ Tambah Calon
            </button>
          </div>

          {#if positionCandidates.length === 0}
            <div style="text-align: center; padding: 2rem; background: #f9fafb; border-radius: 12px; border: 2px dashed #d1d5db;">
              <p style="color: #9ca3af; font-style: italic;">
                Tiada calon ditetapkan. Klik "Tambah Calon" untuk menambah.
              </p>
            </div>
          {:else}
            <div style="display: flex; flex-direction: column; gap: 0.75rem;">
              {#each positionCandidates as candidate}
                <div style="display: flex; justify-content: space-between; align-items: center; padding: 1rem; background: #f9fafb; border-radius: 12px; border: 1px solid #e5e7eb;">
                  <div style="display: flex; align-items: center; gap: 0.75rem;">
                    <span style="font-size: 1.5rem;">👤</span>
                    <span style="font-weight: 600; font-size: 1.125rem; color: #111827; text-transform: uppercase;">
                      {candidate.candidate_name}
                    </span>
                  </div>
                  <button
                    on:click={() => removeCandidate(candidate.id)}
                    style="color: #ef4444; background: transparent; border: none; cursor: pointer; font-size: 1.25rem; padding: 0.25rem 0.5rem; font-family: inherit;"
                    title="Padam calon"
                  >
                    🗑️
                  </button>
                </div>
              {/each}
            </div>
          {/if}
        </div>

        <!-- Info -->
        {#if isEnabled && positionCandidates.length === 0}
          <div style="background: #fef3c7; border-left: 4px solid #f59e0b; padding: 0.75rem 1rem; border-radius: 8px;">
            <span style="color: #92400e; font-size: 0.875rem;">
              ⚠️ Jawatan ini diaktifkan tetapi tiada calon. Sila tambah calon.
            </span>
          </div>
        {/if}
      </div>
    {/each}

    <!-- Info Box -->
    <div style="background: #eff6ff; border-left: 4px solid #3b82f6; padding: 1.25rem; border-radius: 12px; margin-top: 2rem;">
      <div style="font-weight: 600; color: #1e40af; margin-bottom: 0.75rem; font-size: 1.125rem;">
        ℹ️ Panduan:
      </div>
      <ul style="color: #1e3a8a; margin-left: 1.5rem; line-height: 2;">
        <li>Tandakan kotak untuk mengaktifkan jawatan yang akan dibuka untuk pengundian</li>
        <li>Tambah calon dengan klik butang "Tambah Calon" pada setiap jawatan</li>
        <li>Tiada had bilangan calon yang boleh ditambah</li>
        <li>Pengundi hanya boleh mengundi jawatan yang diaktifkan sahaja</li>
        <li>Pastikan semua calon ditambah sebelum mengaktifkan pengundian</li>
      </ul>
    </div>
  {/if}
</div>

<!-- Add Candidate Modal -->
{#if showAddModal}
  <div style="position: fixed; inset: 0; background: rgba(0,0,0,0.5); display: flex; align-items: center; justify-content: center; z-index: 1000;" on:click={closeAddModal}>
    <div style="background: white; padding: 2rem; border-radius: 16px; max-width: 500px; width: 90%;" on:click|stopPropagation>
      <h2 style="font-size: 1.5rem; font-weight: 700; margin-bottom: 0.5rem;">
        Tambah Calon
      </h2>
      <p style="color: #6b7280; margin-bottom: 1.5rem;">
        Jawatan: <strong>{selectedPosition}</strong>
      </p>
      
      <div>
        <label style="display: block; font-weight: 600; color: #374151; margin-bottom: 0.5rem;">
          Nama Calon *
        </label>
        <input
          type="text"
          bind:value={newCandidateName}
          placeholder="Masukkan nama calon..."
          on:keypress={(e) => e.key === 'Enter' && addCandidate()}
          style="width: 100%; padding: 0.875rem; border: 2px solid #d1d5db; border-radius: 8px; font-size: 1rem; font-family: inherit; text-transform: uppercase;"
        />
      </div>

      <div style="display: flex; gap: 1rem; margin-top: 2rem;">
        <button
          on:click={addCandidate}
          style="flex: 1; background: #10b981; color: white; padding: 0.875rem; border-radius: 12px; font-weight: 600; border: none; cursor: pointer; font-family: inherit; font-size: 1rem;"
        >
          ✅ Tambah
        </button>
        <button
          on:click={closeAddModal}
          style="flex: 1; background: #6b7280; color: white; padding: 0.875rem; border-radius: 12px; font-weight: 600; border: none; cursor: pointer; font-family: inherit; font-size: 1rem;"
        >
          ❌ Batal
        </button>
      </div>
    </div>
  </div>
{/if}