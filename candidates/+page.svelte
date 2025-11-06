<script>
  import { onMount } from 'svelte';
  import { goto } from '$app/navigation';
  import { browser } from '$app/environment';
  import { supabase } from '$lib/supabaseClient';

  let adminSession = null;
  let loading = false;
  let error = '';
  let success = '';

  // Form data
  let candidates = {
    president: ['', '', ''],
    vicePresident: ['', '', '', '', ''],
    exco: ['', '', '', '', '', '', '', '', '', '', '', '', '', '', '']
  };

  onMount(async () => {
    if (browser) {
      const session = localStorage.getItem('admin_session');
      if (!session) {
        goto('/admin');
        return;
      }
      adminSession = JSON.parse(session);
    }

    await loadExistingCandidates();
  });

  async function loadExistingCandidates() {
    loading = true;

    try {
      const { data, error: dbError } = await supabase
        .from('voting_candidates')
        .select('*')
        .order('display_order');

      if (dbError) throw dbError;

      if (data && data.length > 0) {
        // Group by position
        const presidentData = data.filter(c => c.position === 'president');
        const vicePresidentData = data.filter(c => c.position === 'vice_president');
        const excoData = data.filter(c => c.position === 'exco');

        // Load president
        presidentData.forEach((c, i) => {
          if (i < 3) {
            candidates.president[i] = c.candidate_name;
          }
        });

        // Load vice president
        vicePresidentData.forEach((c, i) => {
          if (i < 5) {
            candidates.vicePresident[i] = c.candidate_name;
          }
        });

        // Load exco
        excoData.forEach((c, i) => {
          if (i < 15) {
            candidates.exco[i] = c.candidate_name;
          }
        });
      }

      loading = false;
    } catch (err) {
      console.error('Error loading candidates:', err);
      loading = false;
    }
  }

  async function saveCandidates() {
    loading = true;
    error = '';
    success = '';

    try {
      // Delete existing candidates
      await supabase.from('voting_candidates').delete().neq('id', '00000000-0000-0000-0000-000000000000');

      // Prepare data to insert
      const candidatesToInsert = [];

      // President (3 slots)
      candidates.president.forEach((name, index) => {
        if (name.trim()) {
          candidatesToInsert.push({
            position: 'president',
            candidate_name: name.trim(),
            candidate_email: null,
            display_order: index + 1
          });
        }
      });

      // Vice President (5 slots)
      candidates.vicePresident.forEach((name, index) => {
        if (name.trim()) {
          candidatesToInsert.push({
            position: 'vice_president',
            candidate_name: name.trim(),
            candidate_email: null,
            display_order: index + 1
          });
        }
      });

      // Exco (15 slots)
      candidates.exco.forEach((name, index) => {
        if (name.trim()) {
          candidatesToInsert.push({
            position: 'exco',
            candidate_name: name.trim(),
            candidate_email: null,
            display_order: index + 1
          });
        }
      });

      if (candidatesToInsert.length === 0) {
        error = 'Sila isi sekurang-kurangnya satu calon';
        loading = false;
        return;
      }

      // Insert all candidates
      const { error: insertError } = await supabase
        .from('voting_candidates')
        .insert(candidatesToInsert);

      if (insertError) throw insertError;

      success = 'Calon berjaya disimpan! Calon akan dipaparkan di page "Undi Sekarang".';
      loading = false;
      window.scrollTo({ top: 0, behavior: 'smooth' });

    } catch (err) {
      error = 'Gagal simpan: ' + err.message;
      loading = false;
    }
  }

  function clearAllFields() {
    if (confirm('Adakah anda pasti untuk clear semua field?')) {
      candidates = {
        president: ['', '', ''],
        vicePresident: ['', '', '', '', ''],
        exco: ['', '', '', '', '', '', '', '', '', '', '', '', '', '', '']
      };
    }
  }
</script>

<div class="min-h-screen bg-gradient-to-br from-gray-50 to-purple-100">
  <!-- Header -->
  <div class="bg-gradient-to-r from-purple-600 to-pink-600 shadow-lg">
    <div class="max-w-7xl mx-auto px-4 py-6">
      <div class="flex items-center gap-4">
        <a href="/admin/dashboard" class="text-white hover:text-purple-200 transition">
          <svg class="w-8 h-8" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 19l-7-7m0 0l7-7m-7 7h18"/>
          </svg>
        </a>
        <div>
          <h1 class="text-2xl font-bold text-white">Masukkan Calon Undi</h1>
          <p class="text-purple-200 text-sm">Tetapkan calon untuk pengundian online</p>
        </div>
      </div>
    </div>
  </div>

  <div class="max-w-6xl mx-auto px-4 py-8">
    {#if error}
      <div class="bg-red-100 border-l-4 border-red-500 text-red-700 px-4 py-3 rounded-lg mb-6">
        <strong>⚠️ Ralat:</strong> {error}
      </div>
    {/if}

    {#if success}
      <div class="bg-green-100 border-l-4 border-green-500 text-green-700 px-4 py-3 rounded-lg mb-6">
        <strong>✅ Berjaya!</strong> {success}
      </div>
    {/if}

    <!-- Info Box -->
    <div class="bg-blue-50 border-l-4 border-blue-500 p-6 rounded-lg mb-6">
      <div class="flex items-start gap-3">
        <svg class="w-6 h-6 text-blue-500 flex-shrink-0 mt-1" fill="currentColor" viewBox="0 0 20 20">
          <path fill-rule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-7-4a1 1 0 11-2 0 1 1 0 012 0zM9 9a1 1 0 000 2v3a1 1 0 001 1h1a1 1 0 100-2v-3a1 1 0 00-1-1H9z" clip-rule="evenodd"/>
        </svg>
        <div>
          <h3 class="font-semibold text-blue-800 mb-2">Arahan:</h3>
          <ul class="text-sm text-blue-700 space-y-1">
            <li>• Masukkan nama calon untuk setiap jawatan</li>
            <li>• Calon yang dimasukkan akan dipaparkan di page <strong>"Undi Sekarang"</strong></li>
            <li>• Field kosong tidak akan disimpan</li>
          </ul>
        </div>
      </div>
    </div>

    <form on:submit|preventDefault={saveCandidates} class="space-y-6">
      
      <!-- President Section -->
      <div class="bg-white rounded-xl shadow-lg overflow-hidden">
        <div class="bg-gradient-to-r from-purple-600 to-indigo-600 px-6 py-4">
          <h2 class="text-xl font-bold text-white flex items-center">
            <span class="bg-white text-purple-600 rounded-full w-8 h-8 flex items-center justify-center mr-3 text-sm">3</span>
            Presiden
          </h2>
          <p class="text-purple-100 text-sm mt-1">Masukkan maksimum 3 nama calon</p>
        </div>
        
        <div class="p-6 space-y-4">
          {#each [0, 1, 2] as index}
            <div>
              <label class="block text-sm font-medium text-gray-700 mb-2">
                Calon #{index + 1}
              </label>
              <input
                type="text"
                bind:value={candidates.president[index]}
                placeholder="Nama calon presiden"
                class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-purple-500 focus:border-transparent"
              />
            </div>
          {/each}
        </div>
      </div>

      <!-- Vice President Section -->
      <div class="bg-white rounded-xl shadow-lg overflow-hidden">
        <div class="bg-gradient-to-r from-blue-600 to-cyan-600 px-6 py-4">
          <h2 class="text-xl font-bold text-white flex items-center">
            <span class="bg-white text-blue-600 rounded-full w-8 h-8 flex items-center justify-center mr-3 text-sm">5</span>
            Timbalan Presiden dan Naib Presiden
          </h2>
          <p class="text-blue-100 text-sm mt-1">Masukkan maksimum 5 nama calon</p>
        </div>
        
        <div class="p-6 space-y-4">
          {#each [0, 1, 2, 3, 4] as index}
            <div>
              <label class="block text-sm font-medium text-gray-700 mb-2">
                Calon #{index + 1}
              </label>
              <input
                type="text"
                bind:value={candidates.vicePresident[index]}
                placeholder="Nama calon"
                class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              />
            </div>
          {/each}
        </div>
      </div>

      <!-- Exco Section -->
      <div class="bg-white rounded-xl shadow-lg overflow-hidden">
        <div class="bg-gradient-to-r from-green-600 to-emerald-600 px-6 py-4">
          <h2 class="text-xl font-bold text-white flex items-center">
            <span class="bg-white text-green-600 rounded-full w-8 h-8 flex items-center justify-center mr-3 text-sm">15</span>
            Exco
          </h2>
          <p class="text-green-100 text-sm mt-1">Masukkan maksimum 15 nama calon</p>
        </div>
        
        <div class="p-6 grid md:grid-cols-2 gap-4">
          {#each [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14] as index}
            <div>
              <label class="block text-sm font-medium text-gray-700 mb-2">
                Calon #{index + 1}
              </label>
              <input
                type="text"
                bind:value={candidates.exco[index]}
                placeholder="Nama calon exco"
                class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-green-500 focus:border-transparent"
              />
            </div>
          {/each}
        </div>
      </div>

      <!-- Action Buttons -->
      <div class="flex gap-4">
        <button
          type="submit"
          disabled={loading}
          class="flex-1 bg-gradient-to-r from-purple-600 to-pink-600 text-white py-4 rounded-lg font-semibold hover:from-purple-700 hover:to-pink-700 disabled:opacity-50 transition shadow-lg"
        >
          {loading ? '💾 Menyimpan...' : '💾 Simpan Semua Calon'}
        </button>
        <button
          type="button"
          on:click={clearAllFields}
          class="px-8 py-4 border-2 border-gray-300 rounded-lg font-semibold hover:bg-gray-50 transition"
        >
          🗑️ Clear Semua
        </button>
      </div>
    </form>
  </div>
</div>