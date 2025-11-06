<script>
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';

  const positions = [
    { id: 'Presiden', name: 'Presiden' },
    { id: 'Timbalan Presiden', name: 'Timbalan Presiden' },
    { id: 'Naib Presiden I', name: 'Naib Presiden I' },
    { id: 'Naib Presiden II', name: 'Naib Presiden II' },
    { id: 'Exco 1', name: 'Exco 1' },
    { id: 'Exco 2', name: 'Exco 2' },
    { id: 'Exco 3', name: 'Exco 3' },
    { id: 'Exco 4', name: 'Exco 4' },
    { id: 'Exco 5', name: 'Exco 5' },
    { id: 'Exco 6', name: 'Exco 6' },
    { id: 'Exco 7', name: 'Exco 7' },
    { id: 'Pemeriksa Kira-kira 1', name: 'Pemeriksa Kira-kira 1' },
    { id: 'Pemeriksa Kira-kira 2', name: 'Pemeriksa Kira-kira 2' },
  ];

  const states = [
    'Johor', 'Kedah', 'Kelantan', 'Melaka', 'Negeri Sembilan',
    'Pahang', 'Perak', 'Perlis', 'Pulau Pinang', 'Sabah',
    'Sarawak', 'Selangor', 'Terengganu', 'WP Kuala Lumpur',
    'WP Labuan', 'WP Putrajaya'
  ];

  let nominatorName = '';
  let nominatorState = '';
  
  let nominations = {};
  let searchQueries = {};
  let suggestions = {};
  let showSuggestions = {};
  let loading = false;
  let searchingFor = {};
  let error = '';
  let success = false;

  positions.forEach(pos => {
    nominations[pos.id] = null;
    searchQueries[pos.id] = '';
    suggestions[pos.id] = [];
    showSuggestions[pos.id] = false;
    searchingFor[pos.id] = false;
  });

  let searchTimeout = {};

  async function searchVoters(positionId, query) {
    const trimmedQuery = query.trim();
    
    if (!trimmedQuery || trimmedQuery.length < 2) {
      suggestions[positionId] = [];
      showSuggestions[positionId] = false;
      return;
    }

    if (searchTimeout[positionId]) {
      clearTimeout(searchTimeout[positionId]);
    }

    searchTimeout[positionId] = setTimeout(async () => {
      searchingFor[positionId] = true;

      try {
        const { data, error: dbError } = await supabase
          .from('senarai_nama')
          .select('*')
          .ilike('nama', `%${trimmedQuery}%`)
          .order('nama')
          .limit(20);

        if (dbError) throw dbError;

        suggestions[positionId] = data || [];
        showSuggestions[positionId] = data && data.length > 0;
        
      } catch (err) {
        console.error('Search error:', err);
        suggestions[positionId] = [];
        showSuggestions[positionId] = false;
      } finally {
        searchingFor[positionId] = false;
      }
    }, 300);
  }

  function handleInputChange(positionId, event) {
    const query = event.target.value;
    searchQueries[positionId] = query;
    
    if (nominations[positionId] && nominations[positionId].nama !== query) {
      nominations[positionId] = null;
    }

    searchVoters(positionId, query);
  }

  function selectCandidate(positionId, voter) {
    nominations[positionId] = voter;
    searchQueries[positionId] = voter.nama;
    showSuggestions[positionId] = false;
    suggestions[positionId] = [];
  }

  function clearSelection(positionId) {
    nominations[positionId] = null;
    searchQueries[positionId] = '';
    suggestions[positionId] = [];
    showSuggestions[positionId] = false;
  }

  function handleBlur(positionId) {
    setTimeout(() => {
      showSuggestions[positionId] = false;
    }, 200);
  }

  async function handleSubmit() {
    if (!nominatorName.trim()) {
      error = 'Sila masukkan nama pencadang';
      return;
    }

    if (!nominatorState) {
      error = 'Sila pilih negeri bertugas';
      return;
    }

    const hasNominations = Object.values(nominations).some(nom => nom !== null);
    
    if (!hasNominations) {
      error = 'Sila pilih sekurang-kurangnya satu calon';
      return;
    }

    loading = true;
    error = '';

    try {
      const nominationsData = [];
      
      for (const [positionKey, candidate] of Object.entries(nominations)) {
        if (candidate !== null) {
          // Include ALL required fields based on database schema
          nominationsData.push({
            position: positionKey,
            candidate_name: candidate.nama,
            nominator_name: nominatorName.trim(),
            nominator_position: nominatorState,        // Negeri
            nominator_department: 'PPSM'               // ✅ REQUIRED FIELD - default value
          });
        }
      }

      console.log('=== SUBMITTING ===');
      console.log('Data:', nominationsData);
      console.log('Fields:', Object.keys(nominationsData[0]));

      const { data: insertedData, error: dbError } = await supabase
        .from('nominations')
        .insert(nominationsData)
        .select();

      if (dbError) {
        console.error('Database error:', dbError);
        throw dbError;
      }

      console.log('✅ SUCCESS:', insertedData);

      success = true;
      setTimeout(() => goto('/candidates'), 2000);

    } catch (err) {
      error = 'Gagal menghantar pencalonan: ' + err.message;
      console.error('❌ Error:', err);
    } finally {
      loading = false;
    }
  }

  function resetForm() {
    nominatorName = '';
    nominatorState = '';
    positions.forEach(pos => {
      nominations[pos.id] = null;
      searchQueries[pos.id] = '';
      suggestions[pos.id] = [];
      showSuggestions[pos.id] = false;
    });
    error = '';
  }
</script>

<svelte:head>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }
  </style>
</svelte:head>

<div style="min-height: 100vh; background: linear-gradient(135deg, #14b8a6 0%, #06b6d4 50%, #3b82f6 100%); padding: 2rem 1rem;">
  <div style="max-width: 900px; margin: 0 auto;">
    
    <div style="text-align: center; margin-bottom: 2rem;">
      <h1 style="font-size: 3rem; font-weight: 700; color: white; text-shadow: 2px 2px 4px rgba(0,0,0,0.2); letter-spacing: 0.05em;">
        e-VOTING PPSM
      </h1>
    </div>

    {#if success}
      <div style="background: white; border-radius: 24px; box-shadow: 0 20px 60px rgba(0,0,0,0.2); padding: 4rem; text-align: center;">
        <div style="font-size: 4rem; margin-bottom: 1.5rem;">✅</div>
        <h2 style="font-size: 2rem; font-weight: 700; color: #10b981; margin-bottom: 1rem;">Berjaya!</h2>
        <p style="color: #6b7280; font-size: 1.125rem;">Pencalonan telah dihantar</p>
      </div>
    {:else}
      <div style="background: white; border-radius: 24px; box-shadow: 0 20px 60px rgba(0,0,0,0.2); padding: 3rem;">
        
        <h2 style="font-size: 2rem; font-weight: 700; color: #111827; margin-bottom: 0.75rem;">Hantar Pencalonan</h2>
        <p style="color: #6b7280; line-height: 1.6; margin-bottom: 0.5rem;">Pilih calon untuk setiap jawatan</p>
        <p style="color: #6b7280; line-height: 1.6; margin-bottom: 0.5rem;">Sila pilih sekurang-kurangnya satu calon</p>
        <p style="color: #374151; line-height: 1.6; margin-bottom: 2rem;"><strong>Cara menggunakan:</strong> Taip nama calon (minimum 2 huruf). Sistem akan paparkan senarai nama. Klik nama untuk memilih.</p>

        {#if error}
          <div style="background: #fef2f2; border-left: 4px solid #ef4444; color: #991b1b; padding: 1rem 1.25rem; border-radius: 12px; margin-bottom: 2rem;">
            {error}
          </div>
        {/if}

        <form on:submit|preventDefault={handleSubmit} style="display: flex; flex-direction: column; gap: 2rem;">
          
          <div style="background: linear-gradient(to right, #ecfeff, #dbeafe); border: 2px solid #7dd3fc; border-radius: 16px; padding: 2rem;">
            <h3 style="font-size: 1.5rem; font-weight: 700; color: #0c4a6e; margin-bottom: 1.5rem; display: flex; align-items: center; gap: 0.5rem;">
              <span style="font-size: 1.75rem;">👤</span>
              Maklumat Pencadang
            </h3>
            
            <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 1.5rem;">
              <div>
                <label style="display: block; font-size: 1rem; font-weight: 600; color: #1f2937; margin-bottom: 0.5rem;">
                  Nama Pencadang <span style="color: #ef4444;">*</span>
                </label>
                <input
                  type="text"
                  bind:value={nominatorName}
                  placeholder="Masukkan nama anda..."
                  required
                  style="width: 100%; padding: 1rem 1.25rem; font-size: 1rem; border: 2px solid #cbd5e1; border-radius: 12px; background: white; font-family: inherit;"
                />
              </div>

              <div>
                <label style="display: block; font-size: 1rem; font-weight: 600; color: #1f2937; margin-bottom: 0.5rem;">
                  Negeri Bertugas <span style="color: #ef4444;">*</span>
                </label>
                <select
                  bind:value={nominatorState}
                  required
                  style="width: 100%; padding: 1rem 1.25rem; font-size: 1rem; border: 2px solid #cbd5e1; border-radius: 12px; background: white; font-family: inherit; cursor: pointer;"
                >
                  <option value="">-- Pilih Negeri --</option>
                  {#each states as state}
                    <option value={state}>{state}</option>
                  {/each}
                </select>
              </div>
            </div>
          </div>

          <div style="border-top: 2px dashed #cbd5e1;"></div>

          <h3 style="font-size: 1.5rem; font-weight: 700; color: #111827; display: flex; align-items: center; gap: 0.5rem;">
            <span style="font-size: 1.75rem;">🗳️</span>
            Pilih Calon
          </h3>

          {#each positions as position}
            <div>
              <label style="display: block; font-size: 1.125rem; font-weight: 600; color: #1f2937; margin-bottom: 0.75rem;">
                {position.name}
              </label>

              {#if !nominations[position.id]}
                <div style="position: relative;">
                  <input
                    type="text"
                    bind:value={searchQueries[position.id]}
                    on:input={(e) => handleInputChange(position.id, e)}
                    on:focus={() => {
                      if (searchQueries[position.id].length >= 2) {
                        searchVoters(position.id, searchQueries[position.id]);
                      }
                    }}
                    on:blur={() => handleBlur(position.id)}
                    placeholder="Taip nama calon..."
                    style="width: 100%; padding: 1.25rem 1.5rem; font-size: 1.125rem; border: 2px solid #d1d5db; border-radius: 16px; background: #f9fafb; font-family: inherit;"
                  />

                  {#if searchingFor[position.id]}
                    <div style="position: absolute; right: 1.25rem; top: 1.25rem;">
                      <div style="width: 1.5rem; height: 1.5rem; border: 3px solid #3b82f6; border-top-color: transparent; border-radius: 50%; animation: spin 1s linear infinite;"></div>
                    </div>
                  {/if}

                  {#if showSuggestions[position.id] && suggestions[position.id].length > 0}
                    <div style="position: absolute; z-index: 50; width: 100%; margin-top: 0.5rem; background: white; border: 2px solid #d1d5db; border-radius: 16px; box-shadow: 0 20px 40px rgba(0,0,0,0.15); max-height: 18rem; overflow-y: auto;">
                      {#each suggestions[position.id] as voter}
                        <button
                          type="button"
                          on:mousedown={() => selectCandidate(position.id, voter)}
                          style="width: 100%; text-align: left; padding: 1rem 1.5rem; border: none; background: transparent; cursor: pointer; border-bottom: 1px solid #e5e7eb; font-family: inherit;"
                          on:mouseenter={(e) => e.target.style.background = '#eff6ff'}
                          on:mouseleave={(e) => e.target.style.background = 'transparent'}
                        >
                          <div style="font-weight: 600; color: #111827; font-size: 1.125rem; margin-bottom: 0.25rem;">{voter.nama}</div>
                          <div style="font-size: 0.875rem; color: #6b7280;">
                            {#if voter.negeri}📍 {voter.negeri}{/if}
                            {#if voter.jabatan} • {voter.jabatan}{/if}
                          </div>
                        </button>
                      {/each}
                    </div>
                  {/if}
                </div>
              {:else}
                <div style="background: white; border: 2px solid #10b981; border-radius: 16px; padding: 1.25rem 1.5rem; display: flex; align-items: center; justify-content: space-between;">
                  <div style="display: flex; align-items: center; gap: 0.75rem;">
                    <span style="font-size: 1.5rem; color: #10b981;">✓</span>
                    <div>
                      <div style="font-weight: 700; color: #111827; font-size: 1.125rem; text-transform: uppercase; margin-bottom: 0.25rem;">
                        {nominations[position.id].nama}
                      </div>
                      {#if nominations[position.id].negeri}
                        <div style="font-size: 0.875rem; color: #6b7280;">
                          📍 {nominations[position.id].negeri}
                        </div>
                      {/if}
                    </div>
                  </div>
                  <button
                    type="button"
                    on:click={() => clearSelection(position.id)}
                    style="color: #9ca3af; font-size: 1.5rem; border: none; background: transparent; cursor: pointer; font-family: inherit;"
                    on:mouseenter={(e) => e.target.style.color = '#ef4444'}
                    on:mouseleave={(e) => e.target.style.color = '#9ca3af'}
                  >
                    ✕
                  </button>
                </div>
              {/if}
            </div>
          {/each}

          <div style="display: flex; gap: 1rem; padding-top: 1.5rem; border-top: 2px solid #e5e7eb;">
            <button
              type="submit"
              disabled={loading}
              style="flex: 1; background: linear-gradient(135deg, #14b8a6, #3b82f6); color: white; padding: 1.25rem 2rem; border-radius: 16px; font-size: 1.25rem; font-weight: 700; border: none; cursor: pointer; font-family: inherit; transition: all 0.2s; {loading ? 'opacity: 0.5; cursor: not-allowed;' : ''}"
              on:mouseenter={(e) => !loading && (e.target.style.transform = 'translateY(-2px)', e.target.style.boxShadow = '0 6px 20px rgba(59, 130, 246, 0.4)')}
              on:mouseleave={(e) => !loading && (e.target.style.transform = 'translateY(0)', e.target.style.boxShadow = 'none')}
            >
              {loading ? 'Menghantar...' : 'Hantar Pencalonan'}
            </button>
            <button
              type="button"
              on:click={resetForm}
              style="padding: 1.25rem 2rem; border: 2px solid #d1d5db; color: #374151; border-radius: 16px; font-size: 1.25rem; font-weight: 700; background: white; cursor: pointer; transition: all 0.2s; font-family: inherit;"
              on:mouseenter={(e) => (e.target.style.background = '#f9fafb', e.target.style.borderColor = '#9ca3af')}
              on:mouseleave={(e) => (e.target.style.background = 'white', e.target.style.borderColor = '#d1d5db')}
            >
              Reset
            </button>
          </div>
        </form>
      </div>
    {/if}

    <div style="text-align: center; margin-top: 2rem;">
      <a href="/" style="color: white; font-weight: 600; font-size: 1.125rem; text-decoration: none; text-shadow: 1px 1px 2px rgba(0,0,0,0.3);">
        ← Kembali ke Halaman Utama
      </a>
    </div>
  </div>
</div>

<style>
  @keyframes spin {
    from { transform: rotate(0deg); }
    to { transform: rotate(360deg); }
  }

  select {
    appearance: none;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12'%3E%3Cpath fill='%23333' d='M10.293 3.293L6 7.586 1.707 3.293A1 1 0 00.293 4.707l5 5a1 1 0 001.414 0l5-5a1 1 0 10-1.414-1.414z'/%3E%3C/svg%3E");
    background-repeat: no-repeat;
    background-position: right 1rem center;
    padding-right: 2.5rem;
  }
</style>